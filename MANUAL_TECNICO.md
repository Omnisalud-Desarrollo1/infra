# MANUAL TÉCNICO — Ecosistema Omnisalud Digital

## Tabla de Contenido

1. [Arquitectura General](#1-arquitectura-general)
2. [Single Sign-On (SSO)](#2-single-sign-on-sso)
3. [Ruteo y Reverse Proxy (Traefik)](#3-ruteo-y-reverse-proxy-traefik)
4. [Aplicaciones del Ecosistema](#4-aplicaciones-del-ecosistema)
5. [Variables de Entorno](#5-variables-de-entorno)
6. [Estructura de Archivos](#6-estructura-de-archivos)
7. [Bases de Datos](#7-bases-de-datos)
8. [Docker e Infraestructura](#8-docker-e-infraestructura)
9. [Seguridad](#9-seguridad)
10. [Mantenimiento y Troubleshooting](#10-mantenimiento-y-troubleshooting)

---

## 1. Arquitectura General

```
┌──────────────────────────────────────────────────────┐
│                 portalcorporativo.omnisalud.co        │
│                     (Traefik :80/:443)                │
│                                                       │
│  ┌───────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │       /*       │  │   /api/v1/*  │  │   /app1/*  │ │
│  │  portal-       │  │  portal-     │  │  app1-     │ │
│  │  frontend      │  │  auth        │  │  asistencial│ │
│  │  (React/Nginx) │  │  (FastAPI)   │  │  (Flask)   │ │
│  │  puerto 80     │  │  puerto 8000 │  │  puerto 5000│ │
│  └───────────────┘  └──────────────┘  └────────────┘ │
│                                                       │
│  ┌──────────────┐  ┌──────────────────────────────┐  │
│  │   /app2/*    │  │         ngrok (testing)       │  │
│  │  app2-       │  │    túnel HTTPS → Traefik:80  │  │
│  │  comercial   │  └──────────────────────────────┘  │
│  │  (Flask)     │                                     │
│  │  puerto 8000 │                                     │
│  └──────────────┘                                     │
└──────────────────────────────────────────────────────┘
```

### Componentes

| Componente | Tecnología | Puerto | Repositorio |
|-----------|-----------|--------|-------------|
| **Traefik** | v3.x (latest) | 80, 443, 8080 | `infra/` |
| **portal-auth** | Python FastAPI + SQLAlchemy | 8000 | `Portal-Corporativo-Omnisalud/auth-server/` |
| **portal-frontend** | React 18 + Vite 6 → Nginx | 80 | `Portal-Corporativo-Omnisalud/client-app/frontend/` |
| **app1-asistencial** | Python Flask 3.1 + Jinja2 | 5000 | `Desarrollo_Asistencial_Version_1.0/` |
| **app2-comercial** | Python Flask + Flask-Login + Gunicorn | 8000 | `Desarrollo_Comercial/` |

---

## 2. Single Sign-On (SSO)

### Flujo de Autenticación

```
┌──────────┐  (1) login    ┌───────────────┐  (2) JWT + cookie  ┌──────────────┐
│  Usuario  │ ──────────── │  portal-auth   │ ──────────────── │  Navegador    │
│          │              │  (FastAPI)     │                  │  (HttpOnly)   │
└──────────┘              └───────────────┘                  └──────┬───────┘
                                                                    │
                              (3) GET /app1/menu + cookie           │
                                                                    │
                              ┌─────────────────────────────────────┘
                              ▼
                    ┌─────────────────┐
                    │  app1/app2       │
                    │  (Flask)         │
                    │                  │
                    │  before_request  │
                    │  → lee cookie    │
                    │  → decodifica JWT│
                    │  → setea g.user  │
                    └─────────────────┘
```

### Detalles del JWT

| Campo | Valor | Descripción |
|-------|-------|-------------|
| Algoritmo | HS256 | Simétrico (clave compartida) |
| Clave secreta | `JWT_SECRET_KEY` | Mismo valor en todas las apps |
| Cookie | `sso_access_token` | `HttpOnly; SameSite=Lax; Path=/` |
| Claims | `sub, type, exp, iat, usuario, nombre_completo, email, documento, perfiles, roles, perfil_activo, rol_activo` | Emitidos por portal-auth |

### Validación en apps secundarias

**app1-asistencial** (`app/core/auth_middleware.py`):
```python
# before_request hook en Flask
def load_user_from_jwt():
    token = request.cookies.get('sso_access_token')
    if token:
        payload = decode_token(token)         # python-jose
        if payload['type'] == 'access' and not is_token_expired(payload):
            g.user = extract_user_from_payload(payload)
            g.authenticated = True
```

**app2-comercial** (`app/auth/sso.py` + `app/auth/routes.py`):
```python
# Flask-Login request_loader
@login_manager.request_loader
def load_user_from_request(req):
    token = request.cookies.get('sso_access_token')
    payload = validate_sso_token(token)       # python-jose
    user = load_user_from_sso_payload(payload) # busca en comercialDB.usuarios
    return user
```

### Modo Fallback (SSO_ENABLED=false)

Ambas apps secundarias soportan login local cuando `SSO_ENABLED=false`:
- **app1-asistencial**: autentica contra `omn_core_global.core_usuarios` usando bcrypt
- **app2-comercial**: autentica contra `comercialDB.usuarios` con contraseña en texto plano

---

## 3. Ruteo y Reverse Proxy (Traefik)

### Reglas de Ruteo

| Ruta | Servicio | Middleware | Prioridad |
|------|----------|-----------|-----------|
| `/api/v1/*` | portal-auth:8000 | — | 21 |
| `/app1/*` | app1-asistencial:5000 | strip `/app1` | 19 |
| `/app2/*` | app2-comercial:8000 | strip `/app2` | 19 |
| `/*` | portal-frontend:80 | — | 1 |

### Strip Prefix

Traefik remueve el prefijo antes de enviar la petición al backend:
```
Cliente → GET /app1/menu
Traefik → strip /app1 → GET /menu → app1-asistencial:5000
```

Las apps Flask usan `APPLICATION_ROOT` para generar URLs con el prefijo correcto:
```python
app.config['APPLICATION_ROOT'] = '/app1'  # Flask genera /app1/menu en url_for()
```

### HTTPS (Let's Encrypt)

Configurado en `infra/docker-compose.yml`:
- Entrypoint `web` en puerto 80
- Entrypoint `websecure` en puerto 443
- Certificate resolver `letsencrypt` con HTTP challenge
- CA staging para pruebas; remover `caServer` para producción

---

## 4. Aplicaciones del Ecosistema

### 4.1 Portal-Corporativo-Omnisalud (portal-auth + portal-frontend)

| Aspecto | Detalle |
|---------|---------|
| Propósito | Autenticación centralizada SSO, gestión de usuarios, perfiles y roles |
| Backend | FastAPI + SQLAlchemy + MySQL |
| Frontend | React 18 + Vite 6 + React Router DOM |
| DB | `omn_core_global` (MySQL) |
| Tablas | `core_usuarios`, `core_perfiles`, `core_roles`, `core_usuarios_perfiles`, `core_usuarios_roles` |
| API | `/api/v1/auth/{login,logout,session,refresh}`, `/api/v1/users/*`, `/api/v1/health` |
| Dockerfile | `auth-server/Dockerfile` + `client-app/frontend/Dockerfile` |

### 4.2 Desarrollo_Asistencial (app1)

| Aspecto | Detalle |
|---------|---------|
| Propósito | Gestión de horarios, agenda y profesionales asistenciales |
| Framework | Flask 3.1 + Jinja2 (server-rendered) |
| DB Negocio | `omn_asis_gestion_turnos` (MySQL) |
| DB Auth | `omn_core_global` (para fallback local login) |
| Módulos | auth, main, api, gestion_horarios, usuarios |
| Auth | Middleware `before_request` + JWT (modo SSO) o sesión Flask (fallback) |
| Dockerfile | `Dockerfile` (python:3.11-slim, puerto 5000) |

### 4.3 Desarrollo_Comercial (app2)

| Aspecto | Detalle |
|---------|---------|
| Propósito | Cotizador, turnos comerciales, registro de asistencia |
| Framework | Flask + Flask-Login + Gunicorn |
| DB | `comercialDB` (MySQL) |
| Módulos | auth, main, users, api, turnos, cotizador, registro_asistencia |
| Auth | Flask-Login `request_loader` + JWT (modo SSO) o login local (fallback) |
| Sub-app | React 19 + Vite 7 (cotizador) servido desde Flask via manifest |
| Dockerfile | Multi-stage: Node build + Python 3.11-slim, puerto 8000 |

---

## 5. Variables de Entorno

### Variables Compartidas (Todas las Apps)

| Variable | Descripción | Ejemplo |
|----------|-------------|---------|
| `JWT_SECRET_KEY` | Clave secreta para firmar/verificar JWT | `una-clave-secreta-muy-dificil-de-adivinar` |
| `JWT_ALGORITHM` | Algoritmo JWT | `HS256` |
| `COOKIE_ACCESS_TOKEN_NAME` | Nombre de la cookie SSO | `sso_access_token` |
| `MAIN_APP_URL` | URL base del Portal (relativa) | `/` |
| `APP_PATH_PREFIX` | Prefijo de ruta | `/app1` o `/app2` |
| `COOKIE_DOMAIN` | Dominio de la cookie (vacío = host actual) | _vacío_ |
| `COOKIE_SECURE` | Cookie solo en HTTPS | `true` (prod) / `false` (dev) |

### Variables de infraestructura (archivo `infra/.env`)

| Variable | Descripción |
|----------|-------------|
| `MYSQL_HOST` | Host del servidor MySQL |
| `MYSQL_PORT` | Puerto MySQL (3306) |
| `MYSQL_USER` | Usuario MySQL |
| `MYSQL_PASSWORD` | Contraseña MySQL |
| `CORS_ORIGINS` | Orígenes permitidos CORS |
| `FRONTEND_URL` | URL del frontend (vacío = relativo) |
| `LETSENCRYPT_EMAIL` | Email para Let's Encrypt |
| `LETSENCRYPT_CA` | CA server (staging/producción) |
| `APP1_SECRET_KEY` | Flask SECRET_KEY para app1 |
| `APP2_SECRET_KEY` | Flask SECRET_KEY para app2 |
| `APP2_DB_NAME` | Nombre BD para app2 (comercialDB) |
| `NGROK_AUTHTOKEN` | Token ngrok (solo testing) |

### Variables específicas por app

Ver `.env.example` en cada repositorio:
- `Portal-Corporativo-Omnisalud/auth-server/.env.example`
- `Desarrollo_Asistencial_Version_1.0/.env.example`
- `Desarrollo_Comercial/.env.example`

---

## 6. Estructura de Archivos

```
/home/sa/
├── infra/                              # Orquestación central
│   ├── docker-compose.yml              # Stack completo
│   ├── .env                            # Variables de entorno (credenciales)
│   ├── .env.example                    # Plantilla sin secretos
│   └── traefik/
│       └── traefik.yml                 # Configuración estática (doc)
│
├── Portal-Corporativo-Omnisalud/       # App principal (SSO)
│   ├── auth-server/                    # Backend FastAPI
│   │   ├── Dockerfile
│   │   ├── .env / .env.example
│   │   ├── requirements.txt
│   │   └── app/
│   │       ├── main.py                 # FastAPI app factory
│   │       ├── core/                   # config, security, middleware
│   │       ├── api/                    # routers, deps, endpoints
│   │       ├── models/                 # SQLAlchemy models
│   │       ├── services/               # auth_service
│   │       └── db/                     # session, init_db
│   └── client-app/frontend/            # Frontend React
│       ├── Dockerfile
│       ├── nginx.conf
│       └── src/
│           ├── services/authService.js # API calls al auth-server
│           ├── components/             # FeatureCard, AuthGuard...
│           └── modules/                # home, auth, admin
│
├── Desarrollo_Asistencial_Version_1.0/ # App secundaria 1
│   ├── Dockerfile
│   ├── .env / .env.example
│   ├── requirements.txt
│   ├── run.py                          # Entry point
│   └── app/
│       ├── __init__.py                 # Flask app factory
│       ├── core/
│       │   ├── config.py               # Configuración
│       │   ├── database.py             # MySQL connector
│       │   ├── jwt_auth.py             # Verificación JWT
│       │   ├── auth_db.py              # Conexión global_auth
│       │   └── auth_middleware.py       # before_request JWT
│       ├── modules/
│       │   ├── usuarios/               # services, repository
│       │   └── horarios/               # services, repository
│       ├── api/routes.py               # API endpoints
│       ├── auth/routes.py              # Login/logout (SSO + fallback)
│       ├── main/routes.py              # Páginas principales
│       └── templates/                  # Jinja2 templates
│
└── Desarrollo_Comercial/               # App secundaria 2
    ├── Dockerfile                      # Multi-stage (Node + Python)
    ├── .dockerignore
    ├── .env / .env.example
    ├── requirements.txt
    ├── config.py                       # Configuración
    ├── run.py                          # Entry point
    ├── app/
    │   ├── __init__.py                 # Flask app factory + Flask-Login
    │   ├── models.py                   # User model (UserMixin)
    │   ├── db.py                       # MySQL connection
    │   ├── auth/
    │   │   ├── routes.py               # Login/logout (SSO + fallback)
    │   │   └── sso.py                  # JWT validation, helpers
    │   ├── main/routes.py              # / y /health
    │   ├── cotizador/routes.py         # 1378 líneas, API cotizador
    │   ├── api/routes.py               # API usuarios + turnos
    │   ├── users/routes.py             # CRUD usuarios
    │   ├── turnos/routes.py            # Gestión de turnos
    │   └── templates/                  # Jinja2 templates
    └── cotizador_comercial_omnisalud_2026/  # React sub-app
        ├── package.json
        ├── vite.config.js
        └── src/
```

---

## 7. Bases de Datos

| Base de Datos | Servidor | Usada por | Propósito |
|--------------|----------|-----------|-----------|
| `omn_core_global` | `192.168.2.187` (dev) | portal-auth, app1 (auth) | Usuarios, perfiles, roles, autenticación central |
| `omn_asis_gestion_turnos` | `192.168.2.187` (dev) | app1-asistencial | Horarios, agenda, profesionales médicos |
| `comercialDB` | `192.168.2.187` (dev) | app2-comercial | Usuarios locales, turnos, cotizaciones, asistencia |

### Tablas principales en `omn_core_global`

| Tabla | Descripción |
|-------|-------------|
| `core_usuarios` | Usuarios del sistema (id, usuario, contrasena_hash, email, nombre_completo, documento, estado) |
| `core_perfiles` | Perfiles/áreas (nombre, descripcion) |
| `core_roles` | Roles (nombre, descripcion, id_perfil FK) |
| `core_usuarios_perfiles` | Relación M:N Usuario-Perfil (activo, asignado_por, fecha_asignacion) |
| `core_usuarios_roles` | Relación M:N Usuario-Rol (activo, asignado_por, fecha_asignacion) |

---

## 8. Docker e Infraestructura

### Comandos principales

```bash
cd /home/sa/infra

# Iniciar stack completo
docker compose up -d

# Reconstruir una app específica
docker compose up -d --build portal-frontend

# Ver logs
docker compose logs -f app1-asistencial

# Detener todo
docker compose down

# Iniciar con ngrok (Docker)
docker compose --profile ngrok up -d
```

### Health checks

Cada app expone un endpoint de salud:

| App | Endpoint | Respuesta |
|-----|----------|-----------|
| portal-auth | `GET /api/v1/health` | `{"status":"healthy"}` |
| app1-asistencial | `GET /api/v1/health` (vía portal-auth) | `{"status":"healthy"}` |
| app2-comercial | `GET /app2/health` | `{"status":"healthy"}` |

### Redes

Todos los servicios comparten la red `omni_net` (bridge). Traefik se comunica con los backends a través de esta red interna.

---

## 9. Seguridad

### JWT y Cookies

- **Algoritmo**: HS256 (simétrico, clave compartida)
- **Cookie**: `HttpOnly` (no accesible vía JavaScript), `SameSite=Lax`, `Secure` en producción
- **Expiración**: Access token 7200 min (5 días), Refresh token 21600 min (15 días)
- **Rotación**: El frontend intenta refresh automático al recibir 401

### Recomendaciones de Producción

1. **Rotar `JWT_SECRET_KEY`**: Generar clave aleatoria de 256 bits
2. **`COOKIE_SECURE=true`**: Solo transmitir cookies sobre HTTPS
3. **Let's Encrypt producción**: Remover `caServer` staging del docker-compose
4. **No exponer puertos de backend**: Solo Traefik (80/443) expuesto al exterior
5. **Credenciales en `.env`**: Nunca commitear a git; usar `.env.example` con placeholders
6. **CORS restrictivo**: `CORS_ORIGINS=https://portalcorporativo.omnisalud.co`
7. **app2-comercial**: Migrar contraseñas de texto plano a bcrypt antes de producción

---

## 10. Mantenimiento y Troubleshooting

### Problemas comunes

| Problema | Causa probable | Solución |
|----------|---------------|----------|
| 404 en todas las rutas | Traefik no detecta contenedores | `docker compose up -d --force-recreate` |
| 404 solo en rutas con TLS | `tls.certresolver` en entrypoint HTTP | Usar solo `entrypoints=web` para HTTP |
| `/app2/menu` 404 | Ruta incorrecta | Comercial usa `/` no `/menu` |
| Cookie no se envía | `COOKIE_DOMAIN` mal configurado | Dejar vacío para mismo dominio |
| Token inválido | `JWT_SECRET_KEY` no coincide entre apps | Verificar mismo valor en todas las apps |
| Salud del contenedor | Healthcheck fallando | `docker logs app2-comercial` |
| ngrok no inicia | Túnel ya en uso | `pkill -f ngrok` y reintentar |

### Logs

```bash
# Traefik
docker logs traefik --tail 50

# Auth server
docker logs portal-auth --tail 50

# App específica
docker logs app1-asistencial --tail 50
docker logs app2-comercial --tail 50

# Acceso Traefik (habilitado en docker-compose)
docker logs traefik | grep "GET"
```
