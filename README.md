# IMS — Pharmaceutical Inventory Management System

A production-grade warehouse management system for pharmaceutical and manufacturing operations,
covering the full regulated material lifecycle: **material catalogue → goods receipt →
quarantine → QC testing → release or rejection → production consumption → traceability
labelling → audit reporting.**

Nine services, five role-based dashboards, an operational RAG chatbot, a complete
OpenTelemetry observability stack, and continuous deployment to Fly.io and Vercel.

| | |
|---|---|
| **Type** | Team project — 6 developers, Scrum |
| **Duration** | Jan 2026 – Apr 2026 |
| **Course** | Software Engineering Capstone (SEC) — VNUHCM University of Science |
| **Original repository** | [Inventory-management-SEC/SEC_Team_02_2026](https://github.com/Inventory-management-SEC/SEC_Team_02_2026) |
| **My commits** | 18 of 295 |
| **Codebase** | ~17k LOC backend (TypeScript) · ~12k LOC frontend (TypeScript/React) · 2 Python services |
| **Tests** | 28 backend Jest suites · 23 frontend Vitest suites · Playwright E2E harness |

**Live:** [Frontend](https://ims-frontend-sec02.vercel.app) ·
[Backend API](https://ims-backend-sec02.fly.dev) ·
[Health check](https://ims-backend-sec02.fly.dev/health)

---

## Table of contents

- [Why this project is interesting](#why-this-project-is-interesting)
- [The core business rule](#the-core-business-rule)
- [Feature overview](#feature-overview)
- [System architecture](#system-architecture)
- [Tech stack](#tech-stack)
- [Backend design](#backend-design)
- [Frontend design](#frontend-design)
- [RAG chatbot service](#rag-chatbot-service)
- [Observability stack](#observability-stack)
- [Security model](#security-model)
- [Data model](#data-model)
- [CI/CD and deployment](#cicd-and-deployment)
- [Testing](#testing)
- [Repository layout](#repository-layout)
- [Running locally](#running-locally)
- [My contributions](#my-contributions)
- [Engineering notes](#engineering-notes)

---

## Why this project is interesting

Most university capstone projects are CRUD apps with a login screen. This one is not, and the
differences are the point:

1. **The domain has real rules.** Pharmaceutical inventory is regulated — a lot cannot go into
   production until QC releases it, consumption must be atomic with the transaction ledger, and
   every movement must be auditable. Those constraints are enforced in the service layer and the
   database, not just the UI.
2. **Authentication is delegated, not hand-rolled.** Identity runs on **Keycloak 23** with
   OAuth2/OIDC, and the frontend refreshes tokens transparently through an Axios interceptor.
3. **It is genuinely observable.** OpenTelemetry traces from *both* browser and server flow into
   a Grafana/Tempo/Loki/Prometheus stack with provisioned dashboards and Alertmanager rules — not
   a `console.log`.
4. **The RAG service is service-to-service authenticated.** Every `/v1/*` call carries an
   HMAC-SHA256 signature over `timestamp.body`, so the chatbot cannot be driven directly from the
   internet.
5. **It actually ships.** Three GitHub Actions pipelines type-check, test and deploy backend, RAG
   service and frontend independently on path-filtered pushes.

## The core business rule

Everything in the system revolves around the lot status machine:

```
                      ┌──────────────┐
   goods receipt ───► │  Quarantine  │
                      └──────┬───────┘
                             │ QC test recorded
                  ┌──────────┴──────────┐
                  ▼                     ▼
          ┌──────────────┐      ┌──────────────┐
          │   Accepted   │      │   Rejected   │
          └──────┬───────┘      └──────────────┘
                 │ may be added to a production batch
                 ▼
          ┌──────────────┐
          │   Depleted   │ ◄─── quantity reaches zero
          └──────────────┘
```

**Invariants the code enforces:**

- A lot must be `Accepted` before it can be attached to a production batch.
- A batch must be `In Progress` before any material can be consumed from it.
- Consuming material is a single atomic operation that ***simultaneously***
  updates `batch_components.actual_quantity`, decrements `inventory_lots.quantity`
  (flipping the lot to `Depleted` at zero), and appends an `inventory_transactions` row.

That last invariant is why the transaction ledger can be trusted as an audit trail: there is no
code path that moves stock without writing history.

## Feature overview

| Module | Capabilities |
|---|---|
| **Materials** | Catalogue CRUD, compliance metadata, Elasticsearch full-text search |
| **Lots** | Goods receipt, lifecycle transitions, quantity tracking, expiry |
| **QC** | Test recording, QC queue, approve/reject with lot status transition, test-detail modals |
| **Production** | Batch creation, component allocation, atomic consumption, traceability |
| **Transactions** | IN/OUT ledger, immutable movement history |
| **Labels** | Template designer with field selection, Barcode (bwip-js) + QR (qrcode) generation, entity-bound labels, PDF export |
| **Reports** | Inventory / transaction / audit reports, PDF export via jsPDF |
| **Search** | Elasticsearch 8 full-text index across materials and lots |
| **Admin** | User management, role assignment, system statistics |
| **Dashboards** | Five role-specific views: Admin, Inventory Manager, Quality Control, Production, Viewer |
| **Chatbot** | Operational RAG assistant answering inventory questions in Vietnamese and English |

## System architecture

```
                            ┌──────────────────────────────┐
                            │  Browser                     │
                            │  React 19 SPA (Vercel)       │
                            │  ├─ keycloak-js (OIDC)       │
                            │  ├─ TanStack Query cache     │
                            │  └─ OTel web tracing ────────┼──┐
                            └───────────┬──────────────────┘  │
                                        │ REST + Bearer JWT   │
                                        ▼                     │
     ┌──────────────┐          ┌────────────────────────┐     │
     │  Keycloak 23 │ ◄────────┤  Express 4 + TS API    │     │
     │  OAuth2/OIDC │  verify  │  (Fly.io, Singapore)   │     │
     │  realm:      │          │                        │     │
     │  inventory   │          │  security/auth  (JWT)  │     │
     └──────────────┘          │  security/rbac  (matrix)│    │
                               │  13 domain modules     │     │
                               │  OTel node SDK ────────┼──┐  │
                               └──┬────┬────┬────┬──────┘  │  │
                                  │    │    │    │         │  │
              ┌───────────────────┘    │    │    └──────┐  │  │
              ▼                        ▼    ▼           ▼  │  │
     ┌─────────────────┐   ┌────────┐ ┌──────────────┐ ┌───┴──┴──────────┐
     │ PostgreSQL 16   │   │ Redis 7│ │ Elasticsearch│ │ RAG service     │
     │ (Supabase prod) │   │ cache  │ │ 8 full-text  │ │ FastAPI, Fly.io │
     │ 9 tables        │   └────────┘ └──────────────┘ │ HMAC-signed     │
     └─────────────────┘                               │ ├─ ingest       │
                                                       │ ├─ retrieve     │
              ┌────────────────────────────────────────┤ └─ answer       │
              │                                        └────────┬────────┘
              ▼                                                 ▼
     ┌──────────────────┐                              ┌─────────────────┐
     │ AI service       │                              │ Ollama 0.6.6    │
     │ FastAPI analytics│                              │ local LLM +     │
     └──────────────────┘                              │ embeddings      │
                                                       └─────────────────┘

     ═══════════════ Observability plane ═══════════════════════════════
     OTel Collector ──► Tempo (traces) ──┐
     Promtail ────────► Loki (logs) ─────┼──► Grafana ──► provisioned
     Prometheus (metrics) ───────────────┘                dashboards
                    └──► Alertmanager ──► SMTP alerts
```

## Tech stack

### Frontend
| Concern | Choice |
|---|---|
| Framework | React 19.2 |
| Build | Vite 7.3 + `@vitejs/plugin-react` |
| Language | TypeScript 5.9 (strict, `@/*` path alias) |
| UI kit | Ant Design 6 + `@ant-design/icons` + `@ant-design/charts` |
| Styling | Tailwind CSS 4 (`@tailwindcss/vite`), PostCSS, Autoprefixer |
| Server state | TanStack React Query 5 (one hook per domain) |
| Client state | Zustand 5 (UI/sidebar only) |
| Tables | TanStack React Table 8 with shared column factories |
| Charts | Recharts 3 + Ant Design Charts |
| Routing | React Router 7 |
| Auth | `keycloak-js` 26 + `AuthProvider` context |
| HTTP | Axios 1.13 with token-refresh interceptor |
| PDF | jsPDF 4 + `jspdf-autotable` |
| Dates | Day.js |
| Telemetry | OpenTelemetry web SDK, `instrumentation-document-load`, context-zone, `web-vitals` |
| Testing | Vitest 4, Testing Library (React + user-event + jest-dom), jsdom, v8 coverage |
| Lint | ESLint 9 flat config + typescript-eslint 8 + react-hooks 7 |

### Backend
| Concern | Choice |
|---|---|
| Runtime | Node.js 22 |
| Framework | Express 4 + TypeScript 5.1 |
| Database | `pg` 8 connection pool (`DATABASE_URL` or discrete `DB_*` vars) |
| Cache | `redis` 5 — optional, app degrades gracefully |
| Search | `@elastic/elasticsearch` 8 — optional |
| Auth | `jsonwebtoken` — Keycloak JWT verification |
| Barcodes | `bwip-js` 4, `jsbarcode` 3, `qrcode` 1.5 |
| Logging | `pino` 9 structured JSON |
| Metrics | `prom-client` 15 (`/metrics`, token-guarded) |
| Tracing | OpenTelemetry Node SDK + auto-instrumentations + OTLP HTTP exporter |
| Testing | Jest 29 + ts-jest + Supertest 7 |
| Deploy | `@flydotio/dockerfile`, multi-stage Dockerfile, `fly.toml` |

### Python services
| Service | Stack |
|---|---|
| **rag-service** | FastAPI 0.109 · Uvicorn · Pydantic 2 · `psycopg[binary]` 3 · PyMongo 4 · NumPy · APScheduler · `prometheus-client` · httpx |
| **ai-service** | FastAPI 0.109 · async Redis 5 · async Elasticsearch 8 · aiohttp · `python-multipart` |

### Infrastructure
| Concern | Choice |
|---|---|
| Orchestration (dev) | Docker Compose — 16 services with health checks |
| Database | PostgreSQL 16 (Supabase in production) |
| Identity | Keycloak 23, realm exported to `keycloak/inventory-realm.json` |
| LLM runtime | Ollama 0.6.6 (self-hosted models + embeddings) |
| Backend host | Fly.io (`ims-backend-sec02`, region `sin`, node:22-slim) |
| Frontend host | Vercel (`ims-frontend-sec02`) |
| CI/CD | GitHub Actions — three path-filtered pipelines |
| Web server | nginx (frontend container) |

## Backend design

Every domain is a self-contained module with an identical shape, which makes the codebase
navigable without a map:

```
src/modules/<domain>/
  <domain>.routes.ts     # Express router, RBAC guards, request validation
  <domain>.service.ts    # business logic, SQL, transactions
  <domain>.types.ts      # request/response contracts
  __tests__/             # colocated Jest suites
```

**13 modules:** `admin` · `auth` · `chat` · `dashboard` · `labels` · `lots` · `materials` ·
`production` · `qc` · `rag` · `reports` · `search` · `transactions`
(plus `modules/__tests__/` for cross-module integration tests).

**Shared infrastructure**

| Path | Responsibility |
|---|---|
| `security/auth.ts` | JWT verification middleware (Keycloak-issued tokens) |
| `security/rbac.ts` | `PERMISSIONS` matrix — `resource → action → UserRole[]` |
| `shared/db/pool.ts` | Postgres pool, dual config strategy |
| `shared/cache/redis.ts` | Redis client with `reconnectStrategy: false` so a dead Redis never blocks boot |
| `shared/elasticsearch/client.ts` | ES client, optional |

**API response contracts** are uniform: `ApiResponse<T>` for single resources,
`PaginatedResponse<T>` for collections. Routes are mounted under `/api/<domain>`.

**Graceful degradation** is a deliberate design choice — Redis and Elasticsearch are both
optional. The app boots and serves traffic without them, losing only caching and full-text
search. This is why the production Fly.io deployment runs with just Postgres.

## Frontend design

```
src/
├── auth/                      # Keycloak init + AuthProvider context
├── router/index.tsx           # route table, role-gated
├── components/
│   ├── layout/AppLayout.tsx   # sidebar shell
│   ├── dashboard/             # KpiCard · ChartCard · AlertPanel · DataTableCard
│   ├── common/tables/         # columnFactories.tsx — shared TanStack column builders
│   ├── qc/ labels/ lots/      # domain modals & forms
├── pages/
│   ├── dashboard/             # Admin · InventoryManager · QualityControl · Production
│   └── materials/ lots/ qc/ batches/ labels/ transactions/ reports/ users/
├── hooks/                     # useDashboardData · useMaterialsData · useTransactionsData
│                              # useLotsData · useQCData · useBatchesData · useLabelsData
│                              # useReportsData · useUsersData
├── services/api.ts            # Axios client, Keycloak interceptor, auto-refresh on 401
├── stores/uiStore.ts          # Zustand — sidebar state only
├── constants/                 # roles.ts · theme.ts (antd theme tokens)
├── lib/                       # utils.ts · exportUtils.ts (jsPDF)
└── types/index.ts             # every API contract in one place
```

Two conventions carry most of the weight:

- **One React Query hook per domain.** Caching, invalidation and loading states live in the hook,
  so pages stay declarative and cache invalidation is impossible to forget.
- **Shared column factories.** `columnFactories.tsx` builds TanStack Table columns
  (status tags, date cells, action menus) once, so nine data grids look and behave identically.

## RAG chatbot service

A standalone FastAPI microservice (~1,700 LOC) that answers operational questions about live
inventory — *"which lots expire this month?"*, *"how much paracetamol is in quarantine?"*

**Ingestion pipeline**

1. Pulls from PostgreSQL (`materials`, `inventory_lots`) and optional MongoDB collections
2. Builds three document shapes: a global inventory summary, a per-lot operational record, and a
   per-material summary
3. Generates embeddings and upserts vectors into the `ims_inventory` namespace
4. Supports full reindex, manual reindex and incremental sync
5. **APScheduler** runs the sync on a schedule (`RAG_ENABLE_SCHEDULED_SYNC=true` by default)

**API**

| Endpoint | Purpose |
|---|---|
| `POST /v1/retrieve` | Vector retrieval only |
| `POST /v1/answer` | Retrieval + LLM synthesis (`vi-VN` and English) |
| `POST /v1/ingest/documents` | Ingest arbitrary documents |
| `POST /v1/ingest/reindex` · `/rag/reindex` | Full reindex |
| `POST /v1/ingest/inventory/reindex` · `/sync` | Inventory-specific reindex / incremental sync |
| `GET /v1/ingest/status/{job_id}` | Async job status |
| `GET /health` · `GET /metrics` | Liveness + Prometheus metrics |

**Service-to-service authentication.** Every `/v1/*` request must carry:

```
x-rag-timestamp: <unix epoch seconds>
x-rag-signature: <hex HMAC-SHA256 of "timestamp.body" using RAG_SERVICE_SHARED_SECRET>
```

Both the Express backend and the RAG service hold the shared secret. The timestamp makes replay
attacks bounded; the signature means an attacker who finds the Fly.io URL still cannot query it.

## Observability stack

Wired end to end, from browser paint to database query:

| Component | Image | Role |
|---|---|---|
| **OTel Collector** | `otel/opentelemetry-collector-contrib:0.121.0` | Single OTLP ingest point for browser + server spans |
| **Tempo** | `grafana/tempo:2.7.2` | Distributed trace storage |
| **Loki** | `grafana/loki:3.4.2` | Log aggregation |
| **Promtail** | `grafana/promtail:3.4.2` | Container log shipping |
| **Prometheus** | `prom/prometheus:v3.3.1` | Metrics scraping + `alerts.yml` rules |
| **Alertmanager** | `prom/alertmanager:v0.28.1` | Alert routing to SMTP |
| **Grafana** | `grafana/grafana:11.6.0` | Provisioned datasources + `ims-observability-overview` dashboard |

The frontend emits real user monitoring data (`instrumentation-document-load`, `web-vitals`,
context-zone for async correlation), the backend emits auto-instrumented HTTP/PG/Redis spans, and
both export OTLP over HTTP to the same collector — so a single trace spans browser click →
API handler → SQL query. The Prometheus `/metrics` endpoint on the backend is guarded by
`METRICS_AUTH_TOKEN`.

## Security model

**Authentication** — Keycloak 23 issues OIDC JWTs. The realm (`inventory-realm.json`) is
version-controlled, so roles and clients are reproducible. `security/auth.ts` verifies the token
on every protected route. A `BYPASS_AUTH` flag exists for local development only.

**Authorisation** — a declarative permission matrix rather than scattered `if (user.role ===`
checks:

```ts
export const PERMISSIONS = {
  materials: {
    read:   [ADMIN, INVENTORY_MANAGER, QUALITY_CONTROL, PRODUCTION, VIEWER],
    create: [ADMIN, INVENTORY_MANAGER],
    update: [ADMIN, INVENTORY_MANAGER],
    delete: [ADMIN],
  },
  lots: {
    // ...
    updateStatus: [ADMIN, INVENTORY_MANAGER, QUALITY_CONTROL],
    delete: [ADMIN],
  },
  // transactions, qc, production, labels, reports, admin …
};
```

Note `lots.updateStatus` — QC can move a lot between statuses but cannot create or delete lots.
That kind of fine-grained action, distinct from plain `update`, is exactly what a
resource × action × role matrix buys you over role checks sprinkled through handlers.

**Five roles:** `admin` · `inventory_manager` · `quality_control` · `production` · `viewer`,
each landing on a different dashboard.

**Service-to-service:** HMAC-SHA256 request signing between backend and RAG service.

## Data model

PostgreSQL 16, defined in [`db_schema/db-init.sql`](02_Source/01_Source%20Code/db_schema/db-init.sql):

| Table | Role |
|---|---|
| `users` | Accounts and role assignment |
| `materials` | Material catalogue with compliance metadata |
| `inventory_lots` | Physical lots — quantity, status, expiry, FK to material |
| `inventory_transactions` | Immutable IN/OUT ledger — the audit trail |
| `qc_tests` | QC test records tied to lots |
| `production_batches` | Manufacturing batches with status |
| `batch_components` | Lots allocated to a batch, planned vs `actual_quantity` |
| `label_templates` | Reusable label layouts with field selection |
| `generated_labels` | Issued label instances bound to entities |

`db_schema/backup.sh` handles dumps; `db_schema/docker-compose.yml` runs Postgres standalone.

## CI/CD and deployment

Three independent, **path-filtered** GitHub Actions pipelines — touching the frontend does not
redeploy the backend:

| Workflow | Trigger path | Steps |
|---|---|---|
| `deploy-backend.yml` | `backend/**` | Node 22 · `npm ci` · `tsc --noEmit` · `flyctl deploy --remote-only -a ims-backend-sec02` |
| `deploy-frontend.yml` | `frontend/**` | Node 22 · Vercel CLI · `vercel pull` · `vercel build --prod` · deploy |
| `deploy-rag-service.yml` | `rag-service/**` | Python 3.11 · `pip install` · `py_compile` · `unittest` · ensure Fly app · deploy |

All three also support `workflow_dispatch` for manual runs. Secrets (`FLY_API_TOKEN`,
`FLY_API_TOKEN_RAG`, `VERCEL_TOKEN`, `VERCEL_ORG_ID`, `VERCEL_PROJECT_ID`) live in GitHub
repository secrets; runtime secrets are set with `flyctl secrets set`.

**Production topology:** Vercel (SPA) → Fly.io Singapore (API) → Supabase (Postgres), with the
RAG service as a second Fly.io app.

`03_Deployment/01_Deployment_Package/` ships a self-contained IaC bundle — `fly.toml`,
`Dockerfile.backend`, `Dockerfile.frontend`, `nginx.conf`, `vercel.json`,
`docker-compose.prod.yml`, `db-init.sql` — so the system can be redeployed from scratch by
someone who never saw the repository.

## Testing

| Layer | Tooling | Scope |
|---|---|---|
| Backend unit | Jest 29 + ts-jest | Service logic per module, colocated in `modules/*/__tests__/` |
| Backend API | Supertest 7 | Route-level HTTP assertions |
| Backend integration | Jest `--runInBand` | `warehouse-lifecycle-db.integration.test.ts` and `warehouse-lifecycle-api.test.ts` walk the full Quarantine → Accepted → consumed lifecycle against a real database |
| Frontend | Vitest 4 + Testing Library | 23 component/hook suites, v8 coverage |
| E2E | Playwright | Tagged suites: `@smoke`, `@critical`, `@rbac` |

```bash
npm test                    # all backend suites
npm run test:coverage       # with coverage
npm run test:db-integration # full lifecycle against Postgres
npm run test:api-integration
```

## Repository layout

The structure follows the course's prescribed artefact taxonomy
([reference](https://nhbien.github.io/enterprise-project-artifacts/)):

```
SEC_Team_02_2026/
├── 01_Documents/                        # 11 engineering documents (Vietnamese)
│   ├── 01_Product Requirements Document.md
│   ├── 02_Domain Model.md
│   ├── 03_Prototype.md
│   ├── 04_Product Backlog.md
│   ├── 05_Architecture.md
│   ├── 06_Proof of Concept.md
│   ├── 07_Coding Standards.md
│   ├── 08_Project Management.md
│   ├── 09_System Evaluation and Validation.md
│   ├── 10_Inventory_Management_Workflow.md
│   └── 11_Inventory_Management_Workflow_Detail.md
│                                        # each also has a "_Vibe Coding" AI-assisted variant
├── 02_Source/
│   ├── 01_Source Code/
│   │   ├── backend/                     # Express + TS, 13 modules
│   │   ├── frontend/                    # React 19 + Vite SPA
│   │   ├── rag-service/                 # FastAPI RAG microservice
│   │   ├── ai-service/                  # FastAPI analytics service
│   │   ├── e2e/                         # Playwright harness
│   │   ├── db_schema/                   # db-init.sql · backup.sh · standalone compose
│   │   ├── keycloak/inventory-realm.json
│   │   ├── monitoring/                  # otel · tempo · loki · promtail · prometheus
│   │   │                                # · alertmanager · grafana (dashboards + provisioning)
│   │   ├── docker-compose.yml           # 16-service dev stack
│   │   ├── docker-compose.prod.yml
│   │   ├── DEPLOYMENT.md · DOCKER_SETUP.md
│   │   ├── INTEGRATION_TESTING_GUIDE.md · LABEL_GENERATION_GUIDE.md
│   │   └── STACK_TEST_RESULTS.md
│   ├── 02_Raw Data/
│   └── 03_Compilation Guide.md
├── 03_Deployment/
│   ├── 01_Deployment_Package/           # IaC bundle
│   ├── 02_Deployment Guide.md           # for IT administrators
│   └── 03_User Guide.md                 # for end users
├── .github/workflows/                   # 3 CI/CD pipelines
├── CLAUDE.md                            # AI assistant project context
└── README.md
```

## Running locally

**Prerequisites:** Docker Desktop, Node.js 22+, Python 3.11 (only if running the Python services
outside Docker).

### Full stack with Docker Compose

```bash
git clone https://github.com/xiao-honsu/SEC_Team_02_2026.git
cd "SEC_Team_02_2026/02_Source/01_Source Code"
cp .env.example .env          # then fill in the values
docker compose up -d
```

Brings up all 16 services: Postgres, Redis, Elasticsearch, Keycloak, Ollama, backend, frontend,
ai-service, rag-service, and the seven observability containers.

| Service | URL |
|---|---|
| Frontend | http://localhost:5173 |
| Backend | http://localhost:3000 |
| Keycloak | http://localhost:8080 |
| Grafana | http://localhost:3001 |
| Elasticsearch | http://localhost:9200 |
| AI service | http://localhost:8000 |

### Dev mode (fastest inner loop)

```bash
cd "02_Source/01_Source Code"

# 1. backing services only
docker compose up -d postgres redis elasticsearch keycloak

# 2. backend
cd backend && npm install && npm run dev        # :3000

# 3. frontend
cd ../frontend && npm install && npm run dev    # :5173
```

Redis and Elasticsearch are optional — the backend starts without them.

### Database only

```bash
cd "02_Source/01_Source Code/db_schema"
docker compose up -d                            # PostgreSQL 16 on :5432
psql -U myuser -h localhost -d mydatabase -f db-init.sql
```

Detailed instructions: [`02_Source/03_Compilation Guide.md`](02_Source/03_Compilation%20Guide.md)
and [`03_Deployment/02_Deployment Guide.md`](03_Deployment/02_Deployment%20Guide.md).

## My contributions

I worked primarily on **backend security infrastructure, search, and the labelling subsystem**,
with the matching frontend for QC and labels.

### Security foundation
- **`security/auth.ts`** — JWT authentication middleware verifying Keycloak-issued tokens
- **`security/rbac.ts`** — the role-based access control system: the `UserRole` enum and the
  `PERMISSIONS` resource × action × role matrix that every protected route is checked against

### Search & shared infrastructure
- **`modules/search/`** — Elasticsearch 8 full-text search module (routes, service, types)
- **`shared/elasticsearch/client.ts`** — ES client with optional-dependency semantics
- **`shared/cache/redis.ts`** — Redis client, including the non-blocking reconnect strategy that
  keeps a dead cache from stalling application boot
- **`shared/db/pool.ts`** — Postgres pool configuration

### Admin module
- Full `modules/admin/` implementation — user management and system statistics
  (`admin.routes.ts`, `admin.service.ts`, `admin.types.ts`)
- Unit test suites for both routes and service layers

### Labelling subsystem (full stack)
- **Backend `modules/labels/`** — label template CRUD, field-selection model, and barcode/QR
  generation via `bwip-js` and `qrcode`
- **Frontend** — `LabelsPage`, `LabelTemplateFormModal`, `GenerateLabelModal`, `useLabelsData`
- Entity-bound labels (a label references a real material/lot rather than free text)
- Authored [`LABEL_GENERATION_GUIDE.md`](02_Source/01_Source%20Code/LABEL_GENERATION_GUIDE.md)

### Quality Control (full stack)
- **Backend `modules/qc/`** — QC routes, service and types
- **Frontend** — `QCPage`, `QualityControlDashboard`, `LotQCDetailModal`, `QCTestFormModal`,
  `QCApproveRejectButtons`, `useQCData`
- QC table UX optimisation and label-template edit bug fixes

### Test infrastructure & CI/CD
- Jest configuration and global test setup (`jest.config.ts`, `src/__tests__/setup.ts`)
- `Dockerfile.dev` for the backend
- CI/CD pipeline debugging and validation

### Reports & exports
- `ReportsPage` and `lib/exportUtils.ts` (jsPDF-based PDF export)

### Documentation
- `02_Domain Model.md`, `08_Project Management.md`, `09_System Evaluation and Validation.md`
- `03_Compilation Guide.md`, `02_Deployment Guide.md`, `03_User Guide.md`

Contribution history is preserved in the git log of this repository
(`git log --author="vhduc22@clc.fitus.edu.vn"`).

### Team

| MSSV | Name | Role | GitHub |
|---|---|---|---|
| 21127173 | Nguyễn Thiên Thọ | Team Leader | [@thientho03](https://github.com/thientho03) |
| 22127424 | Nguyễn Phước Minh Trí | Developer | [@NguyenTri251004](https://github.com/NguyenTri251004) |
| 22127316 | Nguyễn Ngô Ngọc Như | Developer | [@ngocnhu100](https://github.com/ngocnhu100) |
| 22127176 | Huỳnh Nguyễn Minh Khang | Developer | [@dodgero11](https://github.com/dodgero11) |
| 22127074 | **Võ Hoàng Đức** | Developer | [@xiao-honsu](https://github.com/xiao-honsu) |
| 18127008 | Lê Mạnh Hoàng | Developer | [@Hoangle1009](https://github.com/Hoangle1009) |

**Course:** Software Engineering Capstone · **Instructor:** Ngô Huy Biên ·
**Semester:** HK2 2025–2026

## Engineering notes

**What I would highlight in an interview**

- The permission matrix in `security/rbac.ts` is the piece of this codebase I am most confident
  in. Encoding `lots.updateStatus` as an action distinct from `lots.update` is what let QC
  release a lot without giving them the ability to edit or delete it — a distinction that would
  have been fragile as scattered role checks.
- Making Redis and Elasticsearch optional dependencies was a boot-time reliability decision that
  paid off directly: production runs on Fly.io with Postgres only, no code changes required.
- HMAC-signing the RAG service was the difference between a demo chatbot and one that could face
  real data.

**What I would change**

- **`.env` and `.env.production` are committed to the repository.** Development defaults are
  arguably acceptable; production credentials are not. These should have been GitHub/Fly secrets
  from day one, with only `.env.example` tracked. *(If you are reusing this repository: rotate
  every credential in those files first.)*
- **Sixteen containers is too many for a six-person student project.** The observability stack is
  genuinely well built, but it doubled local setup time and Ollama alone makes the stack
  unrunnable on a typical laptop. Splitting it into an opt-in Compose profile would have been
  better.
- **`chat` and `rag` exist as separate backend modules** alongside the standalone RAG service,
  which blurs where conversational logic actually lives.
- **The `e2e/` Playwright harness is configured but thin.** The scripts and tag taxonomy
  (`@smoke`, `@critical`, `@rbac`) are in place; the specs never caught up with the module count.
- **Documents are duplicated** as `NN_Name.md` and `NN_Name_Vibe Coding.md`, which makes it
  ambiguous which one is canonical.

---

*Course project — Faculty of Information Technology, VNUHCM University of Science.
Forked for portfolio purposes from
[Inventory-management-SEC/SEC_Team_02_2026](https://github.com/Inventory-management-SEC/SEC_Team_02_2026);
full commit history and original authorship preserved.*
