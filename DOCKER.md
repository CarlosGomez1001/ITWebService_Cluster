# Docker - Guía de Comandos y Despliegue

Documentación completa para el manejo de contenedores del IT Web Service Cluster.

## Índice

- [Requisitos Previos](#requisitos-previos)
- [Arquitectura de Servicios](#arquitectura-de-servicios)
- [Estructura de Archivos](#estructura-de-archivos)
- [Configuración Inicial](#configuración-inicial)
- [Comandos de Desarrollo](#comandos-de-desarrollo)
- [Comandos de Producción](#comandos-de-producción)
- [Gestión de Servicios](#gestión-de-servicios)
- [Microservicio de Notificaciones](#microservicio-de-notificaciones)
- [Logs y Monitoreo](#logs-y-monitoreo)
- [Networking](#networking)
- [Volúmenes y Persistencia](#volúmenes-y-persistencia)
- [Despliegue en Servidor Linux](#despliegue-en-servidor-linux)
- [Troubleshooting](#troubleshooting)

---

## Requisitos Previos

- Docker Engine 20.10+
- Docker Compose V2+
- 4GB RAM mínimo disponible (8GB recomendado para stack completo)
- Puertos disponibles: 3000, 3001, 3002, 5173, 80, 443, 27017

Verificar instalación:
```bash
docker --version
docker compose version
```

---

## Arquitectura de Servicios

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     Docker Network: it-web-service-network                   │
│                                                                             │
│  ┌──────────────┐  ┌───────────────┐  ┌────────────────┐                   │
│  │  it-backend  │  │ it-frontend-  │  │ it-frontend-   │                   │
│  │  PHP 8.2     │  │ dev (Vite)    │  │ prod (Nginx)   │                   │
│  │  127.0.0.1:  │  │ :5173→:5173   │  │ 127.0.0.1:     │                   │
│  │  3000→:80    │  │               │  │ 8081→:80       │                   │
│  └──────┬───────┘  └───────────────┘  └────────────────┘                   │
│         │                                                                   │
│         │ HTTP (API Key)                                                    │
│         ▼                                                                   │
│  ┌──────────────────┐       ┌─────────────────────┐                        │
│  │ it-ms-notifications│◄────│ it-mongo-notifications│                       │
│  │ Node.js 20        │      │ MongoDB 7             │                       │
│  │ 127.0.0.1:3002    │      │ :27017→:27017         │                       │
│  │ (REST + Socket.io)│      │                       │                       │
│  └──────────────────┘       └─────────────────────┘                        │
│                                                                             │
│  ┌──────────────────┐                                                      │
│  │ it-ms-formatsandmail│                                                   │
│  │ PHP 8.2           │                                                     │
│  │ 127.0.0.1:3001    │                                                     │
│  └──────────────────┘                                                      │
└─────────────────────────────────────────────────────────────────────────────┘
                              ▲
                              │ 127.0.0.1:<port>
                              │
                  ┌───────────────────────┐
                  │  reverse-proxy (../)   │  standalone project,
                  │  network_mode: host    │  network_mode: host
                  │  :80 / :443            │  routes to any local app
                  └───────────────────────┘
```

> El reverse-proxy **ya no vive en este repo**. Es un proyecto independiente
> (`../reverse-proxy`) que corre con `network_mode: host` y llega a cada
> servicio vía `127.0.0.1:<puerto>`, para poder enrutar también otros
> microservicios del mismo servidor que no pertenecen a este cluster. Ver su
> propio `README.md`.

### Servicios y Profiles

| Servicio | Container | Profile | Puerto (host) | Stack |
|----------|-----------|---------|--------|-------|
| `backend` | it-backend | *(siempre)* | 127.0.0.1:3000 | PHP 8.2 + Apache + SQL Server ODBC |
| `frontend-dev` | it-frontend-dev | `dev` | 5173 | Vite dev server (hot reload) |
| `frontend-prod` | it-frontend-prod | `prod` | 127.0.0.1:8081 | Nginx + React build estático |
| `notifications` | it-ms-notifications | `notifications`, `microservices` | 127.0.0.1:3002 | Node.js 20 + Express + Socket.io |
| `mongo-notifications` | it-mongo-notifications | `notifications`, `microservices` | 27017 | MongoDB 7 |
| `formatsandmail` | it-ms-formatsandmail | `formatsandmail`, `microservices` | 127.0.0.1:3001 | PHP 8.2 + Apache |

El proxy público (80/443) vive en `../reverse-proxy` — ver su `README.md`.

---

## Estructura de Archivos

```
it_web_service-cluster/
├── docker-compose.yml              # Orquestador principal del cluster
├── .env                            # Variables de entorno (crear desde .env.example)
├── .env.example                    # Template de variables
├── nginx/                           # LEGACY, sin usar: el reverse proxy activo
│   ├── nginx.conf                  # ahora vive en ../reverse-proxy (fuera de este repo)
│   └── ssl/
├── it_web_service-Front/
│   ├── Dockerfile                  # Imagen desarrollo
│   ├── Dockerfile.prod             # Imagen producción (multi-stage)
│   └── nginx.conf                  # Nginx config para frontend
├── it_web_service-Backend_R/
│   ├── Dockerfile                  # Imagen PHP + Apache
│   └── .env                        # Variables del backend
└── microservices/
    ├── Notifications/
    │   ├── Dockerfile              # Multi-stage Node.js 20 Alpine
    │   ├── .env                    # Variables del microservicio
    │   ├── .env.example            # Template de variables
    │   ├── package.json            # Dependencias Node.js
    │   ├── NOTIFICATIONS.md        # Documentación completa del API
    │   ├── MONGO_MAINTENANCE.md    # Guía de mantenimiento MongoDB
    │   └── src/
    │       ├── server.js           # Entry point (Express + Socket.io)
    │       ├── config/             # database.js, env.js, socket.js
    │       ├── controllers/        # notificationController.js
    │       ├── middleware/          # apiKeyAuth.js, errorHandler.js
    │       ├── models/             # Notification.js (Mongoose)
    │       ├── routes/             # notifications.js
    │       └── services/           # socketService.js
    └── FormatsAndMails/
        ├── Dockerfile              # PHP 8.2 + Apache
        └── .env                    # Variables del microservicio
```

---

## Configuración Inicial

### 1. Crear archivo de entorno

```bash
cp .env.example .env
```

### 2. Editar variables según ambiente

```bash
# .env
NODE_ENV=development                # o 'production'

# ─── Puertos principales (todos loopback-only, salvo frontend-dev) ─
BACKEND_PORT=3000
FRONTEND_DEV_PORT=5173
FRONTEND_PROD_PORT=8081
VITE_API_URL=http://localhost:3000/index.php
# El puerto público 80/443 lo expone ../reverse-proxy, no este repo.

# ─── Microservicios ──────────────────────────────────────────────
MS_NOTIFICATIONS_PORT=3002
MONGO_NOTIFICATIONS_PORT=27017
MS_FORMATSANDMAIL_PORT=3001

# ─── Notifications Config ────────────────────────────────────────
NOTIFICATIONS_API_KEY=changeme-generate-a-secure-key
NOTIFICATIONS_CORS_ORIGINS=http://localhost:5173,http://localhost:80

# ─── Docker ──────────────────────────────────────────────────────
COMPOSE_PROJECT_NAME=it-web-service
```

### 3. Configurar backend

```bash
cd it_web_service-Backend_R
cp .env.example .env
# Editar credenciales de SQL Server
```

### 4. Configurar microservicio de notificaciones

```bash
cd microservices/Notifications
cp .env.example .env
# Editar API_KEY y CORS_ORIGINS según ambiente
```

---

## Comandos de Desarrollo

### Levantar ambiente de desarrollo

```bash
# Solo Frontend (Vite dev server) + Backend
docker compose --profile dev up -d

# Con Notificaciones (Backend + Frontend + Notifications + MongoDB)
docker compose --profile dev --profile notifications up -d

# Con todos los microservicios (Notifications + FormatsAndMail + etc.)
docker compose --profile dev --profile microservices up -d

# Combinación específica
docker compose --profile dev --profile notifications --profile formatsandmail up -d
```

| Servicio | URL | Descripción |
|----------|-----|-------------|
| Frontend | http://localhost:5173 | Vite con hot reload |
| Backend | http://localhost:3000 | PHP API |
| Notifications | http://localhost:3002 | REST API + WebSocket |
| MongoDB | localhost:27017 | Base de datos Notifications |
| FormatsAndMail | http://localhost:3001 | Generación de PDFs |

### Reconstruir servicios

```bash
# Rebuild completo con notificaciones
docker compose --profile dev --profile notifications up -d --build

# Rebuild sin cache (forzar cambios en Dockerfile)
docker compose --profile dev --profile notifications build --no-cache \
  && docker compose --profile dev --profile notifications up -d

# Rebuild de un solo servicio
docker compose up -d --build backend
docker compose --profile dev up -d --build frontend-dev
docker compose --profile notifications up -d --build notifications
docker compose --profile formatsandmail up -d --build formatsandmail
```

### Levantar solo backend

```bash
docker compose up -d backend
```

### Reconstruir después de cambios en dependencias

```bash
# Si modificaste package.json (frontend)
docker compose --profile dev up -d --build frontend-dev

# Si modificaste composer.json (backend)
docker compose up -d --build backend

# Si modificaste package.json (notifications)
docker compose --profile notifications up -d --build notifications
```

### Modo interactivo (ver logs en tiempo real)

```bash
docker compose --profile dev up
# Ctrl+C para detener
```

### Limpiar red huérfana y reconstruir

Si aparece el error `network ... not found`:

```bash
docker compose down && docker network prune -f \
  && docker compose --profile dev --profile notifications up -d --build
```

---

## Comandos de Producción

### Build y deploy estándar

```bash
# Solo Frontend + Backend
docker compose --profile prod up -d --build

# Con notificaciones
docker compose --profile prod --profile notifications up -d --build

# Con todos los microservicios
docker compose --profile prod --profile microservices up -d --build
```

Todos los puertos de abajo son loopback-only (`127.0.0.1:<puerto>`), pensados
para ser consumidos por el reverse-proxy, no accedidos directo desde fuera:

| Servicio | URL (loopback) | Descripción |
|----------|-----|-------------|
| Frontend | http://127.0.0.1:8081 | Nginx + assets estáticos |
| Backend | http://127.0.0.1:3000 | PHP API |
| Notifications | http://127.0.0.1:3002 | REST API + WebSocket |
| FormatsAndMail | http://127.0.0.1:3001 | Generación de PDFs |

### Punto de entrada único (reverse proxy)

El reverse proxy es un proyecto **independiente** fuera de este repo
(`../reverse-proxy`), que se levanta y actualiza por separado:

```bash
cd ../reverse-proxy
docker compose up -d --build
```

| Servicio | URL | Descripción |
|----------|-----|-------------|
| Proxy | http://localhost | Entrada unificada (puerto 80/443 del server) |
| `/crm/*`, `/logistica/*` | → Backend | API routes |
| `/*` | → Frontend | Aplicación React |

Ver `../reverse-proxy/README.md` para agregar más apps/microservicios al mismo proxy.

### Build con API URL personalizada

```bash
# Pasar URL de API en tiempo de build
docker compose --profile prod build \
  --build-arg VITE_API_URL=https://api.midominio.com/index.php
```

### Solo rebuild de un servicio

```bash
docker compose --profile prod build frontend-prod
docker compose --profile prod up -d frontend-prod
```

---

## Gestión de Servicios

### Ver estado de servicios

```bash
docker compose ps
docker compose ps -a  # Incluye detenidos
```

### Iniciar/Detener/Reiniciar

```bash
# Todos los servicios activos
docker compose stop
docker compose start
docker compose restart

# Servicio específico
docker compose restart backend
docker compose stop frontend-dev
```

### Detener y eliminar contenedores

```bash
# Detener y remover contenedores
docker compose down

# También remover volúmenes (¡cuidado! borra datos)
docker compose down -v

# También remover imágenes
docker compose down --rmi all
```

### Escalar servicios (múltiples instancias)

```bash
# 3 instancias del backend (requiere load balancer)
docker compose up -d --scale backend=3
```

---

## Microservicio de Notificaciones

El microservicio de notificaciones proporciona notificaciones en tiempo real usando REST API y WebSocket (Socket.io).

### Stack técnico

| Componente | Tecnología |
|-----------|------------|
| Runtime | Node.js 20 (Alpine) |
| Framework | Express 4.21 |
| WebSocket | Socket.io 4.8 |
| Base de datos | MongoDB 7 |
| ODM | Mongoose 8.9 |

### Dockerfile (Multi-stage)

El servicio usa un build multi-stage optimizado para producción:
- **Stage 1 (deps):** Instala solo dependencias de producción
- **Stage 2 (production):** Copia dependencias, ejecuta como usuario no-root (`appuser`)
- Health check integrado en `/notifications/health`

### Variables de entorno

| Variable | Descripción | Default |
|----------|-------------|---------|
| `PORT` | Puerto interno del servicio | `3002` |
| `NODE_ENV` | Ambiente de ejecución | `development` |
| `MONGO_URI` | URI de conexión a MongoDB | `mongodb://mongo-notifications:27017/notifications` |
| `API_KEY` | Clave para autenticación backend→microservicio | *(requerido)* |
| `CORS_ORIGINS` | Orígenes permitidos (separados por coma) | `http://localhost:5173,http://localhost:80` |

### Endpoints

**Protegidos por API Key** (solo accesibles desde el backend PHP):

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| POST | `/api/notifications/send` | Enviar notificación a un usuario |
| POST | `/api/notifications/send-bulk` | Enviar a múltiples usuarios |
| POST | `/api/notifications/broadcast` | Broadcast a todos/tenant |

**Públicos** (accesibles desde el frontend):

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| GET | `/api/notifications/:tenant/:userId` | Obtener notificaciones (paginado) |
| GET | `/api/notifications/:tenant/:userId/unread-count` | Contador de no leídas |
| PATCH | `/api/notifications/:id/read` | Marcar como leída |
| PATCH | `/api/notifications/:tenant/:userId/read-all` | Marcar todas como leídas |
| GET | `/notifications/health` | Health check del servicio |

### Eventos WebSocket (Socket.io)

| Evento | Dirección | Descripción |
|--------|-----------|-------------|
| `register` | Cliente → Servidor | Registrar usuario en sala `user:{tenant}:{userId}` |
| `new-notification` | Servidor → Cliente | Nueva notificación recibida |
| `unread-count` | Servidor → Cliente | Actualización del contador |
| `broadcast-notification` | Servidor → Todos | Alerta a nivel sistema/tenant |

### Comandos específicos

```bash
# Levantar solo Notifications + MongoDB
docker compose --profile notifications up -d

# Rebuild del microservicio
docker compose --profile notifications up -d --build notifications

# Ver logs del servicio
docker compose logs -f notifications

# Ver logs de MongoDB
docker compose logs -f mongo-notifications

# Shell dentro del contenedor
docker compose exec notifications sh

# Verificar health check
curl http://localhost:3002/notifications/health

# Conectar a MongoDB (desde host)
docker compose exec mongo-notifications mongosh notifications
```

### Documentación adicional

- Documentación completa del API: `microservices/Notifications/NOTIFICATIONS.md`
- Mantenimiento de MongoDB: `microservices/Notifications/MONGO_MAINTENANCE.md`

---

## Logs y Monitoreo

### Ver logs

```bash
# Todos los servicios
docker compose logs

# Servicio específico
docker compose logs backend
docker compose logs frontend-dev

# Seguir logs en tiempo real
docker compose logs -f
docker compose logs -f backend

# Últimas N líneas
docker compose logs --tail=100 backend
```

### Monitorear recursos

```bash
# Uso de CPU/memoria en tiempo real
docker stats

# Inspeccionar contenedor
docker inspect it-backend
```

### Ejecutar comandos en contenedor

```bash
# Shell interactivo
docker compose exec backend bash
docker compose exec frontend-dev sh

# Comando específico
docker compose exec backend php -v
docker compose exec frontend-dev npm list
```

### Health checks

```bash
# Ver estado de salud
docker inspect --format='{{.State.Health.Status}}' it-backend

# Endpoint de health (si está configurado)
curl http://localhost:3000/health
curl http://localhost/health
```

---

## Networking

### Red por defecto

Todos los servicios se conectan a `it-web-service-network`:

```bash
# Ver redes
docker network ls

# Inspeccionar red
docker network inspect it-web-service-network
```

### Comunicación entre servicios

Dentro de la red Docker, los servicios se comunican por nombre de contenedor:

| Desde | Hacia | URL | Autenticación |
|-------|-------|-----|---------------|
| frontend-dev | backend | http://backend:80 | JWT (Authorization header) |
| backend | notifications | http://notifications:3002 | API Key (x-api-key header) |
| frontend-dev/prod | notifications | http://localhost:3002 | WebSocket (register event) |

`../reverse-proxy` no está en `it-web-service-network` — corre con `network_mode: host` y llega a `backend`/`frontend-prod`/`notifications` vía `127.0.0.1:<puerto>`, no por nombre de contenedor.
| notifications | mongo-notifications | mongodb://mongo-notifications:27017 | Sin auth (red interna) |

### Flujo de comunicación con Notifications

```
Frontend (React)                Backend (PHP)              Notifications (Node.js)
     │                              │                            │
     │                              │  POST /api/notifications/  │
     │                              │  send                      │
     │                              │  x-api-key: <API_KEY>      │
     │                              │ ──────────────────────────► │
     │                              │                            │ → MongoDB
     │  WebSocket: new-notification │                            │
     │ ◄─────────────────────────────────────────────────────────│
     │                              │                            │
```

---

## Volúmenes y Persistencia

### Volúmenes definidos

| Volumen | Propósito |
|---------|-----------|
| `it-backend-vendor` | Dependencias PHP del backend (composer) |
| `it-frontend-node-modules` | Dependencias Node.js del frontend |
| `it-mongo-notifications-data` | Datos persistentes de MongoDB (notificaciones) |
| `it-formatsandmail-vendor` | Dependencias PHP de FormatsAndMail (composer) |
| `it-formatsandmail-cache` | Cache de PDFs generados |

### Gestionar volúmenes

```bash
# Listar volúmenes
docker volume ls

# Inspeccionar
docker volume inspect it-backend-vendor

# Eliminar volumen específico
docker volume rm it-backend-vendor

# Eliminar volúmenes huérfanos
docker volume prune
```

### Backup de volúmenes

```bash
# Backup
docker run --rm -v it-backend-vendor:/data -v $(pwd):/backup \
  alpine tar cvf /backup/vendor-backup.tar /data

# Restore
docker run --rm -v it-backend-vendor:/data -v $(pwd):/backup \
  alpine tar xvf /backup/vendor-backup.tar -C /
```

---

## Despliegue en Servidor Linux

Guía completa para desplegar el stack completo (Backend + Frontend + Microservicios) en un servidor Linux.

### 1. Preparación del servidor

#### Requisitos del servidor

- Ubuntu 22.04+ / Debian 12+ / CentOS 9+
- 4GB RAM mínimo (8GB recomendado)
- 20GB disco disponible
- Acceso SSH
- Puertos abiertos: 80, 443, 3000, 3001, 3002

#### Instalar Docker y Docker Compose

```bash
# Actualizar paquetes
sudo apt update && sudo apt upgrade -y

# Instalar dependencias
sudo apt install -y ca-certificates curl gnupg lsb-release

# Agregar clave GPG de Docker
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Agregar repositorio de Docker
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Instalar Docker Engine + Compose
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Agregar usuario al grupo docker (evitar sudo)
sudo usermod -aG docker $USER
newgrp docker

# Verificar instalación
docker --version
docker compose version
```

#### Configurar firewall

```bash
# UFW (Ubuntu)
sudo ufw allow 22/tcp     # SSH
sudo ufw allow 80/tcp     # Frontend
sudo ufw allow 443/tcp    # Frontend SSL
sudo ufw allow 3000/tcp   # Backend API
sudo ufw allow 3001/tcp   # FormatsAndMail
sudo ufw allow 3002/tcp   # Notifications (REST + WebSocket)
sudo ufw enable
sudo ufw status
```

### 2. Clonar y configurar el proyecto

```bash
# Crear directorio de la aplicación
sudo mkdir -p /opt/it-web-service
sudo chown $USER:$USER /opt/it-web-service
cd /opt/it-web-service

# Clonar repositorios
git clone <url-repo-cluster> .
git clone <url-repo-frontend> it_web_service-Front
git clone <url-repo-backend> it_web_service-Backend_R
git clone <url-repo-notifications> microservices/Notifications
git clone <url-repo-formatsandmail> microservices/FormatsAndMails
```

#### O transferir desde máquina local

```bash
# Desde tu máquina local
rsync -avz --exclude='node_modules' --exclude='vendor' --exclude='.git' \
  /path/to/it_web_service-cluster/ usuario@servidor:/opt/it-web-service/
```

### 3. Configurar variables de entorno

#### Archivo `.env` del cluster

```bash
cd /opt/it-web-service
cp .env.example .env
```

Editar `.env` con los valores de producción:

```bash
# .env (producción)
NODE_ENV=production
COMPOSE_PROJECT_NAME=it-web-service

# ─── Puertos (loopback-only, el proxy público vive en ../reverse-proxy) ──
BACKEND_PORT=3000
FRONTEND_PROD_PORT=8081

# ─── Microservicios ──────────────────────────────────────────────
MS_NOTIFICATIONS_PORT=3002
MONGO_NOTIFICATIONS_PORT=27017
MS_FORMATSANDMAIL_PORT=3001

# ─── Notifications ───────────────────────────────────────────────
# IMPORTANTE: Generar una clave segura con: openssl rand -hex 32
NOTIFICATIONS_API_KEY=<tu-clave-segura-aqui>
NOTIFICATIONS_CORS_ORIGINS=https://tu-dominio.com

# ─── Frontend Build ──────────────────────────────────────────────
VITE_API_URL=https://tu-dominio.com/index.php
```

#### Archivo `.env` del backend

```bash
cd it_web_service-Backend_R
cp .env.example .env
# Configurar credenciales de SQL Server, SEC_KEY, ENCRYPT_KEY, etc.
```

#### Archivo `.env` de notificaciones

```bash
cd microservices/Notifications
cp .env.example .env
# Configurar API_KEY (misma que NOTIFICATIONS_API_KEY del cluster)
# Configurar CORS_ORIGINS con el dominio de producción
```

### 4. Generar claves seguras

```bash
# API Key para Notifications
openssl rand -hex 32

# Copiar el resultado en:
# - .env del cluster → NOTIFICATIONS_API_KEY
# - microservices/Notifications/.env → API_KEY
# - it_web_service-Backend_R/.env → NOTIFICATIONS_API_KEY (si aplica)
```

### 5. Desplegar todos los servicios

#### Deploy completo (todos los servicios)

```bash
cd /opt/it-web-service

# Build y levantar: Frontend(prod) + Backend + Notifications + FormatsAndMail
docker compose --profile prod --profile microservices up -d --build
```

#### Deploy por partes (recomendado para primera vez)

```bash
# Paso 1: Backend primero
docker compose up -d --build backend
docker compose logs -f backend
# Verificar: curl http://localhost:3000/index.php

# Paso 2: MongoDB para notificaciones
docker compose --profile notifications up -d mongo-notifications
docker compose logs -f mongo-notifications
# Esperar a que el health check pase

# Paso 3: Microservicio de notificaciones
docker compose --profile notifications up -d --build notifications
docker compose logs -f notifications
# Verificar: curl http://localhost:3002/notifications/health

# Paso 4: FormatsAndMail
docker compose --profile formatsandmail up -d --build formatsandmail
docker compose logs -f formatsandmail
# Verificar: curl http://localhost:3001/index.php/health

# Paso 5: Frontend producción
docker compose --profile prod up -d --build frontend-prod
docker compose logs -f frontend-prod
# Verificar: curl http://127.0.0.1:8081

# Paso 6: Reverse proxy (proyecto independiente, fuera de este repo)
cd ../reverse-proxy
docker compose up -d --build
docker compose logs -f
```

#### Verificar que todo está corriendo

```bash
# Estado de todos los contenedores del cluster
docker compose ps

# Health checks (loopback, como los ve el proxy)
curl -s http://127.0.0.1:3000/index.php        # Backend
curl -s http://127.0.0.1:3002/notifications/health  # Notifications
curl -s http://127.0.0.1:3001/index.php/health      # FormatsAndMail
curl -s http://127.0.0.1:8081                        # Frontend

# Entrada pública, vía el reverse proxy
curl -s http://localhost                             # Proxy -> Frontend
```

### 6. Configurar SSL con Certbot (Let's Encrypt)

El puerto 80 ahora lo posee el reverse proxy (`../reverse-proxy`), no este repo.
Este paso se hace en el proyecto del proxy:

```bash
cd ../reverse-proxy

# Instalar Certbot
sudo apt install -y certbot

# Detener temporalmente el proxy para liberar el puerto 80
docker compose stop

# Obtener certificado
sudo certbot certonly --standalone -d tu-dominio.com

# Copiar certificados al proyecto del proxy
sudo cp /etc/letsencrypt/live/tu-dominio.com/fullchain.pem nginx/ssl/
sudo cp /etc/letsencrypt/live/tu-dominio.com/privkey.pem nginx/ssl/

# Descomentar el server block HTTPS en nginx/conf.d/it-web-service-cluster.conf,
# ajustar server_name, y levantar de nuevo
docker compose up -d
```

### 7. Configurar inicio automático (systemd)

Crear un servicio systemd por proyecto, para que cada uno tenga su propio
ciclo de vida — el proxy no debería reiniciarse solo porque el cluster se
actualiza, y viceversa:

```bash
sudo tee /etc/systemd/system/it-web-service.service > /dev/null <<EOF
[Unit]
Description=IT Web Service Cluster
Requires=docker.service
After=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/opt/it-web-service
ExecStart=/usr/bin/docker compose --profile prod --profile microservices up -d
ExecStop=/usr/bin/docker compose down
TimeoutStartSec=300

[Install]
WantedBy=multi-user.target
EOF

sudo tee /etc/systemd/system/reverse-proxy.service > /dev/null <<EOF
[Unit]
Description=Standalone Reverse Proxy
Requires=docker.service
After=docker.service it-web-service.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/opt/reverse-proxy
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down
TimeoutStartSec=300

[Install]
WantedBy=multi-user.target
EOF

# Habilitar e iniciar ambos
sudo systemctl daemon-reload
sudo systemctl enable it-web-service reverse-proxy
sudo systemctl start it-web-service reverse-proxy

# Verificar estado
sudo systemctl status it-web-service reverse-proxy
```

### 8. Renovación automática de certificados SSL

```bash
# Crear script de renovación (vive junto al proxy, no al cluster)
sudo tee /opt/reverse-proxy/renew-ssl.sh > /dev/null <<'EOF'
#!/bin/bash
certbot renew --quiet
cp /etc/letsencrypt/live/tu-dominio.com/fullchain.pem /opt/reverse-proxy/nginx/ssl/
cp /etc/letsencrypt/live/tu-dominio.com/privkey.pem /opt/reverse-proxy/nginx/ssl/
cd /opt/reverse-proxy
docker compose restart
EOF

sudo chmod +x /opt/reverse-proxy/renew-ssl.sh

# Cron job (cada mes a las 3am)
(crontab -l 2>/dev/null; echo "0 3 1 * * /opt/reverse-proxy/renew-ssl.sh") | crontab -
```

### 9. Actualizar servicios en producción

#### Actualización estándar (con downtime mínimo)

```bash
cd /opt/it-web-service

# Actualizar código fuente
git pull
cd it_web_service-Front && git pull && cd ..
cd it_web_service-Backend_R && git pull && cd ..
cd microservices/Notifications && git pull && cd ../..

# Rebuild y deploy
docker compose --profile prod --profile microservices up -d --build
```

#### Actualizar solo un servicio (zero downtime para el resto)

```bash
# Solo backend
docker compose up -d --build backend

# Solo frontend
docker compose --profile prod up -d --build frontend-prod

# Solo notifications
docker compose --profile notifications up -d --build notifications

# Solo formatsandmail
docker compose --profile formatsandmail up -d --build formatsandmail
```

#### Rollback de emergencia

```bash
# Ver imágenes anteriores
docker images | grep it-

# Detener y eliminar contenedor problemático
docker compose stop notifications
docker compose rm -f notifications

# Volver a la versión anterior del código
cd microservices/Notifications && git checkout <commit-anterior> && cd ../..

# Rebuild
docker compose --profile notifications up -d --build notifications
```

### 10. Monitoreo en producción

```bash
# Estado general
docker compose ps
docker stats --no-stream

# Logs en tiempo real de todos los servicios
docker compose logs -f

# Logs de un servicio específico
docker compose logs -f --tail=200 notifications
docker compose logs -f --tail=200 mongo-notifications

# Espacio en disco
docker system df
docker volume ls

# Limpieza periódica (imágenes y contenedores sin usar)
docker system prune -f
docker image prune -f
```

### Resumen de puertos en producción

| Puerto | Servicio | Protocolo |
|--------|----------|-----------|
| 80 | Frontend (Nginx) | HTTP |
| 443 | Frontend (Nginx) | HTTPS |
| 3000 | Backend API (PHP) | HTTP |
| 3001 | FormatsAndMail (PHP) | HTTP |
| 3002 | Notifications (REST + WebSocket) | HTTP/WS |
| 27017 | MongoDB | TCP (solo acceso interno recomendado) |

---

## Troubleshooting

### Problema: Puerto en uso

```bash
# Verificar qué usa el puerto
lsof -i :3000
# o
netstat -tulpn | grep 3000

# Cambiar puerto en .env
BACKEND_PORT=3001
```

### Problema: Contenedor no inicia

```bash
# Ver logs de error
docker compose logs backend

# Ver eventos
docker events --filter container=it-backend
```

### Problema: Cambios no reflejados

```bash
# Rebuild forzado sin cache
docker compose build --no-cache backend
docker compose up -d backend
```

### Problema: Sin espacio en disco

```bash
# Limpiar recursos no utilizados
docker system prune -a

# Ver uso de disco
docker system df
```

### Problema: Error de permisos

```bash
# En el contenedor backend
docker compose exec backend chown -R www-data:www-data /var/www/html
```

### Problema: Conexión a SQL Server fallida

```bash
# Verificar conectividad desde contenedor
docker compose exec backend bash
sqlcmd -S IP_SERVER -U USER -P PASS -Q "SELECT 1"

# Verificar variables de entorno
docker compose exec backend env | grep IP_SERVER
```

### Reset completo

```bash
# Detener todo, eliminar contenedores, volúmenes e imágenes
docker compose down -v --rmi all

# Reconstruir desde cero
docker compose --profile dev up -d --build
```

---

## Comandos Rápidos de Referencia

### Desarrollo

| Acción | Comando |
|--------|---------|
| Dev básico | `docker compose --profile dev up -d` |
| Dev + Notifications | `docker compose --profile dev --profile notifications up -d` |
| Dev + todos los MS | `docker compose --profile dev --profile microservices up -d` |
| Rebuild dev completo | `docker compose --profile dev --profile microservices up -d --build` |

### Producción

| Acción | Comando |
|--------|---------|
| Prod básico | `docker compose --profile prod up -d --build` |
| Prod + Notifications | `docker compose --profile prod --profile notifications up -d --build` |
| Prod + todos los MS | `docker compose --profile prod --profile microservices up -d --build` |
| Proxy (proyecto separado) | `cd ../reverse-proxy && docker compose up -d --build` |

### Gestión general

| Acción | Comando |
|--------|---------|
| Estado | `docker compose ps` |
| Ver logs | `docker compose logs -f` |
| Logs de servicio | `docker compose logs -f notifications` |
| Detener todo | `docker compose down` |
| Detener + limpiar | `docker compose down -v --rmi all` |
| Shell backend | `docker compose exec backend bash` |
| Shell notifications | `docker compose exec notifications sh` |
| Shell MongoDB | `docker compose exec mongo-notifications mongosh notifications` |
| Health notifications | `curl http://localhost:3002/notifications/health` |
| Recursos | `docker stats --no-stream` |
| Limpiar | `docker system prune -a` |
