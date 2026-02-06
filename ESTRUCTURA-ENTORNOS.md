# 🏗️ Estructura de Entornos - Mejores Prácticas

## 📋 Descripción

Este documento explica la estructura de archivos de configuración Docker por entorno y cómo escalar el proyecto a diferentes ambientes (desarrollo, staging, producción).

---

## 🎯 Entornos Actuales

### Desarrollo (✅ Implementado)

```
Dockerfile.dev              → Imagen con hot reload y debugging
docker-compose.dev.yml      → Configuración de desarrollo
```

**Características:**
- Volúmenes montados para código en vivo (hot reload)
- Puerto 8000 expuesto
- Base de datos con volumen persistente
- Dependencias de desarrollo incluidas
- Logs detallados

**Uso:**
```bash
docker compose -f docker-compose.dev.yml up -d
```

---

## 🚀 Entornos Futuros

### Staging (Por implementar)

```
Dockerfile.staging          → Imagen similar a producción, con debugging
docker-compose.staging.yml  → Configuración de staging
```

**Cambios vs Desarrollo:**
- Sin volúmenes de código (contenedor sellado)
- Mismo puerto 8000
- Credenciales de BD diferentes
- Validaciones más estrictas

### Producción (✅ Implementado)

```
Dockerfile.prod             → Imagen optimizada, sin debugging
docker-compose.prod.yml     → Configuración de producción
```

**Características:**
- Imagen multietapa optimizada
- Usuario no-root por seguridad
- Sin volúmenes de código (contenedor sellado)
- MariaDB 10.6 (compatible con servidor)
- Red independiente
- Nombres únicos para evitar conflictos
- Health checks robustos
- Límites de recursos configurados
- Integración con proxy inverso (VIRTUAL_HOST)
- Reinicio automático
- Logs controlados

**Cambios vs Desarrollo:**
- Sin volúmenes de código (código copiado en build)
- Imagen multietapa (mucho más pequeña)
- MariaDB 10.6 en lugar de MySQL 8.0
- Usuario appuser no-root
- Credenciales en variables de entorno
- Health checks más robustos
- Límites de memoria y CPU
- Sin --reload en uvicorn

---

## 📁 Estructura Recomendada

```
proyecto/
├── Dockerfile.dev           ← Desarrollo (ACTUAL)
├── Dockerfile.staging       ← Staging (futuro)
├── Dockerfile.prod          ← Producción (futuro)
│
├── docker-compose.dev.yml       ← Desarrollo (ACTUAL)
├── docker-compose.staging.yml   ← Staging (futuro)
├── docker-compose.prod.yml      ← Producción (futuro)
│
├── .env.dev                 ← Variables de desarrollo
├── .env.staging             ← Variables de staging
├── .env.prod                ← Variables de producción
│
├── ESTRUCTURA-ENTORNOS.md   ← Este archivo
├── ENTENDIENDO-DOCKER.md
├── README.md
└── ... (resto del código)
```

---

## 🔧 Dockerfile Múltiples Etapas (Ejemplo de Producción)

```dockerfile
# ETAPA 1: Build (construcción)
FROM python:3.11 AS builder

WORKDIR /build
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# ETAPA 2: Runtime (ejecución)
FROM python:3.11-slim

WORKDIR /app

# Copiar solo lo necesario del builder
COPY --from=builder /usr/local/lib/python3.11/site-packages /usr/local/lib/python3.11/site-packages

COPY . .

EXPOSE 8000

# Sin --reload en producción
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Beneficios:**
- Imagen mucho más pequeña
- Sin herramientas de desarrollo
- Más seguro
- Más rápido en despliegue

---

## 🌍 Variables de Entorno por Ambiente

### `.env.dev`
```bash
# Database
DB_HOST=mysql
DB_PORT=3306
DB_USER=usuario
DB_PASSWORD=usuario123
DB_NAME=clientes_db

# Application
DEBUG=true
LOG_LEVEL=DEBUG
RELOAD=true
```

### `.env.prod`
```bash
# Database
DB_HOST=mysql-prod.internal
DB_PORT=3306
DB_USER=prod_user
DB_PASSWORD=<contraseña_segura>
DB_NAME=clientes_prod

# Application
DEBUG=false
LOG_LEVEL=WARNING
RELOAD=false
```

---

## 📝 docker-compose.prod.yml (Ejemplo)

```yaml
version: '3.8'

services:
  mysql:
    image: mysql:8.0
    container_name: clientes-mysql-prod
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: ${DB_NAME}
      MYSQL_USER: ${DB_USER}
      MYSQL_PASSWORD: ${DB_PASSWORD}
    ports:
      - "3306:3306"  # Cambio: puerto estándar
    volumes:
      - ./init_db.sql:/docker-entrypoint-initdb.d/init_db.sql
      - mysql_prod:/var/lib/mysql  # Volumen diferente
    networks:
      - clientes-network-prod
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      timeout: 10s
      retries: 5
    restart: always  # Cambio: reinicio siempre
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 1G

  app:
    build:
      context: .
      dockerfile: Dockerfile.prod  # Cambio: usar Dockerfile.prod
    container_name: clientes-app-prod
    environment:
      DB_HOST: mysql
      DB_PORT: 3306
      DB_USER: ${DB_USER}
      DB_PASSWORD: ${DB_PASSWORD}
      DB_NAME: ${DB_NAME}
      DEBUG: "false"
    ports:
      - "8000:8000"
    # SIN volúmenes de código
    networks:
      - clientes-network-prod
    depends_on:
      mysql:
        condition: service_healthy
    restart: always  # Cambio: reinicio siempre
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 512M

volumes:
  mysql_prod:
    driver: local

networks:
  clientes-network-prod:
    driver: bridge
```

---

## 🛠️ Comandos por Entorno

| Acción | Desarrollo | Staging | Producción |
|--------|-----------|---------|-----------|
| Levantar | `docker compose -f docker-compose.dev.yml up -d` | `docker compose -f docker-compose.staging.yml up -d` | `docker compose -f docker-compose.prod.yml up -d` |
| Ver logs | `docker compose -f docker-compose.dev.yml logs -f` | `docker compose -f docker-compose.staging.yml logs -f` | `docker compose -f docker-compose.prod.yml logs -f` |
| Detener | `docker compose -f docker-compose.dev.yml down` | `docker compose -f docker-compose.staging.yml down` | `docker compose -f docker-compose.prod.yml down` |

---

## 🔐 Seguridad por Entorno

### Desarrollo
- ✅ Credenciales simples (usuario/usuario123)
- ✅ Debug activado
- ✅ Logs detallados
- ✅ Volúmenes de código

### Staging
- ⚠️ Credenciales más seguras
- ⚠️ Debug desactivado
- ⚠️ Logs moderados
- ❌ Sin volúmenes de código

### Producción
- ✅ Credenciales muy seguras (usar secretos)
- ❌ Debug desactivado
- ⚠️ Logs limitados
- ❌ Sin volúmenes de código
- ✅ Restricciones de recursos
- ✅ Reinicio automático
- ✅ Health checks robustos

---

## 📋 Checklist para Producción

- [x] Usar `Dockerfile.prod` con múltiples etapas
- [x] Variables de entorno en docker-compose
- [x] Puertos y servicios publicados correctamente
- [x] Límites de recursos configurados
- [x] Health checks robustos
- [x] Usuario no-root (appuser)
- [x] Red independiente para evitar conflictos
- [x] Nombres únicos de contenedores
- [x] MariaDB 10.6 compatible
- [x] Integración con proxy inverso (VIRTUAL_HOST)
- ⬜ Copias de seguridad de BD configuradas
- ⬜ Logs centralizados
- ⬜ Monitoreo y alertas

---

## 🚀 Plan de Implementación

### Fase 1 (✅ Completada)
- ✅ Estructurar archivos de desarrollo (.dev)

### Fase 2 (✅ Completada)
- ✅ Crear docker-compose.prod.yml
- ✅ Crear Dockerfile.prod (multietapa)
- ✅ Crear documentación de producción

### Fase 3 (Próxima)
- ⬜ Crear docker-compose.staging.yml
- ⬜ Crear Dockerfile.staging
- ⬜ Crear .env.staging

### Fase 4
- ⬜ Integración con GitHub Actions
- ⬜ Despliegue automático

---

## 📚 Referencias

- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [Docker Compose Environments](https://docs.docker.com/compose/environment-variables/)
- [Multi-stage Builds](https://docs.docker.com/build/building/multi-stage/)

---

**Última actualización:** 6 de febrero de 2026
