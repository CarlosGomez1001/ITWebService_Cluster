# Bitácora de cambios – Notificaciones a producción

Cada entrada indica **qué** se cambió, **por qué** y **cómo se verificó**.
Los componentes bajo `services/` son repos git independientes (ignorados por este repo de plataforma): los cambios están **sin commitear** en cada uno.

| Repo | Ruta | Rama actual |
|------|------|-------------|
| Microservicio | `services/microservices/notifications_service` | `main` |
| Backend | `services/ITwebService_Back` | `notifications` |
| Frontend | `services/ITwebService_Front` | `notificationes` |
| Plataforma | raíz | `lab` |

---

## Fase 1 – Microservicio Node ✅

| Archivo | Cambio | Justificación |
|---------|--------|---------------|
| `src/config/env.js` | Sin defaults para `API_KEY` y `NOTIFICATIONS_JWT_SECRET`; rechaza placeholder `changeme*`, secretos < 32 chars y `API_KEY == JWT secret`. En `NODE_ENV=production` exige `MONGO_URI` y `CORS_ORIGINS`. `process.exit(1)` con mensaje claro | Antes arrancaba con `changeme-generate-a-secure-key`; un olvido de configuración dejaba el servicio con una clave pública (hallazgo 4/6) |
| `src/middleware/apiKeyAuth.js` | Comparación con `crypto.timingSafeEqual` sobre hashes SHA-256 | `!==` filtra información por tiempo de respuesta; el hash evita filtrar la longitud |
| `src/services/tokenService.js` (nuevo) | `verifyNotificationToken`: HS256 únicamente, valida `iss`, `aud`, `exp`, y formato de `tenant`/`sub` | Un solo punto de verificación para REST y Socket.io. Fijar `algorithms` evita `alg:none` |
| `src/middleware/jwtAuth.js` (nuevo) | `Authorization: Bearer` → `req.auth = { tenant, userId }`, 401 con `code` (`TOKEN_EXPIRED`/`TOKEN_INVALID`/`TOKEN_MISSING`) | Cierra la lectura/escritura anónima (hallazgo 2). El `code` permite al front renovar solo cuando corresponde |
| `src/routes/notifications.js` | Rutas de usuario: `GET /`, `GET /unread-count`, `PATCH /read-all`, `PATCH /:id/read` bajo `jwtAuth`. **Se eliminan** `/:tenant/:userId...`. `/send`, `/send-bulk`, `/broadcast` siguen con API key | La identidad deja de venir en la URL: no se puede “cambiar el userId” para leer a otro |
| `src/controllers/notificationController.js` | Rutas de usuario usan `req.auth`. `markAsRead` filtra por `{_id, tenant, recipientId}` y valida `ObjectId`. Entradas validadas (ver `utils/validators.js`) | Antes `PATCH /:id/read` marcaba cualquier notificación de cualquier tenant |
| `src/utils/validators.js` (nuevo) | `tenant`/ids deben ser string/number con `^[\w.\-]{1,128}$` (rechaza `{ "$ne": null }`); `title ≤ 200`, `message ≤ 1000`; `type`/`module` por enum (400 en vez de 500); `metadata` objeto ≤ 4 KB y `metadata.route` debe ser ruta interna (`/…`, no `//…`); bulk ≤ 500 destinatarios, sin duplicados | Evita inyección de operadores NoSQL, documentos gigantes y *open redirect* vía deep-link (`navigate(metadata.route)` en el front) |
| `src/config/socket.js` | `io.use()` verifica `handshake.auth.token`; el socket se une solo al room de su token; se elimina el evento `register`; al llegar `exp` emite `token-expired` y desconecta | Cierra la suplantación en tiempo real (hallazgo 3) e implementa “reconectar al expirar” (decisión 7). Se elimina el mapa `connectedUsers` (no tenía consumidores) |
| `src/server.js` | `x-powered-by` off, `express.json({limit:'50kb'})` | Reduce huella y payloads abusivos |
| `package.json` | `+ jsonwebtoken ^9.0.2` | Verificación JWT |
| `.gitignore` | `+ node_modules` | Solo ignoraba `.env`; `npm install` habría ensuciado el repo |
| `.env.example` | Documenta `NOTIFICATIONS_JWT_SECRET`, `ISSUER`, `AUDIENCE`, `CORS_ORIGINS` de producción; `API_KEY` vacía | Plantilla sin secretos |

**Verificación (copia en scratchpad, sin Mongo):** 29 pruebas automáticas pasaron: 401 sin token / secreto ajeno / `aud` incorrecto / expirado / `alg:none` / sin `sub`; 403 con API key incorrecta; 401 al usar JWT de usuario en rutas de API key; 400 por inyección NoSQL, `type` inválido, `route` externa, metadata/título excesivos, bulk > 500; socket rechazado sin token / inválido / expirado; el evento llega solo al usuario+tenant destino; `register` ya no permite suplantar; el servidor desconecta con `token-expired` al vencer. Arranque: aborta con secretos ausentes/cortos/placeholder/iguales y en producción sin `MONGO_URI`/`CORS_ORIGINS`.
No se probó contra MongoDB real (no hay `mongod` local): las consultas filtran por `tenant`+`recipientId` y deben validarse en el entorno docker (ver Checklist).

---

## Fase 2 – Backend PHP ✅

| Archivo | Cambio | Justificación |
|---------|--------|---------------|
| `includes/funciones.php` (`firmaTokensAccesoYRefresh`) | La respuesta de login incluye `"idusuario"` (de `userData['IdUsuario']`) | Pedido explícito. Nota: `IdUsuario` **ya viajaba dentro del JWT** (`userData` completo en la clave `"0"`); lo que faltaba era devolverlo en la respuesta de login para que el front lo guarde. Solo se agregan 2 líneas |
| `services/Notificaciones/NotificacionesService.php` (nuevo) | `issueToken`, `sendManual`, `broadcast`, `validateMessageInput` | Lógica fuera del controlador (patrón AuthValidator + Service). Namespace `Services\Notificaciones` ya lo cubre el PSR-4 `Services\` → `./services` (no se tocó `composer.json`) |
| ↳ token | HS256 con `NOTIFICATIONS_JWT_SECRET` (≥ 32 chars, **distinto de `SEC_KEY`**); claims `iss=it-backend`, `aud=it-notifications`, `sub=IdEmpleado`, `tenant=database`, `iat`, `exp=+900 s` | Decisión 5: el microservicio no puede forjar sesiones del CRM. TTL corto: robar el token del navegador tiene ventana de 15 min |
| ↳ identidad | `tenant` = `database` del JWT de sesión; remitente = `IdEmpleado` del JWT; si el usuario no tiene empleado → 403 | Nunca se confía en tenant/remitente del body. `CataEmpleados` se une con `LEFT JOIN`, puede haber usuarios sin empleado |
| ↳ envío manual | Cualquier usuario → cualquier empleado **de su tenant** (decisión 3). Valida que `recipientId` exista en `CataEmpleados` del tenant (consulta parametrizada) | Evita notificaciones huérfanas/spam a ids inexistentes. Cross-tenant es imposible: el tenant sale del token |
| ↳ validación | `title ≤ 200`, `message ≤ 1000` (contados en unidades UTF-16 como `String.length` de JS), `module` por lista, `metadata` objeto ≤ 4 KB, `metadata.route` solo ruta interna | Los límites coinciden con los de Node: un mensaje válido aquí no es rechazado allá. `route` bloquea `//host` y `javascript:` (el front hace `navigate(route)`) |
| ↳ broadcast | Lista blanca por `tenant:Usuario` en `.env`: `NOTIFICATIONS_BROADCAST_USERS` (gerentes → solo su tenant) y `NOTIFICATIONS_BROADCAST_GLOBAL_USERS` (soporte → `scope:"all"`). Vacío = nadie. Cada denegación y cada envío queda en `error_log` | Decisión 6. La lista incluye el tenant porque `Usuario` (p. ej. "admin") se repite entre clientes. **Regla provisional**: el código no define “gerente general” en BD; si prefieres un permiso de `CataUsuarios`, solo cambia `isInAllowList()` |
| `controllers/NotificacionesController.php` (nuevo) | `getToken` (GET), `enviar` (POST), `broadcast` (POST), todos con `AuthValidator::validateAuthentication` y `sendResponse` (propaga `nuevoToken`) | Decisión 2. Valida método y cuerpo JSON (en `AiChatController` un body vacío provoca `TypeError`; aquí devuelve 400) |
| `index.php` | `use` + 3 rutas `/notificaciones/getToken`, `/enviar`, `/broadcast` | Exposición al front |
| `classes/NotificationService.php` | `broadcast()`: `$tenant` pasa a `?string` (null = global, se omite del payload) y se agrega `?string $senderId` al final; `post()` serializa `metadata` vacío como objeto `{}` | Solo adiciones compatibles con los llamadores actuales. Lo segundo corrige una incompatibilidad: PHP serializa `[]` como lista y la validación nueva de Node la rechaza (lo detectó la prueba E2E) |
| `.env.example` | `NOTIFICATIONS_URL`, `NOTIFICATIONS_API_KEY`, `NOTIFICATIONS_JWT_SECRET`, `NOTIFICATIONS_BROADCAST_USERS`, `NOTIFICATIONS_BROADCAST_GLOBAL_USERS` | Documenta la configuración requerida |

**Verificación:** `php -l` en todos los archivos tocados. 26 pruebas PHP (token, identidad, validación, autorización de broadcast incl. aislamiento por tenant y límites UTF-16). **E2E real PHP → Node → Socket.io**: el token firmado por PHP fue verificado por Node; el broadcast del gerente llegó solo a su tenant y el de soporte (`scope:"all"`) a ambos tenants.
**No verificado localmente** (requiere SQL Server / sesión real): `employeeExists()` contra `CataEmpleados` y el flujo HTTP completo con `AuthValidator` — ver Checklist.

---

## Fase 3 – Frontend ✅

| Archivo | Cambio | Justificación |
|---------|--------|---------------|
| `src/services/notificationsAuth.js` (nuevo) | `getNotificationToken()` (caché + renovación 60 s antes de vencer + deduplicación de peticiones en vuelo) y `notificationsRequest()` (fetch con Bearer; ante 401 renueva y reintenta una vez). Usa `genAxios.get('notificaciones/getToken')` | El navegador ya no necesita ninguna API key: obtiene del backend un token de 15 min ligado a su sesión. Se calcula el vencimiento con `expiresIn` relativo para no depender del reloj del cliente |
| `src/services/notificationSocket.js` | `auth` es una **función** (token fresco en cada conexión/reconexión); se elimina `register`; ante `io server disconnect` (token vencido) limpia caché y reconecta; ante `connect_error` con `code` (handshake rechazado, donde socket.io no reconecta solo) reintenta con espera creciente (máx. 30 s); `reconnectionAttempts: Infinity` con backoff; `io(URL \|\| undefined)` para mismo origen | Decisión 7 (reconectar al expirar). Antes la identidad la declaraba el cliente |
| `src/store/slices/notificationsSlice/notificationsThunks.js` | Lectura/marcado con `notificationsRequest` y **sin tenant/userId en la URL** (`fetchNotifications()`, `fetchMoreNotifications(page)`, `fetchUnreadCount()`, `markAsReadThunk(id)`, `markAllAsReadThunk()`). `sendManualNotification` (usaba `VITE_NOTIFICATIONS_API_KEY`) se reemplaza por `setThunk_EnviarNotificacion` y `setThunk_BroadcastNotificacion` vía `apiGenThunkPost` hacia el backend | Elimina la API key del bundle (hallazgo 1) y respeta el patrón de thunks del proyecto. `sendManualNotification` no tenía ningún consumidor en `src/` |
| `src/app/hooks/useNotifications.js` | Llama a `connectNotifications()` sin tenant/usuario y a los thunks nuevos; tras una **reconexión** vuelve a cargar lista y contador | Una desconexión por vencimiento de token pudo perder eventos en vivo; al reconectar se resincroniza con Mongo |
| `src/global/notificationsURL.js` | `import.meta.env.VITE_NOTIFICATIONS_URL ?? 'http://localhost:3002'`. Vacía = mismo origen | Decisión de dominio: front y microservicio comparten `crm.itwebservices.net`. Se acabó editar el archivo para cambiar de ambiente |
| `Dockerfile.prod` | `ARG/ENV VITE_NOTIFICATIONS_URL` (vacío por defecto) | Vite incrusta la variable en build |
| `authSlice.js`, `thunks.js` (`validateToken`), `hooks/useCheckAuth.js` | `idusuario` se guarda en Redux (`IdUsuario`) y en `localStorage.session`, y se restaura en `validateUser`; parámetro agregado **al final** de `validateToken` | Pedido explícito (“que login incluya IdUsuario”). Compatible con sesiones existentes: si falta en `localStorage` queda `undefined` hasta el siguiente login |

**Identidad de notificaciones:** sigue siendo `IdEmpleado` (el `recipientId` ya guardado en Mongo). `IdUsuario` queda disponible en el estado/sesión pero no se usa como identidad para evitar migrar datos existentes; cambiarlo es trivial (`NotificacionesService::getIdentity`).

**Verificación:**
- `vite build` compila; `grep` del bundle: **no existe `x-api-key` ni `VITE_NOTIFICATIONS_API_KEY`**. ESLint limpio en los archivos nuevos/reescritos (los errores que reporta en `authSlice.js`/`thunks.js` son anteriores y no se tocaron).
- Prueba funcional con el **código real del front** (empaquetado con esbuild; solo `genAxios` simulado) contra el microservicio real con tokens de 3 s: el servidor desconectó con `io server disconnect` cada 3,0 s, el cliente reconectó en ~10 ms con token nuevo, y las notificaciones se entregaron antes y después de cada reconexión; el REST con ese token no devolvió 401.

---

## Fase 4 – Infraestructura y documentación ✅

| Archivo | Cambio | Justificación |
|---------|--------|---------------|
| `docker-compose.yml` | Servicio `notifications`: `NOTIFICATIONS_JWT_SECRET` obligatorio (`:?`). `frontend-dev`: `VITE_NOTIFICATIONS_URL=http://localhost:${MS_NOTIFICATIONS_PORT}`. `frontend-prod`: build-arg `VITE_NOTIFICATIONS_URL` (vacío = mismo origen) | El microservicio no arranca sin el secreto del token; el front necesita la URL en build. **No** se añadieron variables al servicio `backend`: sigue leyendo su propio `.env` (`env_file`), y declararlas con `:-` vacío anularía (Dotenv *immutable*) los valores de ese archivo |
| `.env.example` (raíz) | `NOTIFICATIONS_JWT_SECRET`, `NOTIFICATIONS_CORS_ORIGINS` (ejemplo de producción), `VITE_NOTIFICATIONS_URL`; advertencia de no usar prefijo `VITE_` en secretos | Plantilla sin secretos |
| `nginx/nginx.conf` (proxy del repo, perfil `proxy`) | `location ~ ^/api/notifications/(send\|send-bulk\|broadcast)/?$ { return 404; }` + `limit_req` en `/api/notifications` + `Connection ""` para keepalive del upstream | Las rutas de API key no deben ser alcanzables desde internet (defensa en profundidad aunque exijan clave). El rate-limit frena abuso de las rutas de usuario |
| `renew-plan-docs/notficaciones/nginx-servidor-crm.snippet.conf` (nuevo) | Bloques `location` para el nginx del **servidor** (`crm.itwebservices.net`, fuera del repo): bloqueo de rutas de servicio, REST y WebSocket hacia `127.0.0.1:3002` | En producción el proxy es el standalone `../reverse-proxy` (80/443); este repo solo publica puertos en loopback. Requiere añadirlo a mano |
| `DOCKER.md`, `services/microservices/notifications_service/NOTIFICATIONS.md` | Variables, endpoints, eventos, diagrama, flujo del front, seguridad y checklist de salida a producción actualizados a v2 | Evitar documentación que describa rutas/eventos que ya no existen |

**Verificación:**
- `docker compose config` válido (con el secreto inyectado solo en esa ejecución); build-args y variables llegan donde corresponde.
- `nginx -t` sobre `nginx/nginx.conf` correcto.
- **nginx real (contenedor temporal) con el snippet del servidor**: `POST /api/notifications/send|send-bulk/|broadcast` con API key válida → **404**; `GET /api/notifications/unread-count` sin token → **401 desde el microservicio** (llega); `/` → front; **WebSocket a través del proxy conectó** (transporte `websocket`).

---

## Fase 5 – Verificación final ✅

| Prueba | Resultado |
|--------|-----------|
| Microservicio (sin Mongo): 29 pruebas de JWT, API key, validación y socket | ✅ 29/29 |
| Microservicio **con MongoDB real** (contenedor temporal): envío, entrega en vivo, paginación, aislamiento usuario/tenant, `PATCH` ajeno → 404, `read-all` acotado, bulk deduplicado, operadores NoSQL en query | ✅ 19/19 |
| PHP: token, identidad, validación, autorización de broadcast (26 pruebas) + `php -l` | ✅ 26/26 |
| E2E PHP → Node → Socket.io (token firmado por PHP verificado por Node; broadcast por tenant y global) | ✅ |
| Front real (esbuild) con tokens de 3 s: reconexión automática al expirar y entrega posterior | ✅ |
| `vite build` y `grep` del bundle: sin `x-api-key` ni `VITE_NOTIFICATIONS_API_KEY` | ✅ |
| nginx: `-t`, bloqueo de rutas de servicio y WebSocket por proxy | ✅ |

Los contenedores temporales (`tmp-mongo-notif-test`, `tmp-nginx-notif-test`) se eliminaron; **no se tocó ningún contenedor ni volumen del entorno local en ejecución**.

### No verificado (requiere el entorno real)
- `NotificacionesService::employeeExists()` contra SQL Server (`CataEmpleados`) y el flujo HTTP completo con `AuthValidator` + sesión real (no hay SQL Server local accesible).
- Navegador real (UI de la campana) contra el stack reconstruido.
- TLS/`wss` del servidor de producción.

---

## Checklist de despliegue

**Secretos (generar con `openssl rand -hex 32`, tres valores distintos entre sí y distintos de `SEC_KEY`)**
- [ ] `NOTIFICATIONS_API_KEY` → `.env` raíz **y** `.env` del backend (mismo valor).
- [ ] `NOTIFICATIONS_JWT_SECRET` → `.env` raíz **y** `.env` del backend (mismo valor).
- [ ] Contraseña de Mongo nueva (`MONGO_NOTIFICATIONS_PASSWORD`): el `.env` del microservicio estuvo versionado. Con volumen existente hay que crear/cambiar el usuario a mano (ver `DOCKER.md`).
- [ ] `.env` del backend: `NOTIFICATIONS_URL=http://notifications:3002`, `NOTIFICATIONS_BROADCAST_USERS` y `NOTIFICATIONS_BROADCAST_GLOBAL_USERS` con entradas `tenant:Usuario`.

**Producción**
- [ ] `NODE_ENV=production`, `NOTIFICATIONS_CORS_ORIGINS=https://crm.itwebservices.net`.
- [ ] `VITE_NOTIFICATIONS_URL` vacío (mismo origen) y reconstruir el front: `docker compose --profile prod up -d --build frontend-prod`.
- [ ] Añadir el snippet `nginx-servidor-crm.snippet.conf` al nginx del servidor y recargar (`nginx -t && nginx -s reload`).
- [ ] Desplegar **microservicio + backend + front a la vez**.

**Humo post-despliegue**
- [ ] Login → la campana conecta (`isConnected`), sin errores 401/503 en consola.
- [ ] `https://crm.itwebservices.net/api/notifications/send` → 404 desde internet.
- [ ] Dos usuarios del mismo cliente: A envía a B (`POST /notificaciones/enviar`) → B la recibe en vivo; A **no** puede leer/marcar las de B.
- [ ] Usuario de otro cliente con el mismo `IdEmpleado` no recibe nada.
- [ ] Dejar la sesión abierta > 15 min: el socket se reconecta solo (log `[Notifications] Conectado`).
- [ ] Broadcast: usuario no listado → 403; gerente listado → llega solo a su cliente; soporte `scope:"all"` → llega a todos.

---

## Impacto en el entorno local que ya está corriendo

El stack de desarrollo de esta máquina está levantado con la imagen **anterior** del microservicio y el backend/front montan el código por volumen:

- **Backend y front ya usan el código nuevo** (bind mounts / HMR). El microservicio **no**: sigue con la imagen vieja y sin `NOTIFICATIONS_JWT_SECRET`.
- Mientras tanto la campana de notificaciones fallará (`getToken` → 503 hasta definir el secreto en el backend; y el microservicio viejo no entiende el token).
- Para ponerlo al día: agregar `NOTIFICATIONS_JWT_SECRET` al `.env` raíz y al `.env` del backend, `NOTIFICATIONS_URL`/`NOTIFICATIONS_API_KEY` en el del backend, y `docker compose --profile dev --profile notifications up -d --build notifications`.
- Desde ahora `docker compose` (cualquier subcomando) exige `NOTIFICATIONS_JWT_SECRET` en el `.env` raíz, igual que ya exigía `NOTIFICATIONS_API_KEY` y las credenciales de Mongo.

---

## Limitaciones conocidas / siguiente iteración

| Tema | Detalle |
|------|---------|
| Regla de broadcast provisional | Lista blanca por `.env`. Si “gerente general” existe como dato en BD (p. ej. `CataUsuarios`), sustituir `isInAllowList()` |
| Broadcast no persiste | Solo llega a quien esté conectado; el front le inventa un `_id` temporal. Pendiente decidir si debe guardarse |
| Spam entre usuarios | `enviar` permite a cualquier usuario escribir a cualquier compañero (decisión 3). No hay límite por usuario/minuto en backend; nginx limita por IP |
| `ClaveAcceso` dentro del JWT de sesión | Hallazgo colateral (**fuera de alcance**): `userData` completo — incluido el hash MD5 de la contraseña — viaja en el JWT en la clave `"0"`. Convendría quitarlo; el token de notificaciones **no** lo incluye |
| Mongo | Sin TTL/retención, sin backups automatizados, sin apagado ordenado (`SIGTERM`), sin Redis adapter (una sola réplica) |
| Pruebas | Las pruebas de esta entrega vivieron en el scratchpad; no se agregó una suite al repo |
| Identidad | `IdEmpleado` (no `IdUsuario`) identifica al destinatario; `idusuario` ya está en login/sesión por si se decide migrar |

---

## Fase 6 – Broadcast de soporte a clientes elegidos ✅ (solicitud posterior)

**Pedido:** que soporte pueda enviar broadcast a clientes seleccionados de una lista (1, varios o todos); dejar en pausa el broadcast de gerentes (`BROADCAST_USERS`).

### Qué cambió y por qué

| Archivo | Cambio | Justificación |
|---------|--------|---------------|
| `ITwebService_Back/services/Notificaciones/NotificacionesService.php` | `broadcast()` admite 3 alcances: `tenant` (mi cliente), **`tenants`** (lista elegida) y `all`. Autorización por alcance en `canBroadcast()` **antes** de validar el cuerpo | Antes soporte solo podía elegir “mi cliente” o “todos”. Autorizar primero evita revelar a un usuario sin permiso qué campos son válidos |
| ↳ pausa de gerentes | Constante `MANAGER_BROADCAST_ENABLED = false`: `NOTIFICATIONS_BROADCAST_USERS` se ignora | Pausar sin borrar código ni variable: reactivar = cambiar una constante. Se dejó como constante (y no como variable de entorno) para que “en pausa” no dependa de un `.env` mal escrito |
| ↳ validación de `tenants` | Debe ser un arreglo (lista) de 1 a 200 strings; **cada uno se valida contra la tabla maestra `Empresa`**; se normaliza a la capitalización canónica y se deduplica; si alguno no existe → 400 con los nombres inválidos | Los rooms se llaman `user:{BaseDatos}:{id}` con la capitalización que firma el login (`Empresa.BaseDatos`); un nombre mal escrito enviaría el mensaje “con éxito” a nadie. Validar contra la fuente evita falsos positivos, destinos inventados y entradas tipo `{"$ne":1}` |
| ↳ `getTenants()` + `fetchTenants()` | Nuevo `GET /notificaciones/getTenants` (solo soporte) con `SELECT IdEmpresa, BaseDatos FROM Empresa`. `fetchTenants()` es `protected` | El selector del front necesita la lista; usa la misma tabla/columnas que ya consulta el login (`Conections::getExisteBD`), así no se asume ninguna columna nueva. Protegida para poder sustituirla en pruebas sin SQL Server |
| ↳ auditoría | `error_log` con quién envió, el alcance y los clientes destino; también se registran las denegaciones | Un broadcast masivo es una acción sensible: debe quedar rastro |
| `controllers/NotificacionesController.php`, `index.php` | Método `getTenants` y ruta `GET /notificaciones/getTenants` | Solo adiciones; mismo patrón `AuthValidator` + `sendResponse` |
| `classes/NotificationService.php` | `broadcast()` recibe `?array $tenants = null` como **último** parámetro y lo envía como `tenants` | Cambio compatible con los llamadores actuales (parámetro opcional al final) |
| `ITwebService_Back/.env.example` | Documenta que `GLOBAL_USERS` = soporte (una entrada por persona) y que `BROADCAST_USERS` está en pausa | Evitar configurar algo que se ignora |
| `notifications_service/src/controllers/notificationController.js`, `utils/validators.js`, `services/socketService.js` | `/broadcast` acepta `tenants` (array 1–200, sin duplicados, ids válidos); `tenant` y `tenants` son excluyentes (400 si llegan ambos); `emitBroadcast` acepta string, array o null | **Una sola llamada** HTTP del backend al microservicio sin importar cuántos clientes sean. Hacer un bucle desde PHP serían N llamadas con timeout de 5 s cada una |
| `NOTIFICATIONS.md`, thunk del front (solo comentario) | Documentación y comentario alineados | Evitar documentación obsoleta |

### Quién puede qué (resultado)

| Usuario | `scope:"tenant"` | `scope:"tenants"` | `scope:"all"` | `getTenants` |
|---|---|---|---|---|
| Soporte (`NOTIFICATIONS_BROADCAST_GLOBAL_USERS`) | ✅ | ✅ | ✅ | ✅ |
| Gerente (`NOTIFICATIONS_BROADCAST_USERS`) | ⏸ pausa → 403 | 403 | 403 | 403 |
| Cualquier otro | 403 | 403 | 403 | 403 |

Configuración: una entrada `BaseDatos:Usuario` **por persona de soporte**, no por cliente destino. Ejemplo: `NOTIFICATIONS_BROADCAST_GLOBAL_USERS=ITR_Monti:soporte1,ITR_Monti:soporte2`.

### Ejemplo de uso (backend)

```jsonc
// 1) Obtener la lista de clientes
GET  /index.php/notificaciones/getTenants
→ { "data": [ { "IdEmpresa": 1, "BaseDatos": "ITR_Monti" }, { "IdEmpresa": 2, "BaseDatos": "ITR_Otra" } ] }

// 2) Enviar a clientes elegidos (1 o varios)
POST /index.php/notificaciones/broadcast
{ "title": "Mantenimiento", "message": "Hoy 22:00 habrá mantenimiento.", "scope": "tenants", "tenants": ["ITR_Monti", "ITR_Otra"] }
→ { "data": { "scope": "tenants", "tenants": ["ITR_Monti", "ITR_Otra"] } }

// Todos: { ..., "scope": "all" }      Mi cliente: { ..., "scope": "tenant" }
```

### Verificación
- **PHP: 24/24** (con la lista de clientes simulada, sin SQL Server): 403 para gerente/usuario normal en los 3 alcances y en `getTenants`; 400 para scope inválido, `tenants` ausente/vacío/string/asociativo/con elemento no-string/con cliente inexistente/>200; 500 si falla la BD; deduplicación y capitalización (`itr_otra`, `ITR_OTRA ` → `ITR_Otra`).
- **E2E PHP → microservicio → 4 clientes conectados por socket**: `tenant` → solo ITR_Monti; `tenants:[Otra,Tercera]` → solo esas dos; `tenants:[Cuarta]` → solo esa; `all` → los cuatro.
- **Microservicio**: 400 si llegan `tenant` y `tenants` juntos, `tenants` vacío, con objeto `{"$ne":1}` o como string.
- **No verificado:** la consulta real a `Empresa` en SQL Server (usa solo `IdEmpresa` y `BaseDatos`, columnas que el login ya consulta). Si quieres mostrar un nombre amigable del cliente en el selector, hay que confirmar el nombre de esa columna.
- **Pendiente en el front (no solicitado):** un thunk `getThunk_Tenants` y la pantalla/selector de soporte para usar `getTenants` y `setThunk_BroadcastNotificacion`.
