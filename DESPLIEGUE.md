# GUÍA DE DESPLIEGUE — Ecosistema Omnisalud Digital

## Tabla de Contenido

1. [Requisitos Previos](#1-requisitos-previos)
2. [Estructura de Directorios Requerida](#2-estructura-de-directorios-requerida)
3. [Despliegue Local (Desarrollo)](#3-despliegue-local-desarrollo)
4. [Despliegue con ngrok (Testing Remoto)](#4-despliegue-con-ngrok-testing-remoto)
5. [Despliegue en Producción](#5-despliegue-en-producción)
6. [Despliegue Individual por App](#6-despliegue-individual-por-app)
7. [Actualización de una App](#7-actualización-de-una-app)
8. [Backup y Restauración](#8-backup-y-restauración)

---

## 1. Requisitos Previos

### Software necesario

| Herramienta | Versión mínima | Verificación |
|------------|---------------|--------------|
| Docker | 29.x+ | `docker --version` |
| Docker Compose | 5.x+ (plugin) | `docker compose version` |
| Git | 2.x+ | `git --version` |
| ngrok (opcional) | 3.x+ | `ngrok version` |

### Recursos del servidor (producción)

| Recurso | Mínimo | Recomendado |
|---------|--------|-------------|
| CPU | 2 vCPUs | 4 vCPUs |
| RAM | 4 GB | 8 GB |
| Disco | 20 GB | 50 GB SSD |
| SO | Ubuntu 22.04+ / Debian 12+ | Ubuntu 24.04 LTS |

### Acceso a Base de Datos

Las aplicaciones requieren acceso a un servidor MySQL con las siguientes bases de datos:

| Base de Datos | Usada por |
|--------------|-----------|
| `omn_core_global` | portal-auth, app1 (auth fallback) |
| `omn_asis_gestion_turnos` | app1-asistencial |
| `comercialDB` | app2-comercial |

> El servidor MySQL puede ser externo o un contenedor en el mismo docker-compose.

---

## 2. Estructura de Directorios Requerida

Clone los repositorios como siblings en el mismo directorio padre:

```bash
mkdir -p /home/sa
cd /home/sa

# Clonar repositorios (o copiarlos si ya existen)
git clone <repo-portal> Portal-Corporativo-Omnisalud
git clone <repo-asistencial> Desarrollo_Asistencial_Version_1.0
git clone <repo-comercial> Desarrollo_Comercial

# El directorio infra/ se crea dentro de Desarrollo_Asistencial
# y contiene docker-compose.yml + traefik/
```

Estructura resultante:

```
/home/sa/
├── infra/                          # ← aquí se ejecuta docker compose
│   ├── docker-compose.yml
│   ├── .env
│   ├── .env.example
│   ├── traefik/
│   │   └── traefik.yml
│   ├── MANUAL_TECNICO.md
│   ├── MANUAL_USUARIO.md
│   └── DESPLIEGUE.md
├── Portal-Corporativo-Omnisalud/
│   ├── auth-server/
│   └── client-app/frontend/
├── Desarrollo_Asistencial_Version_1.0/
└── Desarrollo_Comercial/
```

---

## 3. Despliegue Local (Desarrollo)

### Paso 1: Configurar variables de entorno

```bash
cd /home/sa/infra
cp .env.example .env
```

Editar `.env` con los valores correctos:

```ini
# JWT
JWT_SECRET_KEY=una-clave-secreta-muy-dificil-de-adivinar

# MySQL
MYSQL_HOST=192.168.2.187
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=omnisalud.desarrollo

# Cookies (vacío para desarrollo/testing)
COOKIE_DOMAIN=
COOKIE_SECURE=false

# URLs (vacío = relativo)
FRONTEND_URL=
CORS_ORIGINS=http://localhost

# Let's Encrypt (staging para pruebas)
LETSENCRYPT_EMAIL=admin@omnisalud.co
LETSENCRYPT_CA=https://acme-staging-v02.api.letsencrypt.org/directory

# Apps
APP1_SECRET_KEY=app1-secret-key-change-me
APP2_SECRET_KEY=app2-secret-key-change-me
APP2_DB_NAME=comercialdb

# Ngrok (opcional)
NGROK_AUTHTOKEN=
```

### Paso 2: Construir e iniciar

```bash
cd /home/sa/infra
docker compose up -d --build
```

La primera ejecución descargará imágenes base, instalará dependencias y compilará el frontend React (3-5 minutos).

### Paso 3: Verificar

```bash
# Salud de los servicios
curl http://localhost:80/api/v1/health     # → {"status":"healthy"}
curl http://localhost:80/app1/menu          # → 302 (redirige a login)
curl http://localhost:80/app2/health        # → {"status":"healthy"}
curl http://localhost:80/                   # → HTML del portal

# Estado de los contenedores
docker compose ps

# Dashboard de Traefik
open http://localhost:8080
```

### Paso 4: Acceder

Abra `http://localhost:80` en su navegador. Inicie sesión con sus credenciales corporativas.

---

## 4. Despliegue con ngrok (Testing Remoto)

Útil para probar el sistema desde internet sin configuración de DNS.

### Opción A: ngrok CLI (recomendado)

```bash
# 1. Asegurar que el stack está corriendo
cd /home/sa/infra && docker compose up -d

# 2. Iniciar ngrok apuntando a Traefik
ngrok http http://localhost:80

# 3. ngrok mostrará la URL pública:
#    Forwarding  https://xxxx-xx-xxx.ngrok-free.app -> http://localhost:80

# 4. Abrir esa URL en el navegador
```

### Opción B: ngrok en Docker

```bash
# 1. Configurar token en .env
#    NGROK_AUTHTOKEN=tu-token-aqui

# 2. Iniciar con perfil ngrok
cd /home/sa/infra
docker compose --profile ngrok up -d

# 3. Obtener la URL
curl -s http://localhost:4040/api/tunnels | python3 -c "import sys,json; print(json.load(sys.stdin)['tunnels'][0]['public_url'])"
```

---

## 5. Despliegue en Producción

### Paso 1: Preparar el servidor

```bash
# Ubuntu 24.04 LTS
sudo apt update && sudo apt upgrade -y
sudo apt install -y docker.io git curl

# Agregar usuario al grupo docker
sudo usermod -aG docker $USER
newgrp docker

# Instalar Docker Compose plugin
sudo apt install -y docker-compose-v2
```

### Paso 2: Clonar repositorios

```bash
mkdir -p /opt/omnisalud
cd /opt/omnisalud
git clone <repo-portal> Portal-Corporativo-Omnisalud
git clone <repo-asistencial> Desarrollo_Asistencial_Version_1.0
git clone <repo-comercial> Desarrollo_Comercial
```

### Paso 3: Configurar DNS

Agregar registro A en el DNS:
```
portalcorporativo.omnisalud.co  →  <IP_DEL_SERVIDOR>
```

Verificar:
```bash
dig portalcorporativo.omnisalud.co +short
# Debe devolver la IP del servidor
```

### Paso 4: Configurar variables de producción

```bash
cp /opt/omnisalud/Desarrollo_Asistencial_Version_1.0/infra/.env.example \
   /opt/omnisalud/Desarrollo_Asistencial_Version_1.0/infra/.env
```

Editar `.env` para producción:

```ini
# JWT — ¡CAMBIAR por una clave segura generada!
JWT_SECRET_KEY=<clave-aleatoria-256-bits>

# MySQL (credenciales de producción)
MYSQL_HOST=<ip-servidor-mysql>
MYSQL_PORT=3306
MYSQL_USER=<usuario-produccion>
MYSQL_PASSWORD=<contraseña-produccion>

# Cookies — ¡SEGURO en producción!
COOKIE_SECURE=true

# CORS — dominio real
CORS_ORIGINS=https://portalcorporativo.omnisalud.co

# Let's Encrypt — producción (sin staging)
LETSENCRYPT_EMAIL=admin@omnisalud.co
# LETSENCRYPT_CA=  ← comentado o vacío para CA real

# Apps
APP1_SECRET_KEY=<clave-aleatoria-app1>
APP2_SECRET_KEY=<clave-aleatoria-app2>
APP2_DB_NAME=comercialDB
```

### Paso 5: Iniciar en producción

```bash
cd /opt/omnisalud/Desarrollo_Asistencial_Version_1.0/infra
docker compose up -d --build
```

### Paso 6: Verificar HTTPS

```bash
# El certificado se genera automáticamente (puede tardar 1-2 minutos)
curl https://portalcorporativo.omnisalud.co/api/v1/health
# → {"status":"healthy"}
```

### Paso 7: Configurar reinicio automático

Crear servicio systemd para docker compose:

```bash
sudo tee /etc/systemd/system/omnisalud.service << 'EOF'
[Unit]
Description=Omnisalud Digital Ecosystem
Requires=docker.service
After=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/opt/omnisalud/Desarrollo_Asistencial_Version_1.0/infra
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl enable omnisalud
sudo systemctl start omnisalud
```

---

## 6. Despliegue Individual por App

Cada app puede ejecutarse individualmente sin Docker para desarrollo:

### Portal-Corporativo (auth-server)

```bash
cd Portal-Corporativo-Omnisalud/auth-server
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env  # editar credenciales
python -m uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

### Portal-Corporativo (frontend)

```bash
cd Portal-Corporativo-Omnisalud/client-app/frontend
npm install
cp .env.example .env  # VITE_AUTH_SERVER_URL=http://localhost:8000
npm run dev           # → http://localhost:3000
```

### Desarrollo_Asistencial

```bash
cd Desarrollo_Asistencial_Version_1.0
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env  # editar credenciales
python run.py         # → http://localhost:5000
```

### Desarrollo_Comercial

```bash
cd Desarrollo_Comercial

# Backend Flask
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env  # editar credenciales
python run.py         # → http://localhost:5000

# Frontend Cotizador (opcional, en otra terminal)
cd cotizador_comercial_omnisalud_2026
npm install
npm run dev           # → http://localhost:5173
```

---

## 7. Actualización de una App

### Actualizar código y reconstruir

```bash
cd /home/sa/infra

# Opción 1: Actualizar una app específica
cd ../Portal-Corporativo-Omnisalud && git pull
cd /home/sa/infra
docker compose up -d --build portal-auth portal-frontend

# Opción 2: Actualizar app1
cd ../Desarrollo_Asistencial_Version_1.0 && git pull
cd /home/sa/infra
docker compose up -d --build app1-asistencial

# Opción 3: Actualizar app2
cd ../Desarrollo_Comercial && git pull
cd /home/sa/infra
docker compose up -d --build app2-comercial
```

### Sin tiempo de inactividad (rolling update)

```bash
docker compose up -d --build --scale app1-asistencial=2 app1-asistencial
# Esperar a que el nuevo contenedor esté healthy
docker compose up -d --scale app1-asistencial=1 app1-asistencial
```

---

## 8. Backup y Restauración

### Base de datos

```bash
# Backup de todas las bases
mysqldump -h <host> -u <user> -p omn_core_global > backup_global_$(date +%Y%m%d).sql
mysqldump -h <host> -u <user> -p omn_asis_gestion_turnos > backup_asistencial_$(date +%Y%m%d).sql
mysqldump -h <host> -u <user> -p comercialDB > backup_comercial_$(date +%Y%m%d).sql

# Restauración
mysql -h <host> -u <user> -p omn_core_global < backup_global_20260101.sql
```

### Archivos de configuración

Backup mínimo recomendado:
- `/home/sa/infra/.env`
- `/home/sa/infra/docker-compose.yml`
- `/home/sa/infra/traefik/`

### Volúmenes Docker

```bash
# Backup de certificados Let's Encrypt
tar -czf letsencrypt_backup.tar.gz /home/sa/infra/traefik/letsencrypt/
```

---

## Verificación Post-Despliegue

Ejecutar después de cualquier despliegue:

```bash
# 1. Verificar salud de todos los contenedores
docker compose ps

# 2. Verificar endpoints
curl -f http://localhost:80/api/v1/health || echo "FAIL: auth API"
curl -f -o /dev/null http://localhost:80/ || echo "FAIL: frontend"
curl -f -o /dev/null http://localhost:80/app1/menu || echo "FAIL: app1"
curl -f http://localhost:80/app2/health || echo "FAIL: app2"

# 3. Verificar logs sin errores
docker compose logs --tail 20 | grep -i "error\|fail\|exception"
```
