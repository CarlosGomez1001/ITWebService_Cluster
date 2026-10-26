# Plan: Migración del módulo Logística a app móvil (React Native)

- **Fecha:** 2026-10-22
- **Alcance de la evaluación:** `services/ITwebService_Front/src/app/logisModulo`, su estado en Redux (`store/slices/logisticaSlice`), `services/ITwebService_Back/controllers/LogisticaControl.php` y el modelo `models/logisticaModel/logisticaModel.php`, más los endpoints genéricos que el módulo consume (imágenes y formatos PDF).
- **Estado:** borrador. La Parte 2 (pasos) es deliberadamente de alto nivel; cada etapa se desglosará en documentos propios más adelante.
- **Relacionado:** notificaciones push para la app móvil, ver `../notficaciones/00-plan.md`.

---

# Parte 1. Análisis y factibilidad

## 1.1 Qué es el módulo hoy

Es el flujo del **chofer / repartidor**: ve sus rutas abiertas, entra a una ruta, atiende cada entrega o recolección (RFSO), confirma conceptos, captura checklist de unidad y de equipo, toma fotos, recoge la firma del cliente y cierra la ruta.

| Elemento | Detalle |
|----------|---------|
| Tamaño | 29 archivos, unas 3,700 líneas en `logisModulo`; slice `logistica` (101 líneas) y thunks (257 líneas) |
| Pantallas (pages) | `LogisMainModulo`, `LogisRutaLista`, `LogisRFSOLista`, `LogisRFSO` (721 líneas, la más pesada), `LogisCheck`, `LogisCheckPage` |
| Componentes de ruta | `LogisRutaPendiente`, `LogisRutaActivo`, `LogisRutaCerrado` |
| Media | Foto general, foto por concepto (`FotosConRfso`), galería, zoom, firma del cliente, firma del chofer |
| Hooks propios | `useRFSOPage` y `helpers` (lógica de conceptos), `useLogisCheckPage`, `useFormCheckList` |
| Estado | Slice `logistica` con ruta activa, RFSO, conceptos, checklist, banderas de carga |

Observación: en `logisticaRoutesLista.js` solo están registradas dos rutas (`/logisticaModulo` y `/logisticaModulo/ruta`). Las demás (`entregas`, `firmaLogis`, `galeriaLogis`, etc.) aparecen en los `navigate(...)` del código, pero no localicé dónde se registran. Hay que ubicarlas antes de mapear la navegación móvil.

## 1.2 Backend que consume

`LogisticaControl.php` (360 líneas) expone **12 endpoints** bajo `/logistica/*`, todos delegados a `logisticaModel` (265 líneas, SQL Server):

| Grupo | Endpoints |
|-------|-----------|
| Lectura | `getRutas`, `getDestinosHojaRuta`, `getConceptosRfso`, `getCheckListData`, `getJoinCheckListExistente` |
| Estado de RFSO | `setStateRfso`, `setStateRfsoFinish`, `setStateStopAllRfso`, `setUpdateRFSOAndConRemRFSOStatus`, `setObsEntrega` |
| Checklist | `insertUpdateChecklist`, `insertUpdateChecklistUnitario`, `updateCheckListIndividually` |

Dependencias fuera del controlador:
- **Imágenes y firmas:** `POST /generales/setImgParaDocumentosLogis` (base64 en el JSON, guardado con el procedimiento `GuardaImagen_v3`) y `GET /getImagesToRender`.
- **Formatos PDF:** thunk `getThunkGetFormatoPDFGeneracionDinamicaV3`, que devuelve el PDF como URL tipo blob.
- **Notificaciones:** microservicio aparte (ver plan de notificaciones).

## 1.3 Qué se reutiliza y qué se reescribe

| Capa | Reutilizable | Comentario |
|------|:-----------:|------------|
| Contrato de API y backend | Sí, casi completo | Es REST + JSON sin dependencias del navegador. La app móvil puede llamar los mismos endpoints |
| Slice `logistica`, thunks, `ApiResponse`, helpers | Sí, con ajustes | Redux Toolkit y Axios funcionan igual en React Native. Cambia `getHeaders()` (usa `localStorage`) y la ruta de `session` |
| Hooks de negocio (`useRFSOPage/helpers`, `useFormCheckList`) | Parcial | La lógica de conceptos y checklist sirve. Hay que revisar que no mezclen DOM o `Swal` |
| Componentes visuales (JSX, Tailwind, CSS) | **No** | React Native no usa HTML ni CSS. Hay que reescribir todas las vistas con `View/Text/Pressable/FlatList` |
| Navegación | **No** | `react-router-dom` se cambia por React Navigation o Expo Router |
| Cámara | **No** | `react-html5-camera-photo` pasa a `expo-camera` / `expo-image-picker` |
| Firma | **No** | `react-signature-canvas` pasa a `react-native-signature-canvas` o similar |
| Alertas | **No** | `Swal` (SweetAlert) pasa a `Alert` nativo o un modal propio |
| Descarga/visualización de PDF | **No** | `window.open` y `<a download>` pasan a `expo-file-system`, `expo-sharing` o un visor de PDF |
| Sesión | **No** | `localStorage` pasa a almacenamiento seguro (`expo-secure-store`) |

Dependencias web detectadas en el módulo: `react-html5-camera-photo`, `react-signature-canvas`, `react-router-dom`, `window.open`, creación de `<a>` para descargar, `localStorage`, `Swal`, CSS plano (`imagePreview.css`), `@heroicons/react`, Tailwind. Mezcla Redux manual (`logisticaThunks`) con RTK Query (`todosApi`) para imágenes, y eso hay que unificarlo.

## 1.4 Hallazgos y riesgos

**Funcionales**

1. **No hay modo offline.** El chofer trabaja en campo, a menudo con mala señal. Hoy cada acción depende de una petición en vivo. Es el riesgo de producto más importante: sin cola de sincronización, la app móvil será igual de frágil que la web.
2. **Fotos en base64 dentro del JSON** y hacia un procedimiento almacenado. En datos móviles es pesado y propenso a fallar. Conviene comprimir en el dispositivo y, a futuro, subir como `multipart`.
3. **Cierre de ruta con varios pasos** (`setStateRfso`, `setStateRfsoFinish`, `setStateStopAllRfso`, `setPlaneacionRutaCerrada`). Si se pierde la conexión a la mitad, el estado puede quedar inconsistente. Se necesita idempotencia o reintento seguro.
4. **Sin geolocalización.** Hoy el módulo no captura ubicación. En móvil es una mejora natural (evidencia de entrega, ruta recorrida), pero queda fuera del alcance mínimo y se decide aparte.

**Seguridad del backend** (más delicado al exponer una app móvil)

5. **SQL con valores interpolados en el modelo.** Ejemplos vistos: `where IdEmpleado = $IdEmpleado` en `getRutas`, `UPDATE OperRFSO set AppConfirmado = $status`, y `exec dbo.GuardaImagen_v3 '$base64', '$tipoFoto', ...` en `set_imagen_BD`. Hay otras consultas que sí usan parámetros (`:idOrd`, `:idModelo`), así que el patrón es inconsistente. Conviene parametrizar todo antes de abrir un cliente nuevo.
6. **Doble flujo de validación de token** repetido en cada método (`validadorToken` y luego `validaYRenuevaToken` con `X-Refresh-Token`). Funciona, pero cada endpoint duplica el bloque. En móvil el refresco debe quedar centralizado en un interceptor de Axios.
7. **Dos formatos de respuesta** (`HTTPreponse` legado y `ResponseHelper`). La app móvil debe tratar el legado (`status`, `nuevoToken`), que es el que usa Logística.
8. **El refresh token viaja en cada petición.** En un dispositivo móvil perdido tiene más impacto que en un navegador. Se guarda cifrado y se evalúa su caducidad y revocación (`SesionesApp` ya registra sesiones).

**Operativos**

9. **URL del backend.** Hoy es `localhost:3000`. La app necesita una URL pública con HTTPS (iOS y Android bloquean HTTP plano por defecto).
10. **Distribución.** Hay que decidir cómo se instala la app: tiendas públicas, distribución interna o MDM. Cambia costos y tiempos.

## 1.5 Veredicto de factibilidad

**Factible, con riesgo bajo en el backend y medio-alto en la capa de cliente.**

- El backend ya es una API REST independiente del front, así que **no requiere reescritura** para soportar la app. Solo conviene un endurecimiento (puntos 5 y 6) y, si se decide, un endpoint de carga de imágenes más eficiente.
- La mayor parte del esfuerzo está en **reescribir las vistas y los componentes de cámara, firma y PDF**, no en la lógica de negocio.
- El valor real de la migración está en lo que la web no puede hacer: **cámara nativa, trabajo sin conexión, notificaciones push y ubicación**. Si la app solo replica la web en un teléfono, el beneficio es bajo.

## 1.6 Decisiones pendientes (afectan el plan)

| # | Decisión | Opciones |
|---|----------|----------|
| 1 | Framework | Expo (recomendado, menos fricción con cámara/push/builds) o React Native "bare" |
| 2 | Alcance de offline | Solo lectura en caché, o escritura con cola de sincronización (recomendado para choferes) |
| 3 | Repositorio | Nuevo repo `services/ITmobile_App` (consistente con el resto de `services/`) |
| 4 | Código compartido con la web | Copiar lo necesario, o extraer un paquete común (thunks, constantes) |
| 5 | Plataformas | Android primero o Android + iOS desde el inicio |
| 6 | Distribución | Tiendas, distribución interna o MDM |
| 7 | Geolocalización | Dentro del alcance inicial o fase posterior |
| 8 | Mantenimiento | Mismo repo y base de la app que Logística (se planea migrar ambos módulos) |

---

# Parte 2. Pasos de la migración (alto nivel)

Cada etapa se desglosará en un documento propio con tareas pequeñas y criterios de aceptación.

## Etapa 0. Preparación y decisiones
Cerrar las decisiones de 1.6, localizar las rutas de Logística no registradas, inventariar las pantallas y definir el criterio de "migración terminada". Entregable: alcance firmado.

## Etapa 1. Endurecimiento del backend
Parametrizar el SQL del modelo de Logística y de `set_imagen_BD`. Centralizar la validación de tokens. Confirmar CORS/HTTPS y la URL pública. Se hace antes de exponer la app, y beneficia a la web también.

## Etapa 2. Base de la app móvil
Crear el proyecto, definir estructura de carpetas, configurar entornos (dev/prod), cliente Axios con interceptor de token y refresh, y almacenamiento seguro de la sesión.

## Etapa 3. Autenticación y navegación
Pantalla de login contra `/login` y `/checkAuth2`, persistencia de sesión, manejo de expiración, y el árbol de navegación equivalente al de la web.

## Etapa 4. Capa de datos reutilizada
Portar el slice `logistica`, los thunks y los helpers, quitando dependencias del navegador. Unificar la forma de consultar (hoy mezcla thunks propios y RTK Query).

## Etapa 5. Pantallas de lectura
Lista de rutas, detalle de ruta con sus RFSO y detalle de RFSO con conceptos. Incluye estados vacío, error y carga.

## Etapa 6. Captura en campo
Cámara y galería (general y por concepto), compresión de imágenes, firma del cliente y firma del chofer, observaciones y envío al backend.

## Etapa 7. Checklist y cierre
Checklist de unidad y de equipo, confirmación de conceptos, cambio de estado de RFSO y cierre de ruta, con reintentos seguros.

## Etapa 8. Trabajo sin conexión
Almacenamiento local, cola de acciones pendientes, sincronización al recuperar señal, resolución de conflictos e indicadores visibles al usuario.

## Etapa 9. Formatos PDF y compartir
Generar y ver el formato de la RFSO, descargarlo y compartirlo (WhatsApp, correo) con las APIs nativas.

## Etapa 10. Notificaciones push
Integrar el canal push (FCM/APNs) en el microservicio de notificaciones, registro de dispositivos y reconexión de Socket.io. Se apoya en el plan de notificaciones.

## Etapa 11. Pruebas
Pruebas unitarias de la lógica, pruebas en dispositivos reales (gama baja incluida), pruebas con mala señal y piloto con choferes reales.

## Etapa 12. Publicación y puesta en marcha
Íconos, permisos, políticas de privacidad, firma de builds, distribución según 1.6, capacitación y plan de soporte.

## Etapa 13. Estabilización y siguiente módulo
Corrección tras el piloto, métricas de uso y errores, y reutilización de la base para migrar **Mantenimiento**.

---

## Pronóstico de tiempo con Claude implementando y dirección del responsable

Estimación a ojo, no un compromiso. Reduce el tiempo de escribir código; no reduce pruebas en dispositivos reales, piloto, aprobación de tiendas ni la revisión del responsable. Los puntos con más incertidumbre son offline y las rutas de Logística aún no localizadas.

| Etapa | Tiempo estimado | Qué la limita |
|-------|-----------------|---------------|
| 0. Preparación y decisiones | 1 a 2 días | Respuestas del responsable a las decisiones de 1.6 |
| 1. Endurecimiento del backend | 2 a 3 días | Probar que la web no se rompa |
| 2 a 4. Base, login y capa de datos | 3 a 5 días | Poco, es código repetitivo |
| 5. Pantallas de lectura | 3 a 4 días | Revisión visual de cada pantalla |
| 6. Captura en campo (cámara y firma) | 4 a 6 días | Pruebas en teléfonos reales |
| 7. Checklist y cierre | 3 a 4 días | Validar contra datos reales |
| 8. Trabajo sin conexión | 5 a 10 días | Diseño de sincronización y conflictos |
| 9 y 10. PDF y notificaciones push | 4 a 6 días | Cuentas de Firebase y Apple |
| 11. Pruebas y piloto | 1 a 2 semanas | Choferes reales en campo |
| 12. Publicación | 1 a 2 semanas | Aprobación de las tiendas (menos con distribución interna o MDM) |
| 13. Estabilización | Según hallazgos del piloto | Se planea al cerrar la etapa 11 |

**Total aproximado: 4 a 7 semanas de calendario.**

Lo que no se acelera:
- **Pruebas en dispositivos reales.** Claude no puede ejecutar la app en un teléfono ni usar la cámara; cada ciclo de probar, reportar y corregir depende del responsable.
- **Offline.** Es la parte de más riesgo; los casos raros (cierre de ruta a medias, fotos duplicadas) solo aparecen probando con mala señal.
- **Piloto con choferes.** Una a dos semanas de uso real destapan problemas que las pruebas de escritorio no encuentran.
- **Tiendas.** Apple y Google tardan días en aprobar y Apple puede rechazar la primera versión.
- **Revisión del responsable.** Con unas 3,700 líneas de interfaz que reescribir, el plan se cumple si la revisión es rápida y diaria; en bloques semanales se alarga.

Qué acelera o alarga el plan:
- **Acelera:** Android primero, offline solo de lectura en la primera versión, geolocalización en una fase posterior, distribución interna.
- **Alarga:** iOS desde el inicio, offline con escritura completa, las rutas de Logística aún no localizadas, cambios de alcance a mitad del camino.

Las decisiones que más mueven el calendario son **plataformas**, **alcance de offline** y **forma de distribución**.

Entrega sugerida por fases usables, para que los choferes prueben algo real hacia la semana 2 o 3: primero lectura y captura (etapas 2 a 6), después cierre de ruta (7) y al final offline y push (8 a 10).

---

## Orden recomendado y dependencias

```
0 -> 1 -> 2 -> 3 -> 4 -> 5 -> 6 -> 7 -> 8 -> 9 -> 11 -> 12 -> 13
                                     \-> 10 (en paralelo desde la 3)
```

Las etapas 1 y 2 pueden avanzar en paralelo. La etapa 8 (offline) condiciona cómo se escribe la 7, por lo que conviene diseñarla desde la 4 aunque se implemente después.
