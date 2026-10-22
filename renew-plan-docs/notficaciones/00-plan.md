# Plan: Notificaciones listas para producción

- **Fecha:** 2026-10-20
- **Alcance:** microservicio `notifications_service`, backend PHP, frontend React, `docker-compose.yml`, `nginx/`, docs.
- **Estado de cada fase:** ver [01-cambios.md](01-cambios.md) (bitácora de lo realmente aplicado y su justificación).

## 1. Punto de partida (auditoría)

Hallazgos que bloqueaban producción:

| # | Hallazgo | Riesgo |
|---|----------|--------|
| 1 | `VITE_NOTIFICATIONS_API_KEY` viaja en el bundle del front (`sendManualNotification`) | Cualquiera extrae la key y llama a `/send`, `/send-bulk`, `/broadcast` |
| 2 | Rutas REST de lectura/escritura sin autenticación (`/:tenant/:userId`, `read`, `read-all`) | Leer/marcar notificaciones de otro usuario o de otro cliente cambiando la URL |
| 3 | Socket.io: el evento `register` acepta cualquier `{userId, tenant}` | Suscribirse en vivo a las notificaciones de otro usuario |
| 4 | `API_KEY` con valor por defecto, comparación no constante, `.env` que estuvo versionado | Secreto adivinable / filtrado |
| 5 | URLs de producción apuntan a `localhost` / `:3002`; sin TLS/wss en el proxy del repo; CORS por defecto a localhost | No funciona tras HTTPS; superficie expuesta |

## 2. Decisiones tomadas (respuestas del responsable)

| Tema | Decisión | Consecuencia en el diseño |
|------|----------|---------------------------|
| Identidad | El JWT de acceso ya trae `IdUsuario` e `IdEmpleado` en `userData` (clave `"0"`). El **login no devolvía `idusuario`** | Se agrega `idusuario` a la respuesta de login y al estado/sesión del front. La identidad de notificaciones **sigue siendo `IdEmpleado`** (ya es el `recipientId` almacenado en Mongo; evita migración) |
| Controlador | Nuevo `NotificacionesController` con `AuthValidator` | Archivo propio + rutas `/notificaciones/*` |
| Envío manual | Cualquier usuario → cualquier usuario **de su mismo tenant** | `tenant` y `senderId` salen del token, nunca del body. Se valida que el destinatario exista en la BD del tenant |
| Dominio | `crm.itwebservices.net`; front y microservicio comparten host; el nginx del servidor enruta al backend lo que empieza con `/index.php` | El front usa el mismo origen (`https://crm.itwebservices.net`) y paths `/api/notifications` y `/notifications/socket.io` → hay que añadir esos dos `location` al nginx del servidor |
| Token | El backend firma un **token específico de notificaciones** (no se comparte `SEC_KEY` con Node) | Secreto distinto `NOTIFICATIONS_JWT_SECRET`, `aud=it-notifications`, TTL corto (15 min) |
| Broadcast | Debe existir; lo usan gerentes generales y soporte | Pasa por el backend con lista blanca (`tenant:Usuario`). Gerentes → solo su tenant; soporte → global |
| Expiración | Reconectar el socket | El servidor desconecta al vencer el token; el cliente pide uno nuevo y reconecta |

## 3. Arquitectura objetivo

```
Navegador (crm.itwebservices.net)
  │ 1. GET /index.php/notificaciones/getToken  (Authorization + X-Refresh-Token)
  │ ◄── { token (15 min), tenant, userId }
  │ 2. REST  /api/notifications...            Authorization: Bearer <token notif>
  │ 3. WS    /notifications/socket.io         auth: { token }
  ▼
nginx del servidor ──► backend PHP   (/index.php...)
                  └─► notifications:3002   (/api/notifications, /notifications/socket.io)

Backend PHP ──(red Docker, x-api-key)──► notifications:3002  /send, /broadcast
   · POST /notificaciones/enviar     (cualquier usuario → usuario de su tenant)
   · POST /notificaciones/broadcast  (lista blanca)
```

Reglas de oro:
1. El navegador **nunca** conoce `API_KEY`; solo el backend (red Docker interna).
2. `tenant` y `userId` **siempre** salen del token verificado, nunca de la URL ni del body.
3. `/send`, `/send-bulk`, `/broadcast` del microservicio no se exponen en el proxy público.

## 4. Plan paso a paso

### Fase 0 – Documentación (este archivo)

### Fase 1 – Microservicio Node (`services/microservices/notifications_service`)
1. `config/env.js`: sin valores por defecto en producción; aborta si faltan `API_KEY`, `MONGO_URI`, `NOTIFICATIONS_JWT_SECRET` (≥32 chars), `CORS_ORIGINS`.
2. `apiKeyAuth`: comparación en tiempo constante (`crypto.timingSafeEqual`).
3. Nuevo `middleware/jwtAuth.js` (REST) y `verifyNotificationToken` reutilizado por el handshake de Socket.io (HS256, `iss`, `aud`).
4. Rutas REST de usuario derivadas del token: `GET /api/notifications`, `GET /unread-count`, `PATCH /read-all`, `PATCH /:id/read` (filtra por `_id + tenant + recipientId`). Se eliminan las rutas públicas `/:tenant/:userId`.
5. Socket.io: `io.use()` valida el token, une al room `user:{tenant}:{userId}` y programa la desconexión al expirar; se elimina el evento `register`.
6. Validación/sanitización de entrada (tipos string, formato de ids, enums, tamaño de `metadata`, máx. destinatarios en bulk, límite de body).
7. `package.json` (+`jsonwebtoken`), `.gitignore` (+`node_modules`), `.env.example`.

### Fase 2 – Backend PHP (`services/ITwebService_Back`)
1. `firmaTokensAccesoYRefresh`: agregar `idusuario` a la respuesta de login.
2. `services/Notificaciones/NotificacionesService.php`: firma del token de notificaciones, validación de destinatario, autorización de broadcast.
3. `controllers/NotificacionesController.php` + rutas `GET /notificaciones/getToken`, `POST /notificaciones/enviar`, `POST /notificaciones/broadcast`.
4. `composer.json`: namespace `Services\Notificaciones\`. `.env.example`: variables nuevas.

### Fase 3 – Frontend (`services/ITwebService_Front`)
1. `notificationsURL.js` por variable de entorno (`VITE_NOTIFICATIONS_URL`); `Dockerfile.prod` + compose pasan el build-arg.
2. Nuevo gestor del token de notificaciones (caché, renovación, reintento en 401).
3. `notificationSocket.js`: `auth` como función (token fresco en cada reconexión), reconexión al expirar, sin `register`.
4. `notificationsThunks.js`: Bearer en REST, rutas sin tenant/userId, `sendManualNotification` → thunk al backend; thunk de broadcast. Se elimina `VITE_NOTIFICATIONS_API_KEY`.
5. `idusuario` en `authSlice` / sesión.

### Fase 4 – Infraestructura y documentación
1. `docker-compose.yml`: `NOTIFICATIONS_JWT_SECRET`, build-arg del front, `NODE_ENV` productivo.
2. `nginx/nginx.conf`: bloquear `/api/notifications/{send,send-bulk,broadcast}` hacia fuera + `limit_req`; snippet para el nginx del servidor (`crm.itwebservices.net`).
3. `.env.example` raíz, `DOCKER.md`, `NOTIFICATIONS.md` actualizados.

### Fase 5 – Verificación
- Sintaxis (`php -l`, `node --check`).
- Interoperabilidad: token firmado por PHP verificado por Node.
- Pruebas negativas del microservicio (sin token, token de otro `aud`, expirado, socket sin token, `/send` con key incorrecta).
- `grep` del bundle/código: no queda ninguna API key en el front.

## 5. Despliegue (orden obligatorio)
1. Generar secretos nuevos y cargarlos (ver `01-cambios.md` §Checklist).
2. Desplegar **juntos** microservicio + backend + front (las rutas viejas del microservicio desaparecen).
3. Añadir los `location` al nginx del servidor y recargar.
4. Rotar credenciales de Mongo/API key (el `.env` del microservicio estuvo versionado).

## 6. Fuera de alcance (siguiente iteración)
TTL/retención en Mongo, rate-limit propio del microservicio, apagado ordenado (`SIGTERM`), adapter Redis para varias réplicas, persistencia del broadcast, logs estructurados, pruebas automatizadas.
