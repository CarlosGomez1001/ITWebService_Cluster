---
name: prompt-enhancer
description: Toma una petición informal, corta o ambigua del usuario y la reescribe en un prompt completo y accionable para trabajar en este cluster full-stack — agrega el contexto arquitectónico relevante de los tres CLAUDE.md (cluster, frontend y backend), detecta qué skill(s) existentes de ambos repos hijos (it_web_service-Front e it_web_service-Backend_R) encajan mejor con la petición, y deja explícitos módulo, capa/namespace afectados, convenciones de seguridad, orden de ejecución frontend/backend y criterios de aceptación, ANTES de escribir código. Usar cuando el usuario pida "mejora este prompt", "ayúdame a pedir esto bien", "no sé cómo explicar lo que quiero", "arma/redacta el prompt para...", o cuando una petición llegue tan vaga o corta que convenga clarificarla antes de tocar frontend o backend.
---

# Mejorador de prompts (arquitectura del cluster + skills disponibles en ambos repos)

Esta skill vive en la raíz del cluster, un nivel arriba de sus dos repos
hijos (`it_web_service-Front/` e `it_web_service-Backend_R/`), y por eso es
la única con visión simétrica de ambos: no está sesgada hacia frontend ni
hacia backend. **No implementa la petición del usuario** — la reescribe en
un prompt completo y ejecutable, inyectando el contexto que el usuario
normalmente tendría que explicar a mano: qué repo(s) toca, módulo y
capa/namespace afectados, convenciones de seguridad, y qué skill(s) ya
instaladas en cualquiera de los dos repos conviene usar, en qué orden.

Todo lo que agregues debe salir de los `CLAUDE.md` reales del cluster y de
las skills reales instaladas en ambos repos — nunca inventes namespaces,
rutas, nombres de tabla, columnas o convenciones que no verificaste con
`grep`/`Read` en el código o que no estén documentadas.

## Cuándo NO aplica

Si la petición ya trae módulo, entidad, capa y comportamiento esperado con
claridad, no hace falta esta skill — ve directo a la skill específica que
aplique (`crud-endpoint`, `endpoint-controller`, `http-request-thunk`, etc.).
Esta skill es para el caso contrario: "expón las refacciones al front",
"necesito ver los pedidos vencidos", "conecta esto con el backend", "agrega
algo para actualizar el estatus".

## Procedimiento

### 1. Reunir la petición cruda

Toma `args` tal cual la dio el usuario. Si viene vacía, pregunta en una
frase qué quiere lograr — no inventes el objetivo ni la entidad.

### 2. Leer los tres `CLAUDE.md` y extraer solo lo relevante

Lee (no resumas de memoria), relativos a la raíz del cluster (donde vive
esta skill):

| Archivo | Ruta desde la raíz del cluster |
|---|---|
| Cluster (arquitectura compartida, contrato de API) | `./CLAUDE.md` |
| Frontend | `./it_web_service-Front/CLAUDE.md` |
| Backend | `./it_web_service-Backend_R/CLAUDE.md` |

No cites los tres completos en el prompt final — usa esta tabla de palabras
clave para decidir qué secciones son relevantes a la petición del usuario:

| Palabras clave en la petición | Sección relevante | Dónde |
|---|---|---|
| "endpoint", "ruta", "expón al front", "controller" | Router Pattern, Adding a New Endpoint | Backend + Cluster CLAUDE.md |
| "modelo", "tabla", "columnas", "catálogo", "entidad nueva" | ActiveRecord ORM (`$tabla`, `$primarykey`, `$columnasDB`, mass-assignment en el constructor) | Backend CLAUDE.md |
| "servicio", "lógica de negocio" | Coding Conventions, patrón `Service` usado en `crud-endpoint`/`endpoint-controller` | Backend CLAUDE.md + skills del backend |
| "login", "token", "jwt", "sesión", "permisos", "401", "403" | Authentication Flow, Token Structure, `AuthValidator` vs. `validadorTokenCrm` legacy | Cluster + Backend CLAUDE.md |
| "sql server", "conexión", "base de datos", "ambiente pruebas/producción" | Database Connections, Environment Variables | Backend CLAUDE.md |
| "cors", "el front no conecta", "origin", "producción" | CORS Configuration, Environment Switching | Cluster + Backend CLAUDE.md |
| "formato de respuesta", "json", "HTTPreponse", "ResponseHelper" | Response Format, API Response Formats | Cluster + Backend CLAUDE.md |
| "docker", "deploy", "odbc" | Docker Deployment | Backend CLAUDE.md |
| "traer datos", "guardar", "petición al backend", "conectar con backend" | Request Flow Architecture (Thunks Pattern), General Thunks `apiGenThunkGet`/`apiGenThunkPost` | Cluster CLAUDE.md |
| "permisos", "roles", "quién puede ver/editar" (en pantalla) | Permissions System (roles, `<Protected>`, `usePermissions`) | Front CLAUDE.md |
| "kanban", "cotización", "prospecto", "cliente", "pedido" (en pantalla) | CRM Module / CRM Kanban Board | Front CLAUDE.md |
| "ruta nueva", "página nueva", "módulo nuevo" (en pantalla) | Routing (`AppRouter`, `AppRoutes`, prefijos `/crm/*` `/logisticaModulo/*` `/mttoModulo/*`) | Front CLAUDE.md |
| "formulario", "tabla", "filtro", "listado" (en pantalla) | Custom Hooks, estructura de módulo (`components/`, `context/`) | Front CLAUDE.md |

Si la petición cruza dos categorías (p. ej. "quiero ver refacciones nuevas
en la cotización" implica backend + front), incluye ambas.

### 3. Identificar módulo y, si toca backend, namespace ANTES de redactar el prompt

Toda petición backend necesita resolver a qué módulo pertenece, porque de
eso depende el namespace del Modelo, la carpeta del Servicio y el prefijo de
ruta. Usa esta tabla (o `grep` directo si no es obvio):

| Módulo | Namespace del Modelo | Carpeta | Controlador | Patrón de auth |
|---|---|---|---|---|
| CRM | `CRMModel` | `models/Crm/` | `CRMController` | `AuthValidator` (moderno) |
| Mantenimiento | `ModelMtto` | `models/Mtto/` | `MantenimientoController` | legacy (`validadorTokenCrm`) |
| Logística | `LogisticaModel` | `models/logisticaModel/` | `LogisticaControl` | legacy (`validadorTokenCrm`) |
| Generales | `GlobalesModel` | `models/Globales/` | `GeneralesController` | — |
| Facturación | `FacturacionModel` | `models/Facturacion/` | `Facturacion\FacturacionController` | — |

Si el módulo destino es `mtto` o `logistica`, señala en el prompt final que
ese controlador **todavía no usa** `AuthValidator` + `Service` — y deja como
"dato que falta / confirmar con el usuario" si el prompt debe seguir el
patrón legacy del controlador existente o migrar a `AuthValidator` (mismo
criterio que exige `crud-endpoint`, no lo decidas tú solo).

Si la petición no deja claro el módulo (p. ej. solo da el nombre de una
tabla), busca la tabla/entidad con
`grep -ri "<nombre>" it_web_service-Backend_R/models/ it_web_service-Backend_R/services/`
antes de asumir, y si hay ambigüedad entre módulos pregúntalo.

### 4. Descubrir las skills disponibles en ambos repos (dinámico, no hardcodees)

No asumas una lista fija — las skills cambian con el tiempo. Descúbrelas en
el momento, desde la raíz del cluster:

```bash
# Skills del frontend
for f in it_web_service-Front/.claude/skills/*/SKILL.md; do
  echo "== $f =="; sed -n '1,10p' "$f"
done

# Skills del backend
for f in it_web_service-Backend_R/.claude/skills/*/SKILL.md; do
  echo "== $f =="; sed -n '1,10p' "$f"
done
```

Extrae `name:` y `description:` del frontmatter de cada una y compáralas
contra la petición del usuario (la `description` ya trae las frases gatillo
"usar cuando el usuario pida...").

### 5. Emparejar la petición con skill(s)

- **Solo backend** (p. ej. "agrega un endpoint que use el servicio X"):
  recomienda `endpoint-controller` si el Servicio ya existe, o
  `crud-endpoint` si hace falta Modelo + Servicio desde cero.
- **Solo frontend** (p. ej. "conecta esta pantalla a un endpoint que ya
  existe"): recomienda `http-request-thunk`.
- **Backend + frontend** (típico: "quiero ver X en la pantalla de Y"):
  recomienda primero la skill backend que aplique, **y después**
  `http-request-thunk` del repo frontend para el consumo — dilo
  explícitamente en el prompt final, con el orden correcto (backend antes
  que frontend, porque el thunk necesita el endpoint ya expuesto).
- **Sin coincidencia**: dilo explícitamente y referencia en su lugar la
  sección/patrón concreto del `CLAUDE.md` o del código existente (p. ej.
  "sigue el patrón de `ClientesObras_Service`/`CRMController::insertObra`",
  o "sigue el patrón de `CrmCotizacionContextProvider.jsx`").

Si hay ambigüedad real (¿ya existe el servicio o no?, ¿qué módulo?, ¿qué
columnas?, ¿GET o POST?), usa `AskUserQuestion` en vez de adivinar — igual
que exige cada skill individual.

### 6. Complementar el prompt con el contenido real de la(s) skill(s) emparejada(s)

No te limites a decir "usa la skill X" — lee con `Read` la skill completa
(no solo su `description`) y trae al prompt final las reglas concretas ya
aplicables: qué se genera siempre vs. solo con scripts SQL, el patrón
`AuthValidator::validateGetInteger` / `validateAuthentication` /
`sendResponse`, la regla de "solo adiciones, nunca tocar métodos existentes"
si aplica `endpoint-controller`, nombres de hooks y prefijos
`get.../set...` si aplica `http-request-thunk`. El prompt final debe ser
autosuficiente para alguien que no leyó la skill.

### 7. Redactar el prompt mejorado

Bloque de código markdown, en español, con esta estructura fija:

```
## Objetivo
<una frase clara y concreta>

## Repo(s) y módulo(s) afectados
<frontend | backend | ambos — y dentro de backend: crm | mtto | logistica | generales | facturación>

## Capa(s) a tocar
<Modelo nuevo? Servicio nuevo o existente? solo Controlador+Ruta? componente/hook en el front? — usa el criterio crud-endpoint vs endpoint-controller vs http-request-thunk>

## Contexto de arquitectura relevante
<bullets del paso 2, solo lo aplicable>

## Skill(s) recomendada(s)
<nombre + repo + por qué encaja + orden de ejecución si involucra ambos repos (backend antes que frontend)>
<si no hay match: "Ninguna skill instalada cubre esto — seguir el patrón de <archivo/método existente>">

## Convenciones a respetar
<AuthValidator vs. legacy según el controlador destino; sanitización de $_GET; mass-assignment solo vía $columnasDB; convenciones de hooks/thunks del frontend si aplica>

## Archivos probablemente involucrados
<rutas reales en ambos repos: it_web_service-Backend_R/models/..., it_web_service-Front/src/...>

## Datos que faltan (si aplica)
<tabla, columnas, primary key, endpoint exacto, GET o POST, si el servicio ya existe o no>

## Criterios de aceptación
<checklist corto: respuesta vía AuthValidator::sendResponse, thunk consume el endpoint correcto, git diff solo adiciones si aplica, etc.>
```

### 8. Cerrar con el usuario

Pregunta explícitamente si quiere que:
1. Se ejecute ya invocando la(s) skill(s) recomendada(s) en el/los repo(s)
   correspondiente(s), o
2. Solo se quede con el prompt para usarlo después (por ejemplo para
   pasarlo a quien vaya a trabajar el otro lado, o para otra sesión).

No implementes la petición original como parte de esta skill salvo que el
usuario confirme la opción 1.

## Notas

- Esta skill vive en la raíz del cluster y por eso puede leer ambos repos
  directamente como subcarpetas (`it_web_service-Front/`,
  `it_web_service-Backend_R/`) — no uses rutas `../`, aquí no aplican.
- Si alguno de los dos repos no existe en esa ruta relativa (p. ej. el
  cluster se clonó con otra estructura de carpetas), dilo y pregunta la
  ruta correcta en vez de omitir silenciosamente ese contexto.
- Los repos hijos tienen su propia copia de esta skill (`prompt-enhancer`)
  con rutas relativas a `../` porque se invocan desde dentro de cada repo.
  Esta versión es la que aplica cuando se trabaja desde la raíz del
  cluster; mantenerlas consistentes en contenido si una se actualiza.
- No crees archivos, no corras `composer`/`npm` ni hagas commits como parte
  de esta skill; su única salida es el prompt mejorado (texto) más, si el
  usuario lo pide, la ejecución de la skill recomendada.
