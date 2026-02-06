# 📚 Guía Completa: Despliegue en Servidor de Producción

> **Objetivo**: Documentar paso a paso cómo desplegar la aplicación `clientes-monolito-docker` en el servidor de producción usando Docker Compose.

---

## 📋 Tabla de Contenidos

1. [Requisitos Previos](#requisitos-previos)
2. [Paso 1: Copiar el Proyecto vía FTP/SCP](#paso-1-copiar-el-proyecto)
3. [Paso 2: Preparar el Servidor](#paso-2-preparar-el-servidor)
4. [Paso 3: Construir la Imagen](#paso-3-construir-la-imagen)
5. [Paso 4: Levantar los Servicios](#paso-4-levantar-los-servicios)
6. [Paso 5: Verificaciones Post-Despliegue](#paso-5-verificaciones)
7. [Acceso a la Aplicación](#acceso-a-la-aplicación)
8. [Troubleshooting](#troubleshooting)

---

## 🔍 Requisitos Previos

Antes de comenzar, asegúrate de que:

### En el Servidor de Despliegues

- ✅ **Docker** instalado (versión 20.10+)
- ✅ **Docker Compose** v2+ instalado
- ✅ **MariaDB** compatible (versión 10.6)
- ✅ **nginx-proxy** corriendo (para reverse proxy)
- ✅ Red `nginx-proxy` creada en Docker
- ✅ Carpeta de destino disponible (ej: `/home/docker/proyectos/`)
- ✅ Acceso SSH configurado

### En tu Máquina Local

- ✅ Acceso SSH al servidor
- ✅ `scp` o cliente FTP disponible
- ✅ Rama `main` con todos los cambios commiteados

---

## 📤 Paso 1: Copiar el Proyecto

### Opción A: Usando SCP (Recomendado - Más Seguro)

```bash
# Desde tu máquina local
scp -r /home/sulbaranjc/proyectos/python/App_con_docker/clientes-monolito-docker \
  usuario@tu-servidor:/home/docker/proyectos/

# Ejemplo con IP:
scp -r /home/sulbaranjc/proyectos/python/App_con_docker/clientes-monolito-docker \
  usuario@192.168.1.100:/home/docker/proyectos/
```

### Opción B: Usando FTP Tradicional

```bash
# Abre cliente FTP
ftp tu-servidor

# Navegación en FTP
cd /home/docker/proyectos/
put -r clientes-monolito-docker
```

### Verificación Post-Copia

```bash
# Una vez copiado, verifica que todos los archivos estén
ls -la /home/docker/proyectos/clientes-monolito-docker/

# Deberías ver:
# drwxr-xr-x  app/
# -rw-r--r--  Dockerfile.dev
# -rw-r--r--  Dockerfile.prod
# -rw-r--r--  docker-compose.dev.yml
# -rw-r--r--  docker-compose.prod.yml
# -rw-r--r--  requirements.txt
# -rw-r--r--  init_db.sql
# ... (más archivos)
```

---

## 🔧 Paso 2: Preparar el Servidor

### 2.1 Conectarse al Servidor

```bash
ssh usuario@tu-servidor
# O con IP específica:
ssh usuario@192.168.1.100
```

### 2.2 Navegar a la Carpeta del Proyecto

```bash
cd /home/docker/proyectos/clientes-monolito-docker
pwd  # Verifica que estés en el lugar correcto
```

### 2.3 Crear la Red de nginx-proxy (Si no existe)

```bash
# Verificar si la red ya existe
docker network ls | grep nginx-proxy

# Si NO aparece, crear la red:
docker network create nginx-proxy

# Verificar que se creó:
docker network inspect nginx-proxy
```

> **¿Por qué?** La red `nginx-proxy` es externa y compartida por todos los proyectos en el servidor. Permite que el reverse proxy comunique con todos los contenedores.

### 2.4 Asignar Permisos Adecuados

```bash
# Permitir lectura y ejecución en la carpeta del proyecto
chmod 755 /home/docker/proyectos/clientes-monolito-docker

# Permitir lectura en archivos
chmod 644 /home/docker/proyectos/clientes-monolito-docker/*

# Permitir ejecución en carpetas
chmod 755 /home/docker/proyectos/clientes-monolito-docker/app
chmod 755 /home/docker/proyectos/clientes-monolito-docker/app/templates
```

---

## 🏗️ Paso 3: Construir la Imagen de Producción

### 3.1 Construir la Imagen Docker

```bash
# Desde la carpeta del proyecto
docker compose -f docker-compose.prod.yml build

# Salida esperada:
# [+] Building 45.3s (12/12) FINISHED
# => [internal] load build definition from Dockerfile.prod
# => [stage-0 1/7] FROM python:3.12-slim
# ...
```

> **¿Qué hace?** Lee el archivo `Dockerfile.prod`, descarga la imagen base de Python 3.12, instala dependencias de `requirements.txt` y empaqueta la aplicación en una imagen Docker lista para producción.

### 3.2 Verificar que la Imagen se Creó

```bash
docker images | grep clientes-monolito-docker
```

Deberías ver algo como:
```
clientes-monolito-docker-app   latest   a1b2c3d4e5f6   2 minutes ago   250MB
```

---

## 🚀 Paso 4: Levantar los Servicios

### 4.1 Crear y Arrancar Contenedores

```bash
# Desde la carpeta del proyecto
docker compose -f docker-compose.prod.yml up -d

# Salida esperada:
# [+] Running 2/2
#  ✔ Network clientes-monolito-docker-network  Created
#  ✔ Container clientes-monolito-docker-db     Started
#  ✔ Container clientes-monolito-docker-app    Started
```

> **¿Qué hace?** 
> - `-d`: Ejecuta en modo "detached" (background)
> - Crea la red privada del proyecto
> - Crea el volumen para la BD
> - Inicia el contenedor MariaDB
> - Inicia el contenedor de la aplicación FastAPI

### 4.2 Esperar a que los Servicios Estén Listos

```bash
# La BD puede tardar 10-15 segundos en inicializarse
# Espera un poco y luego verifica:

sleep 15
docker compose -f docker-compose.prod.yml ps
```

Salida esperada:
```
NAME                              COMMAND                  SERVICE      STATUS           PORTS
clientes-monolito-docker-db       docker-entrypoint.sh     db          Up 12 seconds    3306/tcp
clientes-monolito-docker-app      uvicorn app.main:app    app         Up 10 seconds    8000/tcp
```

---

## ✅ Paso 5: Verificaciones Post-Despliegue

### 5.1 Verificar que los Contenedores Estén Activos

```bash
docker ps | grep clientes-monolito-docker
```

Deberías ver 2 contenedores:
- `clientes-monolito-docker-db`
- `clientes-monolito-docker-app`

### 5.2 Verificar los Logs de la Aplicación

```bash
# Ver logs en tiempo real
docker compose -f docker-compose.prod.yml logs -f app

# Salida esperada:
# INFO:     Uvicorn running on http://0.0.0.0:8000
# INFO:     Application startup complete
```

**Presiona `CTRL+C` para salir.**

### 5.3 Verificar los Logs de la Base de Datos

```bash
docker compose -f docker-compose.prod.yml logs db

# Salida esperada:
# ... ready for connections
```

### 5.4 Verificar la Conexión a la Base de Datos

```bash
# Acceder a la BD y listar las tablas
docker exec clientes-monolito-docker-db \
  mysql -u clientes -pclientes123 \
  -e "USE clientes_db; SHOW TABLES;"

# Salida esperada (si init_db.sql existe):
# +--------------------+
# | Tables_in_clientes_db |
# +--------------------+
# | clientes           |
# | ... (otras tablas) |
# +--------------------+
```

### 5.5 Probar la Aplicación Localmente (en el servidor)

```bash
# Desde el servidor, prueba conectar a la API
curl http://localhost:8000

# Deberías obtener una respuesta HTML (la página principal)
```

### 5.6 Verificar Recursos del Contenedor

```bash
# Ver uso de CPU y memoria en tiempo real
docker stats clientes-monolito-docker-app

# Presiona CTRL+C para salir
```

Salida esperada:
```
CONTAINER ID    NAME                           CPU %   MEM USAGE
a1b2c3d4e5f6    clientes-monolito-docker-app  0.5%    120MB
```

---

## 🌐 Acceso a la Aplicación

### Desde el Servidor (Local)

```bash
# Acceso directo a la aplicación por puerto
curl http://localhost:8000

# Ver respuesta HTML
curl -s http://localhost:8000 | head -20
```

### Desde otra Máquina en la Red (por IP)

```bash
# Usando la IP del servidor (ejemplo: 192.168.1.100)
curl http://192.168.1.100:8000

# En navegador:
# http://192.168.1.100:8000
```

### A través del Dominio (Recomendado - nginx-proxy)

```bash
# Desde cualquier lugar con acceso al dominio:
curl http://clientes-monolito-docker.docker.sulbaranjc.com/

# En navegador:
# http://clientes-monolito-docker.docker.sulbaranjc.com/
```

> **Importante**: Esto requiere que:
> 1. nginx-proxy esté corriendo
> 2. El registro DNS apunte al servidor
> 3. La variable `VIRTUAL_HOST: clientes-monolito-docker.docker.sulbaranjc.com` esté configurada en `docker-compose.prod.yml`

---

## ✅ Verificación Completa Post-Despliegue

### Paso 1: Verificar que nginx-proxy esté Corriendo

```bash
# Verificar que nginx-proxy existe
docker ps | grep nginx-proxy

# Si no aparece, debe estar en otro servidor o contenedor
# Consulta con tu equipo de DevOps
```

### Paso 2: Verificar la Red nginx-proxy

```bash
# Inspeccionar la red compartida
docker network inspect nginx-proxy

# Deberías ver en "Containers":
# "clientes-monolito-docker-app": {...}
```

### Paso 3: Probar Conectividad Local (en el servidor)

```bash
# Test 1: Directamente por localhost
curl -i http://localhost:8000

# Salida esperada:
# HTTP/1.1 200 OK
# Content-Type: text/html; charset=utf-8
# ...
```

### Paso 4: Probar desde otra Máquina (por IP)

```bash
# Desde tu máquina local o administrador
# Reemplaza 192.168.1.100 con la IP real del servidor

curl -i http://192.168.1.100:8000

# Salida esperada:
# HTTP/1.1 200 OK
```

### Paso 5: Probar a través del Dominio

```bash
# Desde cualquier lugar con acceso al dominio
curl -i http://clientes-monolito-docker.docker.sulbaranjc.com/

# Salida esperada:
# HTTP/1.1 200 OK
# Server: nginx/...
# Content-Type: text/html; charset=utf-8
```

### Paso 6: Verificar que nginx-proxy Reconoce el Contenedor

```bash
# Ver logs de nginx-proxy
docker logs nginx-proxy | grep clientes-monolito-docker

# Deberías ver algo como:
# ... clientes-monolito-docker.docker.sulbaranjc.com ...
# ... address assigned ...
```

### Paso 7: Verificar la Configuración de VIRTUAL_HOST

```bash
# Ver las variables de entorno del contenedor app
docker inspect clientes-monolito-docker-app | grep VIRTUAL_HOST

# Salida esperada:
# "VIRTUAL_HOST=clientes-monolito-docker.docker.sulbaranjc.com",
```

---

## 🔍 Tabla de Verificación por Acceso

| Tipo de Acceso | URL | Dónde Verificar | Estado Esperado |
|---|---|---|---|
| **Local (servidor)** | `http://localhost:8000` | Terminal del servidor | HTTP 200 ✅ |
| **Red Interna (IP)** | `http://192.168.1.100:8000` | Otra máquina en red | HTTP 200 ✅ |
| **Dominio (nginx)** | `http://clientes-monolito-docker.docker.sulbaranjc.com/` | Navegador/curl | HTTP 200 ✅ |
| **Logs nginx-proxy** | `docker logs nginx-proxy` | Terminal servidor | Vea el dominio configurado |
| **Red conectada** | `docker network inspect nginx-proxy` | Terminal servidor | App en la red |

---

## 🚨 Si No Funciona el Dominio

### Checklist de Diagnóstico

```bash
# 1. ¿nginx-proxy está corriendo?
docker ps | grep nginx-proxy

# 2. ¿La app está en la red nginx-proxy?
docker inspect clientes-monolito-docker-app | grep -A 5 "Networks"

# 3. ¿VIRTUAL_HOST está correcto?
docker inspect clientes-monolito-docker-app | grep VIRTUAL_HOST

# 4. ¿DNS apunta al servidor?
nslookup clientes-monolito-docker.docker.sulbaranjc.com

# 5. ¿El firewall permite acceso?
sudo ufw status
sudo ufw allow 80
sudo ufw allow 443
```

### Soluciones Comunes

```bash
# Si nginx-proxy no reconoce el contenedor
docker compose -f docker-compose.prod.yml restart app

# Si DNS no resuelve
# Contacta a tu administrador de DNS/infraestructura

# Si firewall bloquea
sudo ufw allow from any to any port 80 proto tcp
sudo ufw allow from any to any port 443 proto tcp
```

---

## 🐛 Troubleshooting

### Problema 1: Los Contenedores se Detienen Inmediatamente

```bash
# Ver los logs de error
docker compose -f docker-compose.prod.yml logs app

# Causas comunes:
# - El archivo init_db.sql no existe
# - requirements.txt tiene dependencias incompatibles
# - Puerto 8000 ya está en uso
```

**Solución**:
```bash
# Verificar que init_db.sql existe
ls -la init_db.sql

# Reiniciar el contenedor con logs
docker compose -f docker-compose.prod.yml up app
```

### Problema 2: Error de Conexión a la Base de Datos

```bash
# Verificar que el contenedor de BD está corriendo
docker compose -f docker-compose.prod.yml ps db

# Ver logs de la BD
docker compose -f docker-compose.prod.yml logs db

# Verificar credenciales
docker exec clientes-monolito-docker-db \
  mysql -u clientes -pclientes123 -e "SELECT 1;"
```

**Solución**:
```bash
# Recrear los contenedores desde cero
docker compose -f docker-compose.prod.yml down -v
docker compose -f docker-compose.prod.yml up -d
```

### Problema 3: No Puedo Acceder a la Aplicación

```bash
# Verificar que el contenedor está escuchando en el puerto 8000
docker exec clientes-monolito-docker-app netstat -tlnp | grep 8000

# Prueba desde dentro del contenedor
docker exec clientes-monolito-docker-app curl http://localhost:8000

# Verifica el firewall del servidor
sudo ufw status
sudo ufw allow 8000
```

### Problema 4: El Dominio No Funciona

```bash
# Verificar que nginx-proxy está corriendo
docker ps | grep nginx-proxy

# Inspeccionar la red
docker network inspect nginx-proxy

# Ver logs de nginx-proxy
docker logs nginx-proxy
```

**Solución**: Asegúrate de que la variable `VIRTUAL_HOST` está correcta en `docker-compose.prod.yml`.

### Problema 5: Permisos Denegados

```bash
# Error: "permission denied"

# Verificar permisos actuales
ls -la /home/docker/proyectos/clientes-monolito-docker/

# Asignar permisos correctos
sudo chown -R tu-usuario:tu-usuario /home/docker/proyectos/clientes-monolito-docker/
chmod -R 755 /home/docker/proyectos/clientes-monolito-docker/
```

---

## 📋 Checklist Post-Despliegue

Antes de dar por finalizado el despliegue, verifica:

- [ ] Los 2 contenedores están activos (`docker ps`)
- [ ] Los logs no muestran errores (`docker compose ... logs`)
- [ ] La BD inicializó correctamente
- [ ] Puedes acceder a `http://localhost:8000`
- [ ] El dominio resuelve correctamente
- [ ] Los permisos de carpetas están correctos
- [ ] El firewall permite el tráfico en puerto 8000
- [ ] Hay espacio en disco para los datos de la BD
- [ ] Los backups de BD están configurados

---

## 🔄 Comandos Útiles para Mantenimiento

### Ver Estado General

```bash
# Estado de todos los servicios
docker compose -f docker-compose.prod.yml ps

# Uso de recursos
docker stats
```

### Gestión de Logs

```bash
# Últimas 100 líneas de logs
docker compose -f docker-compose.prod.yml logs --tail=100

# Logs en tiempo real
docker compose -f docker-compose.prod.yml logs -f

# Logs de hace 1 hora
docker compose -f docker-compose.prod.yml logs --since 1h
```

### Detener y Reiniciar

```bash
# Detener sin eliminar datos
docker compose -f docker-compose.prod.yml down

# Reiniciar servicios
docker compose -f docker-compose.prod.yml restart

# Reiniciar solo la aplicación
docker compose -f docker-compose.prod.yml restart app
```

### Backup de Base de Datos

```bash
# Crear backup de la BD
docker exec clientes-monolito-docker-db \
  mysqldump -u clientes -pclientes123 clientes_db \
  > backup_clientes_$(date +%Y%m%d_%H%M%S).sql

# Restaurar desde backup
docker exec -i clientes-monolito-docker-db \
  mysql -u clientes -pclientes123 clientes_db \
  < backup_clientes_20260206_143022.sql
```

---

## 📞 Soporte

Si encuentras problemas:

1. **Revisa los logs**: `docker compose ... logs`
2. **Verifica los requisitos**: Todos los archivos necesarios deben existir
3. **Consulta el troubleshooting**: Sección anterior
4. **Contacta al equipo de DevOps**: @sulbaranjc

---

**Versión**: 1.0  
**Última actualización**: 6 de febrero de 2026  
**Autor**: @sulbaranjc  
**Estado**: ✅ Producción
