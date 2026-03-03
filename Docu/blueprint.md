# Quiz App Infrastructure Blueprint v2

**Target:** first real deployment of the quiz MVP on a single Hetzner VPS  
**Audience:** Radu (infra) + Paul (app dev)  
**Status:** implementation blueprint, updated with concrete findings from app code review  
**Approach:** single-node, Docker Compose, Caddy, PostgreSQL, Redis, Ansible  
**Date:** 2025-03  
**Supersedes:** v1 blueprint + initial capacity estimation PDF

---

## 1. Executive summary

This document is the **final consolidated blueprint** for deploying the quiz app MVP.

It combines the original architecture plan with a detailed code review of the actual frontend and backend repos. Every decision, every env var, every hard-coded value has been traced through the source. The result is a document that is both strategic and immediately actionable.

The agreed direction remains:

- **one Hetzner VPS**
- **Caddy** as reverse proxy and static frontend server
- **Docker Compose** for runtime orchestration
- **Angular SPA** frontend (confirmed Angular 21.1)
- **NestJS API** backend (confirmed NestJS 11)
- **PostgreSQL** as source of truth (via TypeORM)
- **Redis** as cache only (via `@keyv/redis`)
- **Ansible** for host provisioning and deployment
- **GitHub Actions later**, only after manual and Ansible-based deployment works end to end

This is a **single-node vertical-scaling MVP architecture**. That remains the correct complexity level for the current workload.

---

## 2. Confirmed tech stack from code review

This is no longer a proposal. These are the exact dependencies found in both repos.

### 2.1 Frontend (Angular SPA)

| Dependency | Version | Role |
|---|---|---|
| `@angular/core` | ^21.1.0 | framework |
| `@angular/router` | ^21.1.0 | SPA routing |
| `@angular/forms` | ^21.1.0 | reactive forms |
| `jwt-decode` | ^4.0.0 | client-side token inspection |
| `rxjs` | ~7.8.0 | async/observable handling |

Frontend pages confirmed: `login`, `register`, `quiz`, `admin` (question management + category management).

Build system: `@angular/build` (not Webpack). Output: static browser assets (`/dist/medical-quiz/browser`).

### 2.2 Backend (NestJS API)

| Dependency | Version | Role |
|---|---|---|
| `@nestjs/core` | ^11.0.1 | framework |
| `@nestjs/typeorm` | ^11.0.0 | ORM integration |
| `typeorm` | ^0.3.28 | database ORM |
| `pg` | ^8.18.0 | PostgreSQL driver |
| `@keyv/redis` | ^5.1.6 | Redis cache store |
| `cache-manager` | ^7.2.8 | cache abstraction |
| `@nestjs/jwt` | ^11.0.2 | JWT signing/verification |
| `passport` + `passport-jwt` | ^0.7.0 / ^4.0.1 | auth strategy |
| `bcrypt` | ^6.0.0 | password hashing |
| `@nestjs/throttler` | ^6.5.0 | rate limiting |
| `class-validator` / `class-transformer` | latest | request validation |

Backend modules confirmed: `AppModule`, `UserModule`, `QuestionModule`, `CategoryModule`.

Auth system: JWT access tokens (15min) + refresh tokens (7d), bcrypt hashing, admin/roles/premium guards.

### 2.3 Data model (TypeORM entities)

```text
UserEntity
  - id (PK, auto)
  - email (unique)
  - password (hashed)
  - role (enum: user | admin)
  - stripeCustomerId (nullable)
  - hasPremiumAccess (boolean, default false)
  - stripePaymentId (nullable)
  - hashedRefreshToken (nullable)
  - createdAt, updatedAt

QuestionEntity
  - id (PK, auto)
  - question (string)
  - image (nullable)
  - explanation (nullable, text)
  - multiple (boolean, default true)
  - answers -> AnswerEntity[] (cascade, eager)
  - category -> CategoryEntity (eager, not null)

AnswerEntity
  - id (PK, auto)
  - text (string)
  - correct (boolean, default false)
  - question -> QuestionEntity (ManyToOne, CASCADE delete)

CategoryEntity
  - id (PK, auto)
  - name (unique)
  - description (nullable)
  - questions -> QuestionEntity[] (OneToMany)
  - createdAt, updatedAt
```

### 2.4 API endpoints confirmed

```text
GET    /                         → hello world (health-ish)
POST   /users/register           → register
POST   /users/login              → login
POST   /users/refresh            → refresh tokens
POST   /users/logout             → logout (JWT-guarded)
GET    /questions/daily?limit=N  → daily quiz (cached)
POST   /questions                → create question
PATCH  /questions/:id            → update question
GET    /questions/:id            → get single question
GET    /categories               → list categories
POST   /categories               → create category
```

### 2.5 Application behavior notes

- Rate limiting is global: 10 requests per 60 seconds via `ThrottlerGuard` as `APP_GUARD`. This will likely need tuning for production since daily quiz fetches would count against it.
- Daily quiz uses Redis cache with a key like `daily-quiz-YYYY-MM-DD-1`. TTL is currently hard-coded to 1000ms (dev value), should be set to midnight rollover.
- `trust proxy` is already enabled in `main.ts` (good for running behind Caddy).
- `ValidationPipe` is globally applied with `whitelist: true` and `forbidNonWhitelisted: true` (solid).
- Admin seed runs automatically on module init via `onModuleInit()` in `UserModule`.

---

## 3. Critical findings: hard-coded values that must be fixed

This is the most important section. These are concrete lines in the actual source code that will break or compromise a production deployment.

### 3.1 Backend hard-coded values

| File | What | Current value | Must become |
|---|---|---|---|
| `src/app.module.ts` | TypeORM host | `'localhost'` | `process.env.DB_HOST` |
| `src/app.module.ts` | TypeORM port | `5432` | `parseInt(process.env.DB_PORT)` |
| `src/app.module.ts` | TypeORM username | `"postgres"` | `process.env.DB_USER` |
| `src/app.module.ts` | TypeORM password | `"admin"` | `process.env.DB_PASSWORD` |
| `src/app.module.ts` | TypeORM database | `'quizdb'` | `process.env.DB_NAME` |
| `src/app.module.ts` | TypeORM synchronize | `true` | `process.env.NODE_ENV !== 'production'` |
| `src/app.module.ts` | Redis URL | cloud Redis Labs URL with credentials in source | `process.env.REDIS_URL` |
| `src/user/user.module.ts` | JWT secret | `"PULA"` | `process.env.JWT_SECRET` |
| `src/auth/jwt-strategy.ts` | JWT secretOrKey | `"PULA"` | `process.env.JWT_SECRET` |
| `src/user/user.service.ts` | Admin seed email | `"PULA@PULA.com"` | `process.env.ADMIN_EMAIL` |
| `src/user/user.service.ts` | Admin seed password | `"PULAPULA"` | `process.env.ADMIN_PASSWORD` |
| `src/main.ts` | CORS origin | `'http://localhost:4200'` | `process.env.CORS_ORIGIN` |
| `src/main.ts` | Listen port | `3000` | `parseInt(process.env.PORT) \|\| 3000` |

**Severity: critical.** The Redis Labs URL contains a real credential committed in source. The JWT secret is a joke string used for both signing and verification. The admin seed credentials are placeholder values. None of this can go to production as-is.

### 3.2 Frontend hard-coded values

| File | What | Current value | Must become |
|---|---|---|---|
| `src/app/services/admin.service.ts` | API base | `'http://localhost:3000'` | `environment.apiBaseUrl` |
| `src/app/services/auth.service.ts` | API URL | `'http://localhost:3000/users'` | `environment.apiBaseUrl + '/users'` |
| `src/app/services/quiz.service.ts` | API URL | `'http://localhost:3000/questions'` | `environment.apiBaseUrl + '/questions'` |

**Severity: critical.** In production behind Caddy with `/api` routing, these must all point to `/api` (relative) or the full production domain.

### 3.3 Security findings

| Finding | Severity | Location |
|---|---|---|
| Category controller admin guards are **commented out** | high | `src/category/category.controller.ts` lines 2494-2495 |
| Redis Labs credentials committed in source code | critical | `src/app.module.ts` |
| JWT secret is a placeholder string | critical | `src/user/user.module.ts`, `src/auth/jwt-strategy.ts` |
| Admin seed credentials are placeholder values | critical | `src/user/user.service.ts` |
| `synchronize: true` enabled (auto-schema mutation) | medium | `src/app.module.ts` |
| Throttle limit (10 req/60s global) may be too aggressive | medium | `src/app.module.ts` |
| `localStorage` used for token storage | low (acceptable for MVP) | `src/app/services/auth.service.ts` |

---

## 4. App contract: what Paul must deliver before infra is real

Based on the code review, this is the exact task list. Not guesses — these are traced to specific files and lines.

### 4.1 Backend contract (Paul's tasks)

```text
[ ] app.module.ts - TypeORM config must read from env vars
[ ] app.module.ts - Redis URL must come from process.env.REDIS_URL
[ ] app.module.ts - synchronize must be false in production
[ ] user.module.ts - JWT secret must come from process.env.JWT_SECRET
[ ] auth/jwt-strategy.ts - secretOrKey must come from process.env.JWT_SECRET
[ ] user/user.service.ts - admin seed credentials must come from env vars
[ ] main.ts - CORS origin must come from process.env.CORS_ORIGIN
[ ] main.ts - port must come from process.env.PORT
[ ] category.controller.ts - uncomment and wire admin guards on write routes
[ ] Remove committed Redis Labs credentials from source history
[ ] Add a /health or /api/health endpoint for operational checks
[ ] Add Dockerfile for backend
```

### 4.2 Frontend contract (Paul's tasks)

```text
[ ] Create environment.ts / environment.prod.ts with apiBaseUrl
[ ] admin.service.ts - use environment.apiBaseUrl
[ ] auth.service.ts - use environment.apiBaseUrl
[ ] quiz.service.ts - use environment.apiBaseUrl
[ ] Production build must set apiBaseUrl to '/api' (relative)
[ ] Add Dockerfile (multi-stage: build + serve or build-only for artifacts)
```

### 4.3 Why this matters

If any of these remain hard-coded, the infra deployment is not deploying a real application. It is deploying a localhost development setup on a public server. That is not a deployment — it is a liability.

---

## 5. Final architecture decision

### 5.1 Chosen stack (confirmed and locked)

| Layer | Choice | Confirmed by |
|---|---|---|
| Hosting | Hetzner VPS (CX32 for launch) | capacity estimation |
| OS | Debian 12 or Ubuntu 24.04 LTS | preference |
| Reverse proxy | Caddy (containerized) | architecture decision |
| Frontend | Angular 21 SPA → static assets | `angular.json`, `package.json` |
| Backend | NestJS 11 API | `package.json`, `nest-cli.json` |
| Database | PostgreSQL 16 (via TypeORM) | `app.module.ts`, `pg` dep |
| Cache | Redis 7 Alpine (via `@keyv/redis`) | `app.module.ts`, deps |
| Runtime | Docker Compose | architecture decision |
| Provisioning | Ansible | architecture decision |
| CI/CD | GitHub Actions (later) | architecture decision |

### 5.2 Architecture style

- single-node
- stateful (Postgres volume)
- vertical scaling first
- read-heavy optimized by simplicity

### 5.3 Explicitly out of scope for v1

Not part of the first implementation: Kubernetes, Swarm, multi-VPS HA, managed database, object storage, CDN, load balancer, blue/green deploys, service mesh, staging cluster, public Redis or Postgres.

These can all come later if real usage justifies them.

---

## 6. High-level system diagram

```text
                           INTERNET
                               |
                               v
                        [ DNS / Domain ]
                               |
                               v
                     +---------------------+
                     |   Hetzner VPS        |
                     |   CX32 (launch)      |
                     |   public :80/:443    |
                     +---------------------+
                               |
                               v
                     +---------------------+
                     |       Caddy         |
                     |  TLS termination    |
                     |  serve Angular SPA  |
                     |  proxy /api/*       |
                     +---------------------+
                         |             |
                         v             v
                +----------------+   +----------------+
                | frontend dist  |   | NestJS backend |
                | Angular static |   | port :3000     |
                | files          |   | (internal)     |
                +----------------+   +----------------+
                                             |
                                  +----------+----------+
                                  |                     |
                                  v                     v
                          +---------------+     +---------------+
                          | PostgreSQL 16 |     | Redis 7       |
                          | :5432         |     | :6379         |
                          | persistent    |     | disposable    |
                          +---------------+     +---------------+
```

---

## 7. Network model

### 7.1 Public exposure

Only these ports should be reachable from the internet:

- `80/tcp` → Caddy (HTTP, redirect to HTTPS or ACME challenges)
- `443/tcp` → Caddy (HTTPS)
- `22/tcp` → SSH (key-only, ideally restricted by source IP)

### 7.2 Private runtime ports

These stay internal to the Docker Compose network:

- backend `3000`
- postgres `5432`
- redis `6379`

### 7.3 Public traffic flow

```text
Browser
  |
  | HTTPS
  v
Caddy :443
  |
  +--> /           -> Angular static app (SPA)
  |
  +--> /api/*      -> backend:3000
  |
  +--> /users/*    -> backend:3000  (see routing note below)
  |
  +--> /questions/* -> backend:3000
  |
  +--> /categories/* -> backend:3000
```

**Important routing note:** The NestJS controllers currently use `/users`, `/questions`, `/categories` as top-level routes — not prefixed with `/api`. There are two ways to handle this:

**Option A (recommended):** Add a global prefix in NestJS (`app.setGlobalPrefix('api')`) and have Caddy proxy `/api/*` to the backend. This is clean and predictable.

**Option B:** Have Caddy proxy each specific path (`/users/*`, `/questions/*`, `/categories/*`) to the backend. This works but is fragile when new routes are added.

Recommend Option A. The frontend API base then becomes `/api` and all services prefix correctly.

### 7.4 Internal service flow

```text
backend
  |
  +--> postgres:5432   (persistent, source of truth)
  |
  +--> redis:6379      (cache, disposable)
```

---

## 8. Container topology

### 8.1 Docker Compose services

The production stack contains exactly four services:

1. `caddy` — reverse proxy, TLS, SPA serving
2. `backend` — NestJS API
3. `postgres` — database
4. `redis` — cache

Frontend is built into static assets and served directly by Caddy (not a separate container).

### 8.2 Runtime diagram

```text
+----------------------------------------------------------+
|                    Docker host (VPS)                      |
|                                                          |
|  +---------------------+                                 |
|  | caddy               |                                 |
|  | ports: 80,443       |                                 |
|  | mounts:             |                                 |
|  |   Caddyfile         |                                 |
|  |   frontend dist     |                                 |
|  |   caddy_data vol    |                                 |
|  |   caddy_config vol  |                                 |
|  +----------+----------+                                 |
|             |                                            |
|             v                                            |
|  +---------------------+                                 |
|  | backend             |                                 |
|  | internal: 3000      |                                 |
|  | env_file: .env      |                                 |
|  +----+-----------+----+                                 |
|       |           |                                      |
|       v           v                                      |
|  +---------+   +---------+                               |
|  |postgres |   | redis   |                               |
|  |5432 vol |   |6379     |                               |
|  +---------+   +---------+                               |
|                                                          |
+----------------------------------------------------------+
```

---

## 9. Environment variables — exact mapping

These are derived directly from the code. Every var maps to a specific line in the source.

### 9.1 backend.env

```env
# Runtime
NODE_ENV=production
PORT=3000

# CORS
CORS_ORIGIN=https://quiz.example.com

# JWT
JWT_SECRET=<generate: openssl rand -base64 48>

# Database (TypeORM)
DB_HOST=postgres
DB_PORT=5432
DB_NAME=quizdb
DB_USER=quizapp
DB_PASSWORD=<generate: openssl rand -base64 32>

# Redis
REDIS_URL=redis://redis:6379

# Admin seed
ADMIN_EMAIL=<real admin email>
ADMIN_PASSWORD=<generate: strong password>
```

### 9.2 postgres.env

```env
POSTGRES_DB=quizdb
POSTGRES_USER=quizapp
POSTGRES_PASSWORD=<must match DB_PASSWORD above>
```

### 9.3 Frontend build-time

```text
apiBaseUrl=/api
```

This gets baked into the Angular build via `environment.prod.ts`.

---

## 10. Docker Compose blueprint

```yaml
services:
  caddy:
    image: caddy:2
    container_name: quiz-caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - ./releases/current/frontend-dist:/srv/frontend:ro
      - caddy_data:/data
      - caddy_config:/config
    depends_on:
      - backend
    networks:
      - app_net

  backend:
    image: ghcr.io/ORG/quiz-backend:${TAG:-latest}
    container_name: quiz-backend
    restart: unless-stopped
    env_file:
      - ./env/backend.env
    expose:
      - "3000"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    networks:
      - app_net

  postgres:
    image: postgres:16
    container_name: quiz-postgres
    restart: unless-stopped
    env_file:
      - ./env/postgres.env
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U quizapp -d quizdb"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app_net

  redis:
    image: redis:7-alpine
    container_name: quiz-redis
    restart: unless-stopped
    command: ["redis-server", "--save", "", "--appendonly", "no"]
    networks:
      - app_net

networks:
  app_net:
    driver: bridge

volumes:
  postgres_data:
  caddy_data:
  caddy_config:
```

### 10.1 Design points

- Only Caddy publishes host ports
- Backend uses `expose`, not `ports`
- Postgres has a healthcheck so backend waits for it
- Redis has no persistence (cache only)
- Compose network is private to the stack
- Backend image comes from a container registry, frontend is a synced directory

---

## 11. Caddyfile

```caddy
quiz.example.com {
    encode zstd gzip

    # Backend API proxy
    handle /api/* {
        reverse_proxy backend:3000
    }

    # SPA static files
    root * /srv/frontend
    try_files {path} /index.html
    file_server

    log {
        output file /var/log/caddy/quiz-access.log
        format console
    }
}
```

### 11.1 Notes

- `try_files {path} /index.html` is required for Angular SPA route refreshes
- `handle /api/*` must appear before the file_server block so API requests are intercepted first
- `backend` resolves via Docker DNS on the `app_net` network
- Caddy automatically obtains and renews TLS certificates via Let's Encrypt
- If NestJS uses `app.setGlobalPrefix('api')`, then Caddy sends requests to `/api/*` and they arrive at the backend as `/api/*` — backend routes then match `/api/users/login`, etc.

---

## 12. Dockerfiles

### 12.1 Backend Dockerfile

```dockerfile
FROM node:22-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production=false
COPY . .
RUN npm run build

FROM node:22-alpine AS runtime
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY --from=build /app/node_modules ./node_modules
COPY --from=build /app/package.json ./
EXPOSE 3000
CMD ["node", "dist/main"]
```

### 12.2 Frontend Dockerfile (build-only, output artifacts)

```dockerfile
FROM node:22-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npx ng build --configuration=production
# Output: /app/dist/medical-quiz/browser/
```

The frontend build outputs static files that get copied to the VPS and served by Caddy. There is no frontend runtime container.

---

## 13. Directory layout on VPS

```text
/srv/quiz-app/
├── compose.yml
├── Caddyfile
├── env/
│   ├── backend.env          # secrets: DB, JWT, Redis, admin
│   └── postgres.env         # secrets: DB credentials
├── releases/
│   ├── current/
│   │   └── frontend-dist/   # Angular build output
│   └── previous/            # rollback copy
├── backups/
│   └── postgres/            # pg_dump archives
└── scripts/
    ├── deploy.sh            # optional helper
    └── backup-postgres.sh   # cron target
```

---

## 14. Persistent vs disposable data

| Component | Persistent? | Why |
|---|---|---|
| PostgreSQL data | yes | source of truth for users, questions, categories |
| Caddy cert state (`/data`, `/config`) | yes | avoid unnecessary cert reissuance |
| Redis | no | daily quiz cache, fully reconstructable |
| Backend container filesystem | no | stateless app |
| Frontend dist | rebuildable, but keep current + previous | rollback convenience |

---

## 15. Database strategy

### 15.1 PostgreSQL is the source of truth

One local Postgres container, one persistent volume, no public port, one application DB, one application user.

### 15.2 Migrations vs synchronize

The backend currently uses `synchronize: true`. In production this must be `false`. TypeORM migrations should be set up before or shortly after the first production deploy.

For the very first deployment, `synchronize: true` can be tolerated temporarily if the schema is still being finalized. But this must be tracked as explicit tech debt with a concrete deadline.

### 15.3 Backup strategy

Minimum viable backup:

- Daily `pg_dump` via cron or Ansible
- Compressed, timestamped files: `quizdb_YYYY-MM-DD_HHMMSS.sql.gz`
- Keep 14 days locally
- Optional: sync off-host to object storage later

Redis is not part of backups. That is intentional.

---

## 16. Redis strategy

Redis exists to cache generated daily quiz data. The app uses `@keyv/redis` via NestJS `CacheModule`.

**Current state in code:** The app connects to a cloud Redis Labs instance with credentials in source. This must change to a local Redis container with the URL from env vars.

**Operational rule:** Treat Redis as disposable. The app must survive its loss — if Redis is down, the daily quiz should regenerate from Postgres (the code already does this: cache miss → generate → store).

---

## 17. Security model

### 17.1 Baseline rules

- Only SSH, HTTP, HTTPS exposed on host
- DB and Redis internal to Docker network only
- Secrets outside Git, managed via Ansible Vault
- TLS for all public traffic (Caddy handles this)
- Admin write routes protected server-side (currently broken — guards are commented out)

### 17.2 Host hardening checklist

- SSH key auth only, password auth disabled
- Root login disabled or heavily restricted
- Firewall enabled: allow 22, 80, 443 only
- Unattended security updates enabled
- Docker installed from official repo

### 17.3 Docker security

- No unnecessary published ports (only Caddy exposes 80/443)
- Restart policies on all containers
- Named volumes for persistent state
- No secrets in image layers

### 17.4 Application security (Paul's responsibility)

- All hard-coded credentials replaced with env vars
- Admin guards re-enabled on category and question write routes
- CORS origin set to production domain only
- JWT secret is a strong random value, not a placeholder
- Redis Labs credentials removed from source history (consider git filter-branch or BFG)

### 17.5 Secrets management

Secrets live in:

- Ansible Vault (for deployment)
- Generated env files on target host
- GitHub Secrets (for CI/CD later)

Never in:

- Committed repo files
- Docker image layers
- Compose files with real values

---

## 18. Firewall posture

Allow inbound:

- `22/tcp` (SSH)
- `80/tcp` (HTTP)
- `443/tcp` (HTTPS)

Deny everything else by default.

Docker's default behavior can punch holes through iptables. Mitigate by never using `ports:` on anything except Caddy. The private Compose network handles internal communication.

---

## 19. Observability

### 19.1 Day 1 requirements

- `docker compose ps` for container status
- `docker compose logs` for all services
- Caddy access logs (file output)
- A health endpoint on the backend (needs to be added)

### 19.2 What can wait

Prometheus, Grafana, Loki, alerting stacks, distributed tracing. Not because they're bad — but because this app first needs stable deployment basics.

### 19.3 Recommended health endpoint

Add to the NestJS app:

```text
GET /api/health → { status: "ok", timestamp: "...", db: "connected", redis: "connected" }
```

This gives Caddy, monitoring, and smoke tests something concrete to check.

---

## 20. Deployment strategy

### 20.1 Correct rollout order

```text
1. App env contract complete (all hard-coded values replaced)
2. Local Compose stack works (not against localhost services)
3. First manual VPS deploy
4. First Ansible deploy
5. Smoke tests pass
6. Backup tested and restore verified
7. CI/CD automation (GitHub Actions)
```

If you reverse that order, you automate uncertainty.

### 20.2 Frontend artifact flow

```text
App repo CI or local build
        |
        v
Angular dist artifacts (static files)
        |
        v
Synced to VPS /srv/quiz-app/releases/current/frontend-dist/
        |
        v
Caddy serves them
```

### 20.3 Backend artifact flow

```text
App repo builds backend Docker image
        |
        v
Push to container registry (GHCR)
        |
        v
VPS pulls image during deploy
        |
        v
Compose runs it
```

---

## 21. Ansible repo design

```text
ansible-quiz-app/
├── inventories/
│   └── production/
│       ├── hosts.yml
│       └── group_vars/
│           ├── all.yml
│           └── vault.yml
├── playbooks/
│   ├── bootstrap.yml       # OS baseline
│   ├── deploy.yml          # full deploy
│   ├── update.yml          # update images/artifacts
│   ├── backup.yml          # pg_dump
│   ├── restore.yml         # pg restore
│   └── verify.yml          # smoke checks
├── roles/
│   ├── common/             # users, packages, firewall, dirs
│   │   ├── tasks/
│   │   ├── templates/
│   │   └── handlers/
│   ├── docker/             # engine + compose plugin
│   │   ├── tasks/
│   │   └── handlers/
│   ├── reverse_proxy/      # Caddy config + lifecycle
│   │   ├── tasks/
│   │   └── templates/
│   ├── quiz_stack/         # compose, env files, releases
│   │   ├── tasks/
│   │   ├── templates/
│   │   ├── files/
│   │   └── handlers/
│   └── backup/             # pg dump + restore helpers
│       ├── tasks/
│       └── templates/
└── README.md
```

### 21.1 Variable model

Non-secret vars (`group_vars/all.yml`):

```yaml
quiz_domain: quiz.example.com
quiz_app_root: /srv/quiz-app
quiz_backend_port: 3000
quiz_backend_image: ghcr.io/ORG/quiz-backend
quiz_backend_tag: latest
quiz_postgres_db: quizdb
quiz_postgres_user: quizapp
quiz_redis_url: redis://redis:6379
quiz_caddy_email: ops@example.com
quiz_global_api_prefix: api
quiz_throttle_ttl: 60000
quiz_throttle_limit: 30
```

Secret vars (`group_vars/vault.yml` — encrypted):

```yaml
vault_quiz_postgres_password: "..."
vault_quiz_jwt_secret: "..."
vault_quiz_admin_email: "..."
vault_quiz_admin_password: "..."
```

---

## 22. Backup and restore

### 22.1 Backup flow

```text
cron (daily) or ansible-playbook backup.yml
    |
    v
docker compose exec postgres pg_dump -U quizapp quizdb | gzip
    |
    v
/srv/quiz-app/backups/postgres/quizdb_YYYY-MM-DD_HHMMSS.sql.gz
```

### 22.2 Restore flow

```text
1. Stop backend: docker compose stop backend
2. Restore: gunzip < dump.sql.gz | docker compose exec -T postgres psql -U quizapp quizdb
3. Start backend: docker compose start backend
4. Verify: curl https://quiz.example.com/api/health
```

### 22.3 Retention

Keep 14 days locally. Prune older dumps via cron or Ansible.

---

## 23. Release and rollback

### 23.1 Frontend rollback

```text
mv releases/current releases/broken
mv releases/previous releases/current
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
```

### 23.2 Backend rollback

Change image tag in compose file or env var, then:

```bash
docker compose pull backend
docker compose up -d backend
```

### 23.3 Database rollback

Restore from most recent pg_dump before the broken deploy. This is never free — always back up before schema-changing deployments.

---

## 24. Verification checklist after first deploy

### 24.1 Platform

- [ ] Domain resolves to VPS IP
- [ ] Port 80 redirects to 443
- [ ] Port 443 serves valid TLS cert
- [ ] SSH access works with key

### 24.2 Frontend

- [ ] Homepage loads
- [ ] SPA route refresh works (e.g. `/login` direct access)
- [ ] Static assets (JS/CSS) load correctly
- [ ] No `localhost:3000` references in browser network tab

### 24.3 API

- [ ] `/api/health` returns 200 (once added)
- [ ] `/api/users/login` responds (POST)
- [ ] `/api/questions/daily` returns quiz data
- [ ] No CORS errors in browser console
- [ ] Rate limiting works but doesn't block normal usage

### 24.4 Data

- [ ] Backend reads/writes Postgres successfully
- [ ] Redis cache populates on first quiz fetch
- [ ] DB data survives `docker compose restart postgres`

### 24.5 Security

- [ ] Backend port 3000 not reachable from internet
- [ ] Postgres port 5432 not reachable from internet
- [ ] Redis port 6379 not reachable from internet
- [ ] No hard-coded credentials visible in container env inspection
- [ ] Admin routes are actually guarded

### 24.6 Functional smoke tests

- [ ] Login works
- [ ] Register works
- [ ] Daily quiz loads with questions and answers
- [ ] Admin login works
- [ ] Category creation works (admin only)
- [ ] Question creation works (admin only)

---

## 25. Risks and explicit technical debt

### 25.1 App-side tech debt (must fix before or shortly after first deploy)

| Item | Severity | Owner |
|---|---|---|
| Hard-coded credentials throughout backend | critical | Paul |
| Hard-coded frontend API URLs | critical | Paul |
| Category admin guards commented out | high | Paul |
| Redis Labs credentials in git history | high | Paul |
| `synchronize: true` in TypeORM | medium | Paul |
| Global throttle at 10 req/60s may be too aggressive | medium | Paul |
| Daily quiz cache TTL hard-coded to 1000ms | low | Paul |
| No health endpoint | low | Paul |
| Stripe fields on UserEntity but no Stripe integration | info | future |
| Premium guard exists but no premium flow | info | future |

### 25.2 Infra-side tech debt (acceptable for MVP)

| Item | Severity | Owner |
|---|---|---|
| No monitoring stack | low | Radu |
| Backups local-only initially | low | Radu |
| No staging environment | low | Radu |
| No CI/CD initially | low | Radu |
| Manual artifact handling for first deploy | acceptable | Radu |

### 25.3 Not acceptable for first production deploy

- Hard-coded production secrets in repo
- Public DB or Redis ports
- Frontend calling `localhost:3000`
- Deploy without a tested backup/restore path
- Category/question write endpoints unprotected

---

## 26. Implementation phases

### Phase 0 — App contract cleanup (Paul)

Everything in Section 4. This blocks all infra work.

### Phase 1 — Local integration (Radu + Paul)

Prove the Compose stack works with env vars, not localhost assumptions. Caddy serves frontend, proxies backend. Postgres and Redis containers replace local/cloud services.

### Phase 2 — First VPS deploy (Radu)

Bring stack up on Hetzner manually. Working domain, working TLS, working DB persistence, working app end to end.

### Phase 3 — Ansible formalization (Radu)

Turn the manual deploy into reproducible Ansible: bootstrap, deploy, verify, backup playbooks.

### Phase 4 — CI/CD (Radu)

GitHub Actions trigger known-good deployments. App repo publishes artifacts. Infra repo runs deploy.

---

## 27. Practical build order

```text
1. Paul removes all hard-coded values
2. Paul adds Dockerfiles
3. Paul adds /api/health endpoint
4. Paul re-enables admin guards
5. Radu writes compose.yml + Caddyfile + env templates
6. Both test locally with docker compose up
7. Radu provisions Hetzner VPS
8. Radu does first manual deploy
9. Radu converts to Ansible
10. Radu adds backup cron and tests restore
11. Radu adds GitHub Actions
```

Step 1 blocks everything. Step 6 validates everything. Step 8 is the real milestone.

---

## 28. Concrete task list

### App repo tasks (Paul)

- [ ] Replace all hard-coded backend values with env vars (Section 3.1)
- [ ] Replace all hard-coded frontend API URLs (Section 3.2)
- [ ] Uncomment and wire admin guards on category controller
- [ ] Add global API prefix (`app.setGlobalPrefix('api')`)
- [ ] Add `/api/health` endpoint
- [ ] Add backend Dockerfile
- [ ] Add frontend Dockerfile or build script
- [ ] Create `environment.prod.ts` with `apiBaseUrl: '/api'`
- [ ] Fix daily quiz cache TTL (currently 1000ms)
- [ ] Review throttle limits for production usage
- [ ] Remove Redis Labs credentials from git history

### Infra repo tasks (Radu)

- [ ] Create Ansible inventory for production
- [ ] Create `common` role (users, packages, firewall)
- [ ] Create `docker` role (engine + compose)
- [ ] Create `reverse_proxy` role (Caddy)
- [ ] Create `quiz_stack` role (compose, env, releases)
- [ ] Create `backup` role (pg_dump, retention, restore)
- [ ] Write `bootstrap.yml`
- [ ] Write `deploy.yml`
- [ ] Write `verify.yml`
- [ ] Write `backup.yml`
- [ ] Write `restore.yml`

### VPS tasks (Radu)

- [ ] Provision Hetzner CX32
- [ ] Add SSH key auth
- [ ] Set up firewall (22, 80, 443 only)
- [ ] Install Docker + Compose plugin
- [ ] Create `/srv/quiz-app` directory structure
- [ ] Deploy env files from Ansible Vault
- [ ] Deploy compose + Caddyfile
- [ ] Start stack
- [ ] Verify public access
- [ ] Run full smoke test checklist

---

## 29. Estimated monthly cost

### Launch (CX32)

| Item | Cost |
|---|---|
| Hetzner CX32 (4 vCPU, 8GB RAM, 80GB NVMe) | ~€9-12 |
| Domain | ~€1-2 |
| Cloudflare (Free tier, optional for DNS) | €0 |
| **Total** | **~€10-14/month** |

### After stabilization (potential downscale to CX22)

| Item | Cost |
|---|---|
| Hetzner CX22 (2 vCPU, 4GB RAM, 40GB NVMe) | ~€5 |
| Domain | ~€1-2 |
| **Total** | **~€6-7/month** |

---

## 30. Architecture decision register

```text
ADR-001: Deploy on single Hetzner VPS (CX32 for launch)
ADR-002: Use Caddy as reverse proxy (containerized)
ADR-003: Use PostgreSQL 16 as primary datastore
ADR-004: Use Redis 7 as disposable cache only
ADR-005: Expose only 80/443/22 publicly
ADR-006: Deploy with Docker Compose
ADR-007: Provision and deploy with Ansible
ADR-008: Add CI/CD only after manual/Ansible deploy is proven
ADR-009: Frontend served as static files by Caddy (no separate container)
ADR-010: Backend uses global /api prefix for clean routing
ADR-011: All secrets managed via Ansible Vault, never in source
ADR-012: Replace cloud Redis Labs with local Redis container
```

---

## 31. Appendix — smoke test commands

```bash
# Container status
docker compose ps

# Service logs
docker compose logs caddy --tail 100
docker compose logs backend --tail 100
docker compose logs postgres --tail 50
docker compose logs redis --tail 50

# DB readiness
docker compose exec postgres pg_isready -U quizapp -d quizdb

# Public endpoints
curl -I https://quiz.example.com/
curl -s https://quiz.example.com/api/health | jq
curl -s -X POST https://quiz.example.com/api/users/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@test.com","password":"test123"}'

# Internal DNS resolution
docker compose exec backend sh -c 'getent hosts postgres redis'

# Port scan (from outside)
nmap -p 3000,5432,6379 quiz.example.com
# Expected: all filtered/closed
```

---

## 32. Appendix — quick reference

```text
Stack:
  Hetzner CX32 → Caddy → Angular static + NestJS API → Postgres + Redis

Expose:
  80/443/22 only

Internal:
  backend:3000, postgres:5432, redis:6379

Persist:
  Postgres volume, Caddy cert data

Disposable:
  Redis, backend container FS

Env files:
  backend.env → DB, JWT, Redis, CORS, admin creds
  postgres.env → DB name, user, password

Build order:
  1. env vars in app
  2. Dockerfiles
  3. local compose test
  4. first VPS deploy
  5. Ansible
  6. backups
  7. CI/CD
```

---

## 33. Final conclusion

The app exists. The code has been reviewed. The hard-coded values have been identified. The architecture decisions have been locked.

What remains is disciplined execution:

1. Paul cleans the runtime contract
2. Radu wraps it in the smallest reliable infrastructure
3. Both verify it works end to end

The hardest part is not Caddy or Compose or Ansible. The hardest part is making sure the app actually reads its configuration from the environment instead of from string literals in source code.

Once that happens, the rest is a sequence of well-understood steps.
