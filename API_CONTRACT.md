# API Contract — IT Web Service Cluster

Fuente única y canónica del contrato de red entre `it_web_service-Backend_R`
y `it_web_service-Front`. Los tres `CLAUDE.md` del cluster explican el flujo
en prosa (arquitectura, auth, ejemplos); este archivo es la referencia
**exacta y verificada contra el código real** que las skills
(`crud-endpoint`, `endpoint-controller`, `http-request-thunk`) y los agentes
(`BackendAgent`, `FrontAgent`) deben seguir al generar código nuevo que
cruza la frontera backend↔frontend.

Si este archivo y algún `CLAUDE.md` llegan a diferir, repórtalo como
inconsistencia a corregir — no asumas cuál es la versión correcta sin
verificar contra el código fuente citado en cada sección.

## 1. Formato de respuesta en el cable (wire format)

Todo endpoint, sin excepción, responde este único shape JSON. Lo produce
`HTTPreponse()` (`includes/funciones.php`), llamado directo o a través de
`AuthValidator::sendResponse()` (`services/Auth/AuthValidator.php`):

```json
{
  "status": 200,
  "ok": true,
  "message": "Exito",
  "nuevoToken": null,
  "data": []
}
```

| Campo | Tipo | Regla |
|---|---|---|
| `status` | int | Código HTTP-like. **El frontend solo trata la respuesta como éxito si `status === 200` exactamente** (`apiGenThunkGet`/`apiGenThunkPost`/`apiThunkGet`/`apiThunkPost` hacen `if (data.status !== 200) throw ...`). Un `201`/`204` no cuentan como éxito para los thunks actuales. |
| `ok` | bool | `status >= 200 && status < 300`. Informativo — el frontend **no** lo usa para decidir éxito/error, usa `status`. |
| `message` | string | Mensaje humano. En error, es lo único que llega como `error.message` en el `ApiResponse` del frontend. |
| `nuevoToken` | string\|null | JWT de acceso refrescado. Si no es `null`, el frontend debe actualizar la sesión (ver sección 4). |
| `data` | any\|null | Payload real. Es exactamente lo que el frontend recibe como `ApiResponse.data` — la forma exacta de `data` (columnas del SELECT, claves del objeto) es lo que un componente/tabla del front va a consumir directo. No cambies esta forma sin coordinar el lado frontend (o sin usar un alias en el propio SQL). |

Nunca devuelvas un array crudo ni el shape interno de `ResponseHelper`
(sección 2) directo al cliente — siempre debe pasar por `HTTPreponse()` /
`AuthValidator::sendResponse()`.

## 2. Formato interno Servicio → Controlador (`ResponseHelper`)

Capa intermedia, **nunca llega tal cual al frontend** — la traduce
`AuthValidator::sendResponse()` al wire format de la sección 1.

```php
// classes/ResponseHelper.php
ResponseHelper::success($data, 'mensaje');
// -> ['success' => true, 'message' => 'mensaje', 'data' => $data]

ResponseHelper::error('mensaje', 403, $extra = []);
// -> ['success' => false, 'message' => 'mensaje', 'error_code' => 403, 'extra'?]
```

`AuthValidator::sendResponse(array $resultado, array $tokenData): void`:

- Si `$resultado['success']` → `HTTPreponse(1, 200, $resultado['message'], $resultado['data'], $nuevoToken)`.
- Si no → `HTTPreponse(0, $resultado['error_code'] ?? 500, $resultado['message'], null, $nuevoToken)`.

**Detalle no obvio:** en el camino de error, `data` siempre viaja `null` —
aunque el servicio haya puesto algo en `extra` (p. ej. una lista de errores
de validación vía `ResponseHelper::error($msg, 403, $errores)`), ese
`extra` **no llega al frontend**, solo `message`. Si un endpoint necesita
mandar detalle estructurado del error, tiene que ir concatenado/serializado
en `message`, o el endpoint deberá construir su propia respuesta sin pasar
por `sendResponse`.

## 3. Contrato de request (backend)

| | GET | POST |
|---|---|---|
| Headers | `Authorization: Bearer <token>` + `X-Refresh-Token: Bearer <token>` (ambos obligatorios) | mismos |
| Auth | `AuthValidator::validateAuthentication($headers)` — aborta la request con `HTTPreponse` internamente si falla (401/403), no hace falta un `if` extra alrededor | igual |
| Params | Query string, cada uno sanitizado con `AuthValidator::validateGetInteger` / `validateGetString` / `validateGetDate` — **nunca `$_GET[...]` crudo** | Body JSON completo vía `json_decode(file_get_contents("php://input"), true)` |
| Método no permitido | — | siempre `HTTPreponse(0, 405, "Metodo no permitido")` fuera del `if ($_SERVER['REQUEST_METHOD'] === 'POST')` |

Las plantillas completas de controlador (con este contrato ya aplicado)
viven en las skills `crud-endpoint` y `endpoint-controller` — este archivo
documenta el contrato en sí, no reimplementa las plantillas.

## 4. Contrato de consumo (frontend)

`apiGenThunkGet` / `apiGenThunkPost`
(`src/store/slices/general/thunksBase.js`) son la vía estándar para
peticiones nuevas (ver skill `http-request-thunk`). Su contrato interno:

1. Llaman al endpoint vía `genAxios` — GET manda `params` como query
   string, POST los manda como body (`data: params`). `genAxios.baseURL`
   es la raíz del API sin prefijo de módulo, por eso `endpoint` **debe
   incluir siempre** el prefijo (`crm/...`, `logistica/...`, etc.).
2. Si `data.status !== 200` → lanza error → el thunk resuelve
   `ApiResponse.rejected({ error: message, status })`.
3. Si `data.nuevoToken` es truthy → `dispatch(actualizarToken({ tokenNuevo: data.nuevoToken }))`.
   El interceptor de respuesta de `genAxios`/`crmAxios` **también** llama a
   `setNuevoToken(response.data.nuevoToken)` por su cuenta — es
   intencional (doble camino de refresh), no lo "arregles" quitando uno de
   los dos si tocas ese código sin que te lo pidan explícitamente.
4. Éxito → `ApiResponse.fullfilled(data.data)`. Si se pasó `slicer`
   (solo disponible en `apiGenThunkGet`), en vez de regresar el dato
   despacha `data.data` a ese reducer y resuelve `ApiResponse.fullfilled()`
   sin payload.

`ApiResponse` (`src/store/apiClass/responses.js`):

```js
{ success: bool, data: any|null, error: { error: string, status: number } | null }
```

Es lo que devuelven `dispatch(thunkAction(...))` y, envuelto, los hooks
`useGetData` / `useSetData` / `useGetDataWithReturn` (`src/app/hooks/`,
documentados en la skill `http-request-thunk`).

## 5. Prefijos de ruta por módulo

| Módulo | Prefijo backend | Controlador |
|---|---|---|
| CRM | `/crm/listas/*` (GET), `/crm/set/*` (POST) | `CRMController` |
| Logística | `/logistica/listas/*`, `/logistica/set/*` | `LogisticaControl` |
| Mantenimiento | raíz o `/mtto/*` según endpoint | `MantenimientoController` |
| Generales | `/generales/*`, `/formatos/*` | `GeneralesController` |

Tabla completa de endpoints existentes y ejemplos reales: sección "API
Endpoints Summary" de `CLAUDE.md` (raíz del cluster).

## 6. Checklist para cualquier endpoint nuevo

**Backend:**

- [ ] Responde siempre vía `HTTPreponse()` / `AuthValidator::sendResponse()`
      — nunca un array crudo ni el shape de `ResponseHelper` directo.
- [ ] Éxito = `status` literal `200` (no otro 2xx).
- [ ] GET: todos los params pasan por `AuthValidator::validateGet*`.
- [ ] POST: body vía `json_decode(file_get_contents("php://input"), true)`.
- [ ] La forma de `data` (columnas/claves exactas) queda documentada en la
      respuesta al usuario, para que el thunk del frontend sepa qué esperar.

**Frontend:**

- [ ] Thunk nuevo con `apiGenThunkGet` / `apiGenThunkPost`, `endpoint`
      incluye el prefijo de módulo completo (sección 5).
- [ ] Hook elegido según consumo real (`useGetData` / `useSetData` /
      `useGetDataWithReturn`).
- [ ] No asume campos en `data` que el backend no documentó/expuso.

## 7. Dónde referenciar este archivo

- Backend: `it_web_service-Backend_R/.claude/skills/crud-endpoint/SKILL.md`,
  `.../endpoint-controller/SKILL.md`, `.claude/agents/BackendAgent.md`.
- Frontend: `it_web_service-Front/.claude/skills/http-request-thunk/SKILL.md`,
  `.claude/agents/FrontAgent.md`.

Si agregas una skill o agente nuevo que genere código de un lado u otro del
contrato de API, agrégalo también a esta lista.
