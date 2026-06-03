# Project Handover Report: Ecosistema Omnisalud Digital

**Date:** 2026-06-03  
**Phase:** Multi-app SSO Integration Complete — Production-Ready  
**Scope:** infra, Portal-Corporativo-Omnisalud, Desarrollo_Asistencial (app1), Desarrollo_Comercial (app2), Negociaciones_control_costo_v1 (app3)

---

## Executive Summary

### Ecosystem Purpose
The Omnisalud Digital platform is a suite of internal web applications for **Omnisalud** (Colombian healthcare organization). All apps share a centralized Single Sign-On system and are deployed behind a single Traefik reverse proxy with path-based routing on one domain.

### Apps Overview

| App | Path | Tech Stack | Purpose |
|-----|------|------------|---------|
| Portal | `/` | FastAPI + React/Vite | SSO authentication, user management, app launcher |
| App1 | `/app1` | Flask + Jinja2 | Medical schedule management (turnos/horarios) |
| App2 | `/app2` | Flask + React sub-app | Commercial operations (quotes, shifts, attendance) |
| App3 | `/app3` | Express + React/Vite | Cost control & provider negotiations |

### Current Phase
All four apps are functional, fully containerized, and integrated with shared SSO. The infrastructure is deployable via a single `docker compose up -d --build` command from `/home/sa/infra/`.

---

## Architecture Overview

### Request Flow
```
Browser → Traefik (:80/:443)
  ├─ /api/v1/*   → portal-auth:8000  (FastAPI auth server)
  ├─ /app1/*     → app1-asistencial:5000  (Flask, prefix stripped)
  ├─ /app2/*     → app2-comercial:8000  (Flask + Gunicorn, prefix stripped)
  ├─ /app3/*     → app3-costos:3001  (Express, prefix stripped)
  └─ /*          → portal-frontend:80  (React/Nginx SPA, lowest priority)
```

### Network
All services on shared bridge `omni_net`. Inter-service communication via container names. Only Traefik ports (80, 443, 8080) exposed to host.

### SSO Architecture
```
                    ┌──────────────────────────────┐
                    │  portal-auth (FastAPI :8000)  │
                    │  POST /api/v1/auth/login      │
                    │    → validates credentials     │
                    │    → sets sso_access_token     │
                    │      (HttpOnly, SameSite=Lax)  │
                    │  GET /api/v1/auth/logout       │
                    └──────────────────────────────┘
                                  │
                    JWT Cookie (HS256, shared secret)
                                  │
          ┌───────────────────────┼───────────────────────┐
          ▼                       ▼                       ▼
    app1 (Flask)            app2 (Flask)            app3 (Express)
    before_request      request_loader          ssoAuthMiddleware
    decode JWT          decode JWT              decode JWT (jose)
    set g.user          load User object        set req.user
```

All apps validate the same `sso_access_token` cookie using the shared `JWT_SECRET_KEY`. The JWT contains `sub`, `usuario`, `nombre_completo`, `email`, `documento`, `perfiles`, `roles`, `perfil_activo`, `rol_activo`.

---

## 1. infra/ — Orchestration Layer

### Files
| File | Purpose |
|------|---------|
| `docker-compose.yml` | Orchestrates 6 services + optional ngrok |
| `.env` | Shared credentials (JWT secret, MySQL, Let's Encrypt) |
| `.env.example` | Template with placeholders |
| `DESPLIEGUE.md` | Full deployment guide (447 lines) |
| `MANUAL_TECNICO.md` | Technical manual with architecture diagrams (425 lines) |
| `MANUAL_USUARIO.md` | User manual (215 lines) |
| `traefik/traefik.yml` | Reference static Traefik config |

### Services (docker-compose.yml)

| Service | Container | Image/Build | Internal Port | Traefik Label |
|---------|-----------|-------------|---------------|---------------|
| `traefik` | traefik | `traefik:latest` | 8080 | Dashboard |
| `portal-auth` | portal-auth | `../Portal-Corporativo-Omnisalud/auth-server` | 8000 | `PathPrefix(/api/v1)` |
| `portal-frontend` | portal-frontend | `../Portal-Corporativo-Omnisalud/client-app/frontend` | 80 | `PathPrefix(/)` priority=1 |
| `app1-asistencial` | app1-asistencial | `../Desarrollo_Asistencial_Version_1.0` | 5000 | `PathPrefix(/app1)` strip |
| `app2-comercial` | app2-comercial | `../Desarrollo_Comercial` | 8000 | `PathPrefix(/app2)` strip |
| `app3-costos` | app3-costos | `../Negociaciones_control_costo_v1` | 3001 | `PathPrefix(/app3)` strip |
| `ngrok` (optional) | ngrok | `ngrok/ngrok:latest` | 4040 | `--profile ngrok` |

### Shared Environment Variables (infra/.env)
```
JWT_SECRET_KEY=una-clave-secreta-muy-dificil-de-adivinar
JWT_ALGORITHM=HS256
MYSQL_HOST=192.168.2.187
MYSQL_PASSWORD=omnisalud.desarrollo
COOKIE_DOMAIN= (empty — same-domain path routing)
COOKIE_SECURE=false
LETSENCRYPT_CA=https://acme-staging-v02.api.letsencrypt.org/directory  (staging)
```

### Deployment
```bash
cd /home/sa/infra
docker compose up -d --build              # full stack
docker compose up -d --build app3-costos  # single app rebuild
docker compose --profile ngrok up -d      # with public tunnel
```

---

## 2. Portal-Corporativo-Omnisalud/ — SSO Identity Provider

### Auth Server (`auth-server/`)
| Aspect | Detail |
|--------|--------|
| Framework | FastAPI 0.115 |
| Database | `omn_core_global` via SQLAlchemy 2.0 + mysqlclient |
| Auth Libs | `python-jose[cryptography]` (JWT), `passlib[bcrypt]` (passwords) |
| Container | `python:3.11-slim`, uvicorn on :8000 |
| Routes | `/api/v1/auth/login`, `/refresh`, `/session`, `/logout`; `/api/v1/users/*` (CRUD) |

**Core files:**
- `app/core/config.py` — Pydantic Settings from env; `database_url`, `cors_origins`, JWT TTLs
- `app/core/security.py` — `bcrypt` hashing (12 rounds), `create_token()`/`decode_token()` with `python-jose`
- `app/api/v1/endpoints/auth.py` — Login (sets cookies), refresh (rotates tokens), session, logout
- `app/api/v1/endpoints/users.py` — Admin-only user CRUD, profile/role listing, medico management
- `app/models/user.py` — `core_usuarios` with M:N to profiles (`core_usuarios_perfiles`) and roles (`core_usuarios_roles`)
- `app/services/auth_service.py` — `authenticate()`, `create_token_pair()`, cookie management

**JWT Claims:**
```json
{
  "sub": "user_id",
  "type": "access",
  "exp": 1234567890,
  "iat": 1234567000,
  "usuario": "jdoe",
  "nombre_completo": "John Doe",
  "email": "john@omnisalud.co",
  "documento": "12345678",
  "perfiles": ["ADMIN"],
  "roles": ["COORDINADOR"],
  "perfil_activo": "ADMIN",
  "rol_activo": "COORDINADOR"
}
```

**Tokens:** Access (5d TTL), Refresh (15d TTL). Cookies: `sso_access_token`, `sso_refresh_token`.

**Key Endpoints:**
| Method | Path | Auth | Purpose |
|--------|------|------|---------|
| `POST` | `/api/v1/auth/login` | None | Authenticate + set cookies, return token pair |
| `POST` | `/api/v1/auth/refresh` | Refresh cookie | Rotate tokens |
| `GET` | `/api/v1/auth/session` | Access cookie | Return current user + profiles/roles |
| `GET/POST` | `/api/v1/auth/logout` | None | Clear cookies, redirect |
| `POST` | `/api/v1/users/register` | Admin | Register new user |
| `GET` | `/api/v1/users` | Admin | List all users |
| `PUT` | `/api/v1/users/{id}` | Admin | Update user |

### Frontend (`client-app/frontend/`)
| Aspect | Detail |
|--------|--------|
| Framework | React 18 + Vite 6 |
| Router | react-router-dom v6 |
| Icons | lucide-react |
| Container | Multi-stage: Node 20 build → Nginx 1.27 serve |
| Routes | `/login`, `/app`, `/app/users/*`, `/app/feature/:slug` |

**Core components:**
- `AuthGuard.jsx` — Validates session on protected routes, redirects to `/login?redirect=<path>` on failure
- `GuestGuard.jsx` — Redirects to `/app` if already authenticated
- `LoginPage.jsx` — Reads `?redirect=` param, normalizes it (handles nested encoding, external URLs, loop prevention), redirects after login
- `AppShell.jsx` — Loads session, shows user info, admin dropdown, logout

---

## 3. Desarrollo_Asistencial_Version_1.0/ — Medical Schedule Management (app1)

| Aspect | Detail |
|--------|--------|
| Framework | Flask 3.1 + Jinja2 templates |
| Auth Libs | `python-jose[cryptography]` (JWT), `bcrypt` (password) |
| Database | `omn_asis_gestion_turnos` (business) + `omn_core_global` (auth fallback) |
| Container | `python:3.11-slim`, Gunicorn on :5000 |

### SSO Implementation
- `app/core/auth_middleware.py` — `before_request` hook `load_user_from_jwt()`: reads `sso_access_token` cookie, decodes JWT, sets `g.user` and `g.authenticated`
- `app/core/jwt_auth.py` — `decode_token()`, `extract_user_from_payload()` — extracts `sub`, `usuario`, `nombre_completo`, `email`, `documento`, `perfiles`, `roles`
- `app/auth/routes.py` — When `SSO_ENABLED=true`, `/login` immediately redirects to `external_login_url()` → `/login` (portal)
- Fallback mode (`SSO_ENABLED=false`): bcrypt password check against `omn_core_global.core_usuarios`

### Key Routes
| Path | Auth | Purpose |
|------|------|---------|
| `/` | Required | Redirect to `/menu` |
| `/menu` | Required | Main dashboard |
| `/horarios/*` | Required | Schedule management (individual/team views) |
| `/api/*` | Required | JSON API endpoints |
| `/login` | Public | SSO redirect or local login form |
| `/logout` | Public | Clear session or SSO logout redirect |

### Profiles
- `Asistencial` — View own schedule
- `Administrativo2` — View/manage team schedules
- `Superadmin` — Full access including DELETE

---

## 4. Desarrollo_Comercial/ — Commercial Operations (app2)

| Aspect | Detail |
|--------|--------|
| Framework | Flask 3.1 + Flask-Login + React sub-app (cotizador) |
| Auth Libs | `python-jose[cryptography]` (JWT) |
| Database | `comercialdb` (business) |
| Container | Multi-stage: Node build → `python:3.11-slim`, Gunicorn on :8000 |

### SSO Implementation
- `app/auth/sso.py` — `validate_sso_token()`, `load_user_from_sso_payload()`, `get_sso_result()`, `external_login_url()`
- `app/auth/routes.py` — `Flask-Login` `request_loader` hooks into SSO; on `not_found` renders error page (prevents redirect loop)
- Login URL construction: `{MAIN_APP_URL}/login` — no params, same pattern as app1/app3
- Logout URL: `{MAIN_APP_URL}/api/v1/auth/logout?next={MAIN_APP_URL}/login`

### Key Routes
| Path | Auth | Purpose |
|------|------|---------|
| `/auth/login` | Public | SSO redirect or local login |
| `/auth/logout` | Required | Logout + SSO redirect |
| `/users/*` | Required | User management |
| `/asistencia/*` | Required | Attendance tracking |
| `/cotizador/*` | Required | React sub-app (quotations) |
| `/api/*` | Required | JSON API |

### Security Issues
- **Local login uses plaintext passwords** against `comercialdb.usuarios.contrasena` — no hashing
- `.env` contains commented-out production DB credentials

---

## 5. Negociaciones_control_costo_v1/ — Cost Control & Negotiations (app3)

| Aspect | Detail |
|--------|--------|
| Framework | Express 4.21 + React 19/Vite 8 |
| Auth Libs | `jose` (JWT decode), `bcryptjs` (password verify) |
| Database | `omn_ng_control_costos_rn` (business) + `omn_core_global` (auth fallback) |
| Container | Multi-stage: Node 22 build → Node 22 runner on :3001 |

### SSO Implementation (implemented this session)
- `src/server/auth/ssoMiddleware.js` — Express middleware: reads `sso_access_token` cookie, decodes JWT via `jose`, sets `req.user`. Silent on failure (calls `next()`)
- `src/server/auth/requireAuth.js` — When SSO enabled, trusts `req.user` set by middleware. When SSO disabled, validates Bearer HMAC-SHA256 token
- `src/server/controllers/authController.js` — `sessionHandler()` returns user from cookie; `loginHandler()` in SSO mode returns `{ sso: true, loginUrl: '/login' }`
- `src/components/LoginScreen.jsx` — On mount: calls `checkSSOSession()`, if no session → `window.location.replace('/login')`. No manual button needed
- `src/adapters/httpClient.js` — All `fetch()` calls use `credentials: 'include'` when SSO enabled
- `src/config/env.js` (client) — `apiBaseUrl` (empty in dev, `/app3` in Docker) ensures API calls route through Traefik

### Local Auth Fallback (implemented this session)
- `src/server/repositories/userRepository.js` — Separate MySQL pool to `omn_core_global`. `authenticate(username, password)` — bcrypt verify against `core_usuarios.contrasena_hash`, checks `estado = 'activo'`
- `src/server/services/authService.js` — Signs HMAC-SHA256 token with user claims

### Business Features (pre-existing)
- **5 Tabs:** Resultados (P&L), Consultar (search), Costos (negotiate), Regional (dispersion), Resumen (history)
- Cost negotiation with full audit trail (`cc_costo_historial`)
- IPC management (yearly percentages)
- CIF (indirect costs) configuration
- Regional cost dispersion analysis
- Excel export (SheetJS CDN)
- City & service catalog CRUD with caching (5-min TTL)

### API Endpoints
| Method | Path | Auth | Purpose |
|--------|------|------|---------|
| `POST` | `/api/auth/login` | None | SSO redirect or DB/local auth |
| `GET` | `/api/auth/verify` | Token/SSO | Verify current session |
| `GET` | `/api/auth/session` | SSO cookie | Return user from cookie |
| `GET` | `/api/auth/sso-logout` | None | Return portal logout URL |
| `GET/POST` | `/api/ciudades` | Required | City catalog CRUD |
| `GET` | `/api/servicios` | Required | Service catalog |
| `GET` | `/api/costos/catalog` | Required | Cost catalog with ref data |
| `POST` | `/api/costos/negociar` | Required | Execute negotiation (transactional) |
| `GET/POST` | `/api/costos/configuracion` | Required | App configuration |
| `POST/DELETE` | `/api/costos/ipc` | Required | IPC management |
| `GET` | `/api/costos/ahorro/*` | Required | Savings analysis |
| `GET` | `/api/costos/historial/*` | Required | Cost history audit |
| `GET/POST` | `/api/costos/dispersion/*` | Required | Regional dispersion |

### Key Design Decisions (app3)
| Decision | Rationale |
|----------|-----------|
| `jose` for JWT decode (not `jsonwebtoken`) | Matches `python-jose` behavior; Web Crypto native, no native deps |
| `bcryptjs` not `bcrypt` | Pure JS — works on alpine without native compilation |
| Separate MySQL pool for auth DB | Isolates auth queries from business queries |
| `VITE_API_BASE_URL=/app3` in Docker | Required for Traefik to route API calls to app3 (not portal-frontend) |
| `NODE_ENV=production` gates static serving | Prevents Express from serving stale `dist/` in dev mode |
| No `?redirect=` param on login URL | Matches app1/app2; avoids SPA navigation issues in portal |
| `credentials: 'include'` on all fetches | Required for cookie-based SSO |

---

## Database Map

### omn_core_global (shared across all apps)
| Table | Used By | Purpose |
|-------|---------|---------|
| `core_usuarios` | Portal, app1, app3 | User accounts (bcrypt hashed passwords) |
| `core_perfiles` | Portal | Profile definitions |
| `core_roles` | Portal | Role definitions |
| `core_usuarios_perfiles` | Portal | User-to-profile assignments (M:N) |
| `core_usuarios_roles` | Portal | User-to-role assignments (M:N) |
| `core_ciudades_nacional` | app3 | City catalog with providers |
| `core_servicios_nacional` | app3 | Service catalog with pricing |
| `core_servicios_sedes_propias` | app3 | Service name/code reference |
| `core_sgm_operaciones_precios_staging` | app3 | Sales data for savings calculation |
| `core_medicos` | app1 | Medical professionals |
| `core_sedes_propias` | app1 | Locations by region |

### omn_asis_gestion_turnos (app1)
| Table | Purpose |
|-------|---------|
| `gt_agenda_v2` | Schedule data (EAV model) |
| `gt_field_catalog` | Schedule field definitions |
| `gt_usuarios` | Legacy user table |

### comercialdb (app2)
| Table | Purpose |
|-------|---------|
| `usuarios` | App users (plaintext passwords ⚠️) |
| Business tables | Quotes, shifts, attendance |

### omn_ng_control_costos_rn (app3)
| Table | Purpose |
|-------|---------|
| `cc_configuraciones` | Key-value config (CIF, REGIONALES, OBJETIVOS) |
| `cc_ipc_historial` | Yearly IPC percentages |
| `cc_negociaciones` | Negotiation records |
| `cc_costo_historial` | Full audit trail of cost changes |
| `cc_dispersion_historico` | Pre-calculated dispersion (auto-created) |

---

## Technical Debt

### Cross-Project
| Issue | Severity | Projects |
|-------|----------|----------|
| JWT secret in plain `.env` files | **High** | All 4 |
| DB credentials in `.env` files | **High** | All 4 |
| Let's Encrypt staging CA (not production) | **High** | infra |
| `COOKIE_SECURE=false` in dev | **Medium** | All |
| No automated tests | **Medium** | app1, app2, app3 |
| No refresh token handling in app1/app3 | **Medium** | app1, app3 |

### app1 (Desarrollo_Asistencial)
| Issue | Severity |
|-------|----------|
| Raw SQL queries throughout (no ORM) | Medium |
| EAV data model (`gt_agenda_v2`) — complex queries | Medium |
| Legacy `gt_usuarios` table still referenced | Low |

### app2 (Desarrollo_Comercial)
| Issue | Severity |
|-------|----------|
| **Plaintext passwords** in `comercialdb.usuarios` | **Critical** |
| Commented-out production credentials in `.env` | High |
| Two codebases in one repo (Flask + React sub-app) | Medium |

### app3 (Negociaciones_control_costo_v1)
| Issue | Severity |
|-------|----------|
| Monolithic component `control-costos.jsx` (~2500 lines) | High |
| `APP_PASSWORD` fallback in auth controller | Medium |
| In-memory caches (no Redis) | Medium |
| `NVIDIA_API_KEY` exposed in `.env` | Low |
| No role-based access within app (all users get full access) | Low |

---

## Risks

### Security
| Risk | Severity | Mitigation |
|------|----------|------------|
| `JWT_SECRET_KEY` shared in plain env files | **High** | Use Docker secrets or vault in production |
| `MYSQL_PASSWORD` in plain env files | **High** | Same as above |
| Plaintext passwords in app2 local DB | **Critical** | Hash with bcrypt, migrate existing passwords |
| No CSRF protection on API | Medium | SSO uses `SameSite=Lax`; non-SSO apps rely on Bearer tokens |
| Let's Encrypt staging CA | Medium | Remove `caServer` from Traefik config for production |
| `NVIDIA_API_KEY` in app3 `.env` | Low | Move to secrets, verify if still needed |

### Performance
| Risk | Impact | Mitigation |
|------|--------|-----------|
| In-memory caches lost on restart | Cache miss storm | Acceptable at current scale; add Redis for HA |
| Single MySQL host for all apps | Noisy neighbor | Monitor; consider read replicas for analytics |
| Large component re-renders in app3 | Slow UI | Memoization already used; consider code splitting |

### Maintenance
| Risk | Impact | Mitigation |
|------|--------|-----------|
| 4 different tech stacks (FastAPI, Flask, Express, React) | High cognitive load | Document patterns; consider gradual convergence |
| No CI/CD pipeline | Manual deploys | Add GitHub Actions for build + push |
| Auth logic duplicated across apps | Inconsistent behavior | Extract to shared auth library (future) |

---

## Recommended Next Steps

### Critical
1. **Hash app2 passwords** — `comercialdb.usuarios.contrasena` is plaintext. Migrate to bcrypt matching app1/app3.
2. **Production Let's Encrypt** — Remove `caServer` from Traefik, set `COOKIE_SECURE=true`
3. **Rotate JWT secret** — `una-clave-secreta-muy-dificil-de-adivinar` is shared everywhere
4. **Add app3 card to portal dashboard** — `Portal-Corporativo-Omnisalud/client-app/frontend/src/modules/home/pages/HomePage.jsx`

### Important
5. **Implement refresh token in app3** — Currently redirects to login on expiry; app2 already handles this
6. **Add Playwright E2E tests** — Playwright is already in app3's devDependencies
7. **Clean `.env` files** — Remove commented-out production credentials, move API keys to secrets
8. **Add health check endpoints** — Portal already has `/health`; add to app1, app2, app3 (app3 has implicit DB check on startup)
9. **Container resource limits** — Add `mem_limit` / `cpu_limit` to docker-compose services

### Future Improvements
10. **Refactor app3 `control-costos.jsx`** — Split into per-tab components (~500 lines each)
11. **Add Redis caching** — Replace in-memory caches in app3 for multi-instance support
12. **API rate limiting** — Add to Express and Flask apps
13. **Extract shared auth library** — Common JWT validation code used by all 3 apps
14. **CI/CD pipeline** — GitHub Actions: build → test → push images → deploy
15. **DB connection monitoring** — Add pool metrics export for all apps

---

## Quick Reference: Deploy Commands

```bash
# Full stack
cd /home/sa/infra
docker compose up -d --build

# Single app rebuild
docker compose up -d --build app3-costos

# With public tunnel (testing)
docker compose --profile ngrok up -d
curl -s http://localhost:4040/api/tunnels | grep -o 'https://[^"]*'

# View logs
docker compose logs -f app3-costos

# Stop
docker compose down
```

## Quick Reference: Local Dev

```bash
# app3 (Negociaciones)
cd /home/sa/Negociaciones_control_costo_v1
SSO_ENABLED=false VITE_SSO_ENABLED=false npm run dev:all
# → Frontend: http://localhost:5173
# → API: http://localhost:3001

# app1 (Asistencial)
cd /home/sa/Desarrollo_Asistencial_Version_1.0
python run.py
# → http://localhost:5000

# app2 (Comercial)
cd /home/sa/Desarrollo_Comercial
python run.py
# → http://localhost:8000

# Portal
# Auth server: cd auth-server && uvicorn app.main:app --port 8000
# Frontend: cd client-app/frontend && npm run dev
```

---

## Directory Map

```
/home/sa/
├── infra/                                    # Docker orchestration (run from here)
│   ├── docker-compose.yml
│   ├── .env / .env.example
│   ├── DESPLIEGUE.md
│   ├── MANUAL_TECNICO.md
│   ├── MANUAL_USUARIO.md
│   └── traefik/
├── Portal-Corporativo-Omnisalud/             # SSO Identity Provider
│   ├── auth-server/                          # FastAPI backend
│   │   ├── app/main.py
│   │   ├── app/core/{config,security}.py
│   │   ├── app/api/v1/endpoints/{auth,users}.py
│   │   ├── app/models/{user,profile,role}.py
│   │   └── Dockerfile
│   └── client-app/frontend/                  # React SPA
│       ├── src/App.jsx
│       ├── src/components/{AuthGuard,GuestGuard}.jsx
│       ├── src/modules/auth/pages/LoginPage.jsx
│       └── Dockerfile
├── Desarrollo_Asistencial_Version_1.0/       # app1 — Medical Schedules
│   ├── app/
│   │   ├── core/{auth_middleware,jwt_auth,config}.py
│   │   ├── auth/routes.py
│   │   └── modules/{usuarios,horarios}/
│   └── Dockerfile
├── Desarrollo_Comercial/                     # app2 — Commercial Operations
│   ├── app/
│   │   ├── auth/{sso,routes}.py
│   │   ├── cotizador_comercial_omnisalud_2026/  # React sub-app
│   │   └── config.py
│   └── Dockerfile
└── Negociaciones_control_costo_v1/           # app3 — Cost Control
    ├── server.js                             # Entry point
    ├── src/
    │   ├── server/
    │   │   ├── app.js                        # Express setup
    │   │   ├── auth/{ssoMiddleware,requireAuth,tokenUtils}.js
    │   │   ├── controllers/authController.js
    │   │   ├── services/authService.js
    │   │   ├── repositories/userRepository.js
    │   │   └── config/{env,database}.js
    │   ├── control-costos.jsx                # Main React component
    │   ├── components/LoginScreen.jsx
    │   ├── adapters/httpClient.js
    │   ├── services/authService.js
    │   └── config/env.js
    ├── Dockerfile
    └── vite.config.js
```
