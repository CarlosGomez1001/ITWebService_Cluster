# Docker - Guía de Comandos y Despliegue

Documentación completa para el manejo de contenedores del IT Web Service Cluster.

## Índice

- [Requisitos Previos](#requisitos-previos)
- [Estructura de Archivos](#estructura-de-archivos)
- [Configuración Inicial](#configuración-inicial)
- [Comandos de Desarrollo](#comandos-de-desarrollo)
- [Comandos de Producción](#comandos-de-producción)
- [Gestión de Servicios](#gestión-de-servicios)
- [Logs y Monitoreo](#logs-y-monitoreo)
- [Networking](#networking)
- [Volúmenes y Persistencia](#volúmenes-y-persistencia)
- [Despliegue en la Nube](#despliegue-en-la-nube)
- [Troubleshooting](#troubleshooting)

---

## Requisitos Previos

- Docker Engine 20.10+
- Docker Compose V2+
- 4GB RAM mínimo disponible
- Puertos disponibles: 3000, 5173, 80, 443

Verificar instalación:
```bash
docker --version
docker compose version
```

---

## Estructura de Archivos

```
it_web_service-cluster/
├── docker-compose.yml          # Orquestador principal
├── .env                        # Variables de entorno (crear desde .env.example)
├── .env.example                # Template de variables
├── nginx/
│   ├── nginx.conf              # Configuración reverse proxy
│   └── ssl/                    # Certificados SSL (producción)
├── it_web_service-Front/
│   ├── Dockerfile              # Imagen desarrollo
│   ├── Dockerfile.prod         # Imagen producción (multi-stage)
│   └── nginx.conf              # Nginx config para frontend
└── it_web_service-Backend_R/
    ├── Dockerfile              # Imagen PHP + Apache
    └── .env                    # Variables del backend
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
NODE_ENV=development          # o 'production'
BACKEND_PORT=3000
FRONTEND_DEV_PORT=5173
FRONTEND_PROD_PORT=80
VITE_API_URL=http://localhost:3000/index.php
```

### 3. Configurar backend

```bash
cd it_web_service-Backend_R
cp .env.example .env
# Editar credenciales de SQL Server
```

---

## Comandos de Desarrollo

### Levantar ambiente de desarrollo

```bash
# Frontend (Vite dev server) + Backend
docker compose --profile dev up -d
```

```bash
# Notificaciones + stack completo de desarrollo
docker compose --profile dev --profile notifications up 
```

```bash
# Notificaciones + stack completo de desarrollo build
docker compose --profile dev --profile notifications up -d --build
```

```bash
# Para reconstruir todo (frontend dev + backend + microservicios de notificaciones con MongoDB)
docker compose --profile dev --profile notifications up -d --build


```bash
#Si quieres forzar rebuild sin cache (para asegurar que tome todos los cambios):

docker compose --profile dev --profile notifications build --no-cache && docker compose --profile dev --profile notifications
```

```bash
#Error response from daemon: network e0c21a478c9db0643cd4202aec1a6fe78241e6584772c3291935690eae183704 not found
#Eso pasa cuando Docker tiene una referencia a una red que ya fue eliminada. Limpia las referencias huérfanas y vuelve a levantar:
docker compose down && docker network prune -f && docker compose --profile dev --profile notifications up -d --build

```

| Servicio | URL | Descripción |
|----------|-----|-------------|
| Frontend | http://localhost:5173 | Vite con hot reload |
| Backend | http://localhost:3000 | PHP API |

### Levantar solo backend

```bash
docker compose up -d backend
```

### Reconstruir después de cambios en dependencias

```bash
# Si modificaste package.json
docker compose --profile dev up -d --build frontend-dev

# Si modificaste composer.json
docker compose up -d --build backend
```

### Modo interactivo (ver logs en tiempo real)

```bash
docker compose --profile dev up
# Ctrl+C para detener
```

---

## Comandos de Producción

### Build y deploy estándar

```bash
docker compose --profile prod up -d --build
```

| Servicio | URL | Descripción |
|----------|-----|-------------|
| Frontend | http://localhost:80 | Nginx + assets estáticos |
| Backend | http://localhost:3000 | PHP API |

### Con reverse proxy (punto de entrada único)

```bash
docker compose --profile prod --profile proxy up -d --build
```

| Servicio | URL | Descripción |
|----------|-----|-------------|
| Proxy | http://localhost:8080 | Entrada unificada |
| `/crm/*`, `/logistica/*` | → Backend | API routes |
| `/*` | → Frontend | Aplicación React |

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

Dentro de la red Docker, los servicios se comunican por nombre:

| Desde | Hacia | URL |
|-------|-------|-----|
| frontend-dev | backend | http://backend:80 |
| nginx-proxy | backend | http://backend:80 |
| nginx-proxy | frontend-prod | http://frontend-prod:80 |

### Exponer puertos adicionales

Editar `docker-compose.yml`:
```yaml
services:
  backend:
    ports:
      - "3000:80"
      - "3001:8080"  # Puerto adicional
```

---

## Volúmenes y Persistencia

### Volúmenes definidos

| Volumen | Propósito |
|---------|-----------|
| `it-backend-vendor` | Dependencias PHP (composer) |
| `it-frontend-node-modules` | Dependencias Node.js |

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

## Despliegue en la Nube

### 1. Build de imágenes

```bash
# Build todas las imágenes de producción
docker compose --profile prod build

# Verificar imágenes creadas
docker images | grep it-
```

### 2. Tag y push a registry

```bash
# Docker Hub
docker tag it-frontend-prod:latest usuario/it-frontend:latest
docker tag it-backend:latest usuario/it-backend:latest

docker login
docker push usuario/it-frontend:latest
docker push usuario/it-backend:latest
```

```bash
# AWS ECR
aws ecr get-login-password | docker login --username AWS --password-stdin 123456789.dkr.ecr.region.amazonaws.com

docker tag it-frontend-prod:latest 123456789.dkr.ecr.region.amazonaws.com/it-frontend:latest
docker push 123456789.dkr.ecr.region.amazonaws.com/it-frontend:latest
```

### 3. Docker Compose en servidor remoto

```bash
# Copiar archivos necesarios al servidor
scp docker-compose.yml .env usuario@servidor:/app/
scp -r nginx/ usuario@servidor:/app/

# En el servidor
ssh usuario@servidor
cd /app
docker compose --profile prod up -d
```

### 4. Con Docker Swarm

```bash
# Inicializar swarm
docker swarm init

# Deploy stack
docker stack deploy -c docker-compose.yml it-web-service

# Ver servicios
docker service ls
```

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

| Acción | Comando |
|--------|---------|
| Desarrollo | `docker compose --profile dev up -d` |
| Producción | `docker compose --profile prod up -d --build` |
| Con proxy | `docker compose --profile prod --profile proxy up -d` |
| Ver logs | `docker compose logs -f` |
| Detener | `docker compose down` |
| Rebuild | `docker compose up -d --build` |
| Shell backend | `docker compose exec backend bash` |
| Shell frontend | `docker compose exec frontend-dev sh` |
| Estado | `docker compose ps` |
| Limpiar | `docker system prune -a` |
