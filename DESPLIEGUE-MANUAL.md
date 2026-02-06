# 🚀 Despliegue Manual - Guía Simplificada

> **Objetivo**: Aprender a desplegar cambios manualmente desde `dev` a `main` sin automatización.
> Esta es la forma más sencilla para entender todo el proceso.

---

## 📋 Tabla de Contenidos

1. [Flujo General](#flujo-general)
2. [Paso 1: Cambios en DEV (tu máquina)](#paso-1-cambios-en-dev)
3. [Paso 2: Preparar DEPLOY](#paso-2-preparar-deploy)
4. [Paso 3: Desplegar a MAIN](#paso-3-desplegar-a-main)
5. [Paso 4: Desplegar en Servidor](#paso-4-desplegar-en-servidor)
6. [Paso 5: Verificar](#paso-5-verificar)
7. [Comandos de Referencia](#comandos-de-referencia)

---

## 🔄 Flujo General

```
Tu máquina (LOCAL)
│
├─ 1️⃣  Modificas código en app/
├─ git add + git commit en rama dev
├─ git push origin dev
│
├─ 2️⃣  git checkout deploy
├─ git merge dev (trae cambios)
├─ git push origin deploy
│
└─ 3️⃣  git checkout main
   git merge deploy (trae cambios)
   git push origin main

Servidor (REMOTE)
│
└─ 4️⃣  git pull origin main
   docker compose build (reconstruye imagen)
   docker compose down (detiene contenedores)
   docker compose up -d (levanta nuevos)
   curl localhost:8000 (verifica)
```

---

## � Las Mejores Prácticas: ¿Por qué 3 ramas?

### ¿Por qué "duplicamos" cambios si pasamos de dev a deploy Y de deploy a main?

Excelente pregunta. Esto parece redundante, pero es la **práctica profesional estándar** y te explico por qué:

#### 🚨 La realidad: `deploy` es tu **GATEKEEP** de producción

```
dev     = "Escribo código aquí (inestable, experimental)"
deploy  = "Pruebo cambios aquí ANTES de producción (testing)"
main    = "Solo cambios que PASARON el testing y van a PRODUCCIÓN"
```

#### ¿Por qué no desplegar directo desde deploy?

**Mala práctica:**
```bash
git pull origin deploy
docker compose up -d
# ⚠️  Cualquier cosa en deploy va directo a producción
# ¿Qué pasó en deploy? ¿Está testeado? ¿Es seguro?
# No se sabe.
```

**Buena práctica (profesional):**
```bash
git pull origin main
docker compose up -d
# ✅ Main SOLO tiene cambios que PASARON testing
# ✅ Main es INMUTABLE hasta que realmente funciona
# ✅ Auditoria clara: "¿qué versión está en producción?" → main
```

#### 📊 Comparación de flujos

| Aspecto | Malo: Desplegar de deploy | Bueno: Desplegar de main |
|---------|---------------------------|--------------------------|
| **¿Qué hay en main?** | Lo que alguien metió | Solo lo que funciona |
| **Testing** | Ninguno | Obligatorio en deploy |
| **Rollback** | Confuso, main desincronizado | Limpio: git reset main |
| **Auditoría** | "¿Qué versión está viva?" | "Mira main, eso está vivo" |
| **Seguridad** | Alto riesgo | Control total |

#### 🎯 La "redundancia" es en realidad **control de calidad**

```
Tu máquina:
  dev ──(merge)──> deploy [TESTING AQUÍ]
                      ↓
                   ¿OK?
                      ↓
                    SÍ → main ──(push)──> SERVIDOR
```

**Lo que pasa:**

1. **En dev:** Escribes código sin restricciones
2. **En deploy:** Haces merge y PRUEBAS. ¿Se rompió algo? No afecta main
3. **En main:** SOLO lo que pasó el testing. Esto es SAGRADO
4. **En servidor:** Siempre trae main. Siempre es la versión que sabes que funciona

#### 💡 Analógía del mundo real

Es como un medicamento:

```
Laboratorio (dev)     → Formulan medicina experimental
Pruebas Clínicas (deploy) → La prueban en voluntarios
Aprobación (main)     → "OK, esto es seguro"
Farmacia (servidor)   → Distribuyen lo aprobado
```

Si la farmacia toma medicinas de "pruebas clínicas" sin aprobación final, ¡catástrofe!

---

## �🎨 Paso 1: Cambios en DEV

Cuando haces modificaciones al código en tu máquina.

### 1.1 Asegúrate de estar en rama dev

```bash
git checkout dev
```

Salida esperada:
```
Ya está en 'dev'
Tu rama está actualizada con 'origin/dev'.
```

### 1.2 Trae cambios remotos (por si otros desarrolladores hicieron cambios)

```bash
git pull origin dev
```

### 1.3 Haz cambios a los archivos

Por ejemplo, edita un archivo:
```bash
# Ejemplo: modificar la vista
nano app/templates/index.html

# O cambiar lógica backend
nano app/main.py
```

### 1.4 Verifica los cambios que hiciste

```bash
git status
```

Salida esperada:
```
En la rama dev
Cambios no preparados para la confirmación:
	modificado:   app/main.py
	modificado:   app/templates/index.html
```

### 1.5 Agregar los cambios

```bash
git add .
```

O archivos específicos:
```bash
git add app/main.py app/templates/index.html
```

### 1.6 Hacer commit (guardar cambios locales)

```bash
git commit -m "feat: agregar nueva vista de reportes"
```

**Mensajes de commit recomendados:**
- `feat:` para nuevas características
- `fix:` para correcciones
- `docs:` para documentación
- `style:` para CSS/estilos

### 1.7 Enviar a GitHub

```bash
git push origin dev
```

Salida esperada:
```
Enumerando objetos: 5, listo.
Total 3 (delta 2)
To https://github.com/sulbaranjc/clientes-monolito-docker.git
   a1b2c3d..e5f6g7h  dev -> dev
```

---

## 🚀 Paso 2: Preparar DEPLOY (Testing)

**Importante:** En deploy NO DESPLIEGAS AÚN. Solo traes cambios para **PROBAR** antes de producción.

### 2.1 Cambiar a rama deploy

```bash
git checkout deploy
```

Salida esperada:
```
Cambiado a rama 'deploy'
Tu rama está actualizada con 'origin/deploy'.
```

### 2.2 Traer cambios de origin (actualizar)

```bash
git pull origin deploy
```

### 2.3 Traer los cambios de dev a deploy

```bash
git merge dev
```

Salida esperada:
```
Actualizando a1b2c3d..e5f6g7h
Fast-forward
 app/main.py | 10 ++++++++--
 1 file changed, 8 insertions(+), 2 deletions(-)
```

### 2.4 Ver qué cambios se van a desplegar

```bash
git log --oneline -5
```

Salida esperada:
```
e5f6g7h (HEAD -> deploy, origin/deploy) feat: agregar nueva vista de reportes
a1b2c3d docs: agregar reporte final de validación
9a3c2c9 fix: agregar red nginx-proxy
...
```

### 2.5 Enviar deploy a GitHub

```bash
git push origin deploy
```

✅ **Ahora tienes cambios listos en rama deploy**

---

## 📦 Paso 3: Desplegar a MAIN (Producción)

**Esto es importante:** Solo mergeas deploy a main CUANDO TODO ESTÁ TESTED Y FUNCIONA.
Main es SAGRADO. Main es PRODUCCIÓN.

### 3.1 Cambiar a rama main

```bash
git checkout main
```

Salida esperada:
```
Cambiado a rama 'main'
Tu rama está actualizada con 'origin/main'.
```

### 3.2 Traer cambios remotos (muy importante)

```bash
git pull origin main
```

### 3.3 Traer los cambios de deploy a main

```bash
git merge deploy
```

Salida esperada:
```
Actualizando a1b2c3d..e5f6g7h
Fast-forward
 app/main.py | 10 ++++++++--
 1 file changed, 8 insertions(+), 2 deletions(-)
```

### 3.4 Enviar main a GitHub

```bash
git push origin main
```

Salida esperada:
```
Total 3 (delta 2)
To https://github.com/sulbaranjc/clientes-monolito-docker.git
   a1b2c3d..e5f6g7h  main -> origin/main
```

✅ **Cambios están en GitHub main, ahora falta actualizar servidor**

---

## 🖥️ Paso 4: Desplegar en Servidor (desde MAIN)

**Regla de oro:** El servidor SIEMPRE trae cambios de `main`, NUNCA de `deploy`.

Esto garantiza que en servidor solo hay código que pasó testing.

### 4.1 Conectarse al servidor

```bash
ssh sulbaranjc@docker.sulbaranjc.com
```

### 4.2 Navegar a la carpeta del proyecto

```bash
cd ~/apps/clientes-monolito-docker
```

### 4.3 Traer los cambios desde GitHub

```bash
git pull origin main
```

Salida esperada:
```
remote: Enumerating objects: 5, done.
Actualizando a1b2c3d..e5f6g7h
Fast-forward
 app/main.py | 10 ++++++++--
 1 file changed, 8 insertions(+), 2 deletions(-)
```

### 4.4 Reconstruir la imagen Docker

```bash
docker compose -f docker-compose.prod.yml build
```

Salida esperada:
```
[+] Building 30.2s (12/12) FINISHED
 => [internal] load build definition from Dockerfile.prod
 => [stage-0 2/7] FROM python:3.12-slim
 ...
 => => naming to clientes-monolito-docker-app
```

### 4.5 Detener contenedores antiguos

```bash
docker compose -f docker-compose.prod.yml down
```

Salida esperada:
```
 Container clientes-monolito-docker-app Stopping
 Container clientes-monolito-docker-app Stopped
 Container clientes-monolito-docker-app Removing
 Container clientes-monolito-docker-db Stopping
 ...
```

### 4.6 Levantar nuevos contenedores con imagen actualizada

```bash
docker compose -f docker-compose.prod.yml up -d
```

Salida esperada:
```
 [+] Running 2/2
  ✔ Container clientes-monolito-docker-db   Started
  ✔ Container clientes-monolito-docker-app  Started
```

### 4.7 Esperar a que inicie (15 segundos)

```bash
sleep 15
```

---

## ✅ Paso 5: Verificar

Asegurar que todo funciona correctamente en el servidor.

### 5.1 Ver estado de contenedores

```bash
docker ps | grep clientes-monolito-docker
```

Salida esperada:
```
e5f6g7h   clientes-monolito-docker-app    "uvicorn app.main..." Up 20 seconds   8000/tcp
a1b2c3d   clientes-monolito-docker-db     "docker-entrypoint..." Up 30 seconds   3306/tcp
```

### 5.2 Ver logs para asegurar que no hay errores

```bash
docker compose -f docker-compose.prod.yml logs app | tail -15
```

Salida esperada:
```
INFO:     Uvicorn running on http://0.0.0.0:8000
INFO:     Application startup complete
```

### 5.3 Probar que responde localmente

```bash
curl -i http://localhost:8000
```

Salida esperada:
```
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
...
```

### 5.4 Probar vía dominio

```bash
curl -i http://clientes-monolito-docker.docker.sulbaranjc.com/
```

Salida esperada:
```
HTTP/1.1 200 OK
Server: nginx/1.29.4
Content-Type: text/html; charset=utf-8
...
```

### 5.5 Salir del servidor

```bash
exit
```

✅ **¡Despliegue completado exitosamente!**

---

## 📋 Comandos de Referencia

### En tu máquina (LOCAL)

```bash
# Ver rama actual
git branch

# Ver cambios no commiteados
git status

# Ver historial
git log --oneline -10

# Cambiar de rama
git checkout dev
git checkout deploy
git checkout main

# Actualizar rama
git pull origin <rama>

# Mergear cambios
git merge <otra-rama>

# Enviar cambios
git push origin <rama>

# Agregar cambios
git add .
git add archivo.py

# Commit
git commit -m "mensaje"
```

### En el servidor (REMOTO)

```bash
# Conectarse
ssh sulbaranjc@docker.sulbaranjc.com

# Navegar
cd ~/apps/clientes-monolito-docker

# Traer cambios
git pull origin main

# Reconstruir imagen
docker compose -f docker-compose.prod.yml build

# Parar contenedores
docker compose -f docker-compose.prod.yml down

# Iniciar contenedores
docker compose -f docker-compose.prod.yml up -d

# Ver estado
docker ps | grep clientes-monolito-docker

# Ver logs
docker compose -f docker-compose.prod.yml logs app

# Probar aplicación
curl -i http://localhost:8000
```

---

## 🎯 Ejemplo Completo (Paso a Paso)

> **Recuerda:** dev = escribir | deploy = testear | main = producción

### DÍA 1: Desarrollo en tu máquina

```bash
# 1. Estás en dev
git checkout dev
git pull origin dev

# 2. Modificas archivo
nano app/main.py

# 3. Verificas cambios
git status

# 4. Agregas y commites
git add .
git commit -m "feat: agregar búsqueda de clientes"

# 5. Envías a GitHub
git push origin dev
```

### DÍA 2: Preparar despliegue

```bash
# 1. Cambias a deploy
git checkout deploy
git pull origin deploy

# 2. Traes cambios de dev
git merge dev

# 3. Verificas
git log --oneline -3

# 4. Envías
git push origin deploy
```

### DÍA 3: Desplegar a producción

```bash
# 1. Cambias a main
git checkout main
git pull origin main

# 2. Traes cambios de deploy
git merge deploy

# 3. Envías
git push origin main

# ⏸️  AHORA VAS AL SERVIDOR
```

### DÍA 3: En el servidor

```bash
# 1. Conectas
ssh sulbaranjc@docker.sulbaranjc.com

# 2. Navegas
cd ~/apps/clientes-monolito-docker

# 3. Traes cambios
git pull origin main

# 4. Reconstruyes
docker compose -f docker-compose.prod.yml build

# 5. Paras
docker compose -f docker-compose.prod.yml down

# 6. Inicias
docker compose -f docker-compose.prod.yml up -d

# 7. Esperas
sleep 15

# 8. Verificas
docker ps
curl http://localhost:8000

# 9. Salís
exit
```

✅ **¡Cambios en vivo!**

---

## ❌ Si Algo Sale Mal (Rollback)

### Opción 1: Deshacer último commit (rápido)

```bash
# En tu máquina
git checkout main
git revert HEAD
git push origin main
```

Luego en el servidor:
```bash
git pull origin main
docker compose -f docker-compose.prod.yml build
docker compose -f docker-compose.prod.yml down
docker compose -f docker-compose.prod.yml up -d
```

### Opción 2: Volver a commit anterior

```bash
# Ver historial
git log --oneline -10

# Ir a un commit anterior (ejemplo: a1b2c3d)
git checkout main
git reset --hard a1b2c3d
git push origin main --force
```

Luego en el servidor:
```bash
git pull origin main
docker compose -f docker-compose.prod.yml build
docker compose -f docker-compose.prod.yml down
docker compose -f docker-compose.prod.yml up -d
```

---

## ✅ Checklist Pre-Despliegue

Antes de desplegar, verifica:

- [ ] Cambios commiteados en dev
- [ ] Sin conflictos en merge dev→deploy
- [ ] Sin conflictos en merge deploy→main
- [ ] Push a main completado
- [ ] En servidor: git pull exitoso
- [ ] Docker build sin errores
- [ ] Contenedores iniciaron correctamente
- [ ] `curl localhost:8000` retorna HTTP 200
- [ ] Dominio accesible

---

**Versión:** 1.0  
**Actualizado:** 6 de febrero de 2026  
**Modo:** Manual (sin automatización)
