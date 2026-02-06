# 🚀 Guía de Despliegue en Servidor - clientes-monolito-docker

## 📋 Descripción

Este documento describe cómo desplegar el proyecto `clientes-monolito-docker` en el servidor de producción.

---

## 🎯 Características de la Configuración

✅ **Nombres únicos** para evitar conflictos con otros proyectos
✅ **Red independiente** para aislamiento
✅ **MariaDB 10.6** compatible con servidor
✅ **Proxy inverso** con VIRTUAL_HOST integrado
✅ **Imagen optimizada** con multietapa
✅ **Usuario no-root** por seguridad
✅ **Health checks** automáticos
✅ **Reinicio automático** en caso de fallos

---

## 🏗️ Nombres Únicos (Evitan Conflictos)

```
┌─────────────────────────────────────────────────┐
│  Contenedor App                                  │
│  clientes-monolito-docker-app                   │
└─────────────────────────────────────────────────┘
       ↓ conecta a ↓
┌─────────────────────────────────────────────────┐
│  Contenedor DB                                   │
│  clientes-monolito-docker-db                    │
└─────────────────────────────────────────────────┘
       ↓ guarda datos en ↓
┌─────────────────────────────────────────────────┐
│  Volumen de Datos                                │
│  clientes-monolito-docker-db-data               │
└─────────────────────────────────────────────────┘
       ↓ comunica via ↓
┌─────────────────────────────────────────────────┐
│  Red Docker                                      │
│  clientes-monolito-docker-network               │
└─────────────────────────────────────────────────┘
```

---

## 🚀 Pasos de Despliegue

### 1. Clonar el Repositorio

```bash
cd /ruta/proyectos/docker
git clone https://github.com/sulbaranjc/clientes-monolito-docker.git
cd clientes-monolito-docker
```

### 2. Verificar la Rama de Producción

```bash
git checkout main
git pull origin main
```

### 3. Verificar Archivos Necesarios

```bash
ls -la | grep -E "docker-compose.prod|Dockerfile.prod|init_db.sql"
```

Deberías ver:
```
-rw-r--r-- docker-compose.prod.yml
-rw-r--r-- Dockerfile.prod
-rw-r--r-- init_db.sql
```

### 4. Verificar Espacio y Recursos

```bash
# Espacio en disco
df -h /

# Información del servidor Docker
docker info

# Redes existentes
docker network ls
```

### 5. Construir la Imagen

```bash
docker compose -f docker-compose.prod.yml build
```

**Salida esperada:**
```
[+] Building 45.2s (15/15) FINISHED
```

### 6. Levantar los Servicios

```bash
docker compose -f docker-compose.prod.yml up -d
```

**Salida esperada:**
```
[+] Running 2/2
 ✔ Container clientes-monolito-docker-db  Started
 ✔ Container clientes-monolito-docker-app  Started
```

### 7. Verificar que está Funcionando

```bash
# Ver estado de contenedores
docker compose -f docker-compose.prod.yml ps

# Ver logs de la app
docker compose -f docker-compose.prod.yml logs app

# Ver logs de la BD
docker compose -f docker-compose.prod.yml logs db

# Verificar health status
docker inspect clientes-monolito-docker-app --format='{{.State.Health.Status}}'
docker inspect clientes-monolito-docker-db --format='{{.State.Health.Status}}'
```

Ambos deberían mostrar: `healthy` ✅

### 8. Probar Conectividad

```bash
# Acceder desde dentro del contenedor
docker compose -f docker-compose.prod.yml exec db mariadb -u clientes -pclientes123 clientes_db -e "SELECT 1"

# Verificar API
docker compose -f docker-compose.prod.yml exec app curl http://localhost:8000/docs
```

---

## 🌐 Acceso Externo

### Configuración del Proxy Inverso

El archivo `docker-compose.prod.yml` incluye:

```yaml
environment:
  VIRTUAL_HOST: clientes.docker.sulbaranjc.com
  VIRTUAL_PORT: "8000"
```

Esto requiere que **nginx-proxy** esté configurado en el servidor.

### Verificar Dominio

```bash
# Resolver DNS
nslookup clientes.docker.sulbaranjc.com

# Probar conexión
curl -H "Host: clientes.docker.sulbaranjc.com" http://localhost:8000

# En navegador
https://clientes.docker.sulbaranjc.com/docs
```

---

## 📊 Monitoreo Diario

```bash
# Ver estado general
docker compose -f docker-compose.prod.yml ps

# Ver última hora de logs
docker compose -f docker-compose.prod.yml logs --tail 100 app

# Estadísticas de recursos
docker stats --no-stream clientes-monolito-docker-app clientes-monolito-docker-db

# Verificar volumen de datos
docker volume ls | grep clientes-monolito-docker
du -sh /var/lib/docker/volumes/clientes-monolito-docker-db-data/_data
```

---

## 🔄 Actualizar Código

Si hay cambios en el código:

```bash
# Traer cambios
git pull origin main

# Reconstruir imagen
docker compose -f docker-compose.prod.yml build

# Reiniciar servicios
docker compose -f docker-compose.prod.yml up -d

# Verificar que está bien
docker compose -f docker-compose.prod.yml ps
docker compose -f docker-compose.prod.yml logs app
```

---

## 🛑 Detener Servicios

```bash
# Detener sin eliminar volumen (datos se conservan)
docker compose -f docker-compose.prod.yml down

# Detener y eliminar volumen (CUIDADO: pierde datos)
docker compose -f docker-compose.prod.yml down -v
```

---

## 🔧 Tareas de Mantenimiento

### Backup Manual de Base de Datos

```bash
# Crear backup
docker compose -f docker-compose.prod.yml exec db mariadb-dump \
  -u clientes -pclientes123 clientes_db \
  > backup_clientes_$(date +%Y%m%d_%H%M%S).sql

# Restaurar backup
docker compose -f docker-compose.prod.yml exec -T db mariadb \
  -u clientes -pclientes123 clientes_db \
  < backup_clientes_20260206_123456.sql
```

### Limpiar Contenedores Antiguos

```bash
docker container prune -f
docker image prune -f
```

### Ver Espacio Usado por Proyecto

```bash
docker system df
```

---

## ⚠️ Problemas Comunes y Soluciones

### El contenedor de app no inicia

```bash
# Ver logs detallados
docker compose -f docker-compose.prod.yml logs app

# Posibles causas:
# 1. BD no está lista: esperar 30 segundos
# 2. Puerto 8000 ocupado: verificar con 'lsof -i :8000'
# 3. Dependencias Python faltantes: revisar requirements.txt
```

### La BD no inicia

```bash
docker compose -f docker-compose.prod.yml logs db

# Posibles causas:
# 1. Volumen corrupto: docker volume rm clientes-monolito-docker-db-data
# 2. Permisos: verificar permisos de /var/lib/docker/volumes/
# 3. Espacio en disco: df -h
```

### No puedo acceder por el dominio

```bash
# Verificar que nginx-proxy está corriendo
docker ps | grep nginx-proxy

# Verificar logs del proxy
docker logs nginx-proxy

# Verificar que VIRTUAL_HOST está configurado
docker inspect clientes-monolito-docker-app | grep VIRTUAL_HOST
```

### Contenedores no se reinician automáticamente

```bash
# Verificar restart policy
docker inspect clientes-monolito-docker-app --format='{{.HostConfig.RestartPolicy}}'

# Debería mostrar: {always 0}
```

---

## 📈 Escalabilidad Futura

Si necesita escalar:

```yaml
# Agregar a docker-compose.prod.yml
deploy:
  replicas: 2  # Múltiples instancias
```

---

## 🔐 Seguridad

- ✅ Usuario no-root (appuser)
- ✅ Red aislada
- ✅ Health checks automáticos
- ✅ Límites de recursos
- ✅ Reinicio automático

**Recomendaciones adicionales:**
- Cambiar contraseña default de MariaDB en producción
- Usar secretos de Docker para credenciales sensibles
- Configurar logs centralizados
- Implementar monitoreo con Prometheus/Grafana

---

## 📞 Contacto y Soporte

En caso de problemas, revisar:
1. [README.md](README.md) - Guía general
2. [ENTENDIENDO-DOCKER.md](ENTENDIENDO-DOCKER.md) - Explicación técnica
3. [ESTRUCTURA-ENTORNOS.md](ESTRUCTURA-ENTORNOS.md) - Estructura de entornos

---

**Última actualización:** 6 de febrero de 2026
