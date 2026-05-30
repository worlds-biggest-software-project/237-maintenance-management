# CMMS (Maintenance Management) — Phased Development Plan

> Project: 237-maintenance-management · Created: 2026-05-29
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan delivers an AI-native, open-source Computerised Maintenance Management System (CMMS) covering work orders, preventive maintenance, asset hierarchies, MRO inventory, IoT/sensor ingestion, predictive maintenance, mobile field execution, and an MCP server for AI assistants. The target operator profile is mid-market manufacturing, facilities, utilities, and healthcare — organisations excluded from the IBM Maximo / SAP PM / Infor EAM enterprise tier and underserved by mid-market SaaS that gates APIs and AI behind enterprise plans.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language (backend) | **Python 3.12** | The differentiating value of this CMMS is AI/ML on sensor streams (anomaly detection, failure forecasting, spare-parts demand forecasting) and LLM orchestration (NL work-order querying, procedure generation). Python's ecosystem (scikit-learn, statsmodels, river for online anomaly detection, openai/anthropic SDKs, LangChain) makes ML and LLM workflows a first-class citizen, which TypeScript/Go cannot match without bridging. |
| API framework | **FastAPI 0.115+** | Native async (essential for MQTT/OPC UA streaming and webhook fanout), automatic OpenAPI 3.1 spec generation (a stated standard requirement), Pydantic validation against JSON Schema, mature ecosystem for OAuth2/JWT/RBAC. |
| Database | **PostgreSQL 16 + TimescaleDB 2.15 extension** | Standard relational for transactional CMMS data (assets, work orders, parts) plus a time-series hypertable for sensor readings. The data-model-suggestion-4 analysis identified this as the architecture for IoT-first CMMS, and using a single PG instance avoids running a separate TSDB. TimescaleDB provides automatic time-based partitioning, compression (10-20x), continuous aggregates, and retention policies — required for the IoT/SCADA workload the README promises. |
| ORM / DB toolkit | **SQLAlchemy 2.0 (async) + Alembic** | Industry standard, async support, robust migration tooling. Pydantic-SQLAlchemy integration ensures API schemas and DB models stay aligned. |
| Object storage | **MinIO (S3-compatible)** | Attachments (photos, voice clips, equipment manuals, PDF SOPs) need durable object storage. MinIO is self-hostable and S3-API compatible so the same code runs against AWS S3, Cloudflare R2, or MinIO in self-hosted deployments — supporting the README's "self-hosted, cloud, hybrid" deployment matrix. |
| Cache / session store | **Redis 7** | Required for rate limiting, ephemeral JWT denylist, webhook delivery queue dedupe, and sensor-anomaly debounce windows. |
| Task queue | **Celery + Redis broker** | The system has a substantial async workload: webhook dispatch, PM materialisation (rolling work orders from schedules), KPI snapshot rollups, ML inference jobs, ERP sync, LLM calls for NL responses. Celery is mature, distributable, and integrates with Prometheus for monitoring. |
| Message bus for IoT | **EMQX 5 (MQTT broker)** | Required by the README for MQTT-based sensor ingestion. EMQX supports MQTT v5, Sparkplug B, OPC UA gateway, scales horizontally, and is open-source. |
| OPC UA bridge | **asyncua (Python)** | Native Python async OPC UA client/server. Lets the CMMS connect directly to PLCs/SCADA where MQTT is not exposed. |
| Frontend (web) | **Next.js 15 (App Router) + React 19 + TypeScript + Tailwind + shadcn/ui** | Operators need rich dashboards (calendar scheduling, asset hierarchy trees, KPI charts) plus a public-facing marketing/docs site. Next.js handles both with SSR for SEO and React Server Components for dashboard performance. shadcn/ui delivers production-grade primitives. |
| Mobile (field) | **React Native (Expo SDK 52) + react-native-mmkv** | The MVP requires an offline-capable iOS+Android app for technicians. Expo accelerates camera/voice/QR/file APIs; MMKV provides fast offline storage; codebase shares TypeScript types with the web frontend. |
| Offline sync | **Custom delta-sync over REST + idempotency tokens** | Atlas CMMS uses a similar approach; works without coupling to a vendor-specific sync engine and respects the UUID-everywhere design. |
| LLM / AI gateway | **Anthropic Claude (Sonnet 4.5 for chat, Haiku 4.5 for cheap classification) + pluggable provider abstraction** | Claude's tool-use is well-suited to NL→SQL and NL→work-order workflows. The abstraction layer (`backend/services/llm/`) accepts OpenAI/Azure/Bedrock backends for customers with existing contracts. |
| MCP server | **modelcontextprotocol Python SDK** | The standards.md explicitly identifies MCP as a differentiator. Exposing CMMS data as MCP tools allows Claude Desktop, Claude Code, and other MCP clients to query and mutate maintenance data via natural language. |
| ML / anomaly detection | **river (online streaming) + scikit-learn + statsmodels + ONNX Runtime** | `river` provides online learning suitable for live sensor streams (no batch retraining). scikit-learn for classical models (isolation forest, ARIMA forecasting). ONNX Runtime lets users deploy custom-trained models. |
| Auth | **Authlib (OAuth2/OIDC) + python-jose (JWT) + passlib (bcrypt) + SAML via python3-saml** | Covers the full enterprise auth surface from standards.md: OAuth 2.0, JWT bearer tokens, SAML 2.0 SSO, OIDC. |
| Containerisation | **Docker + docker-compose for dev, Helm chart for production K8s** | Mirrors IBM Maximo's Red Hat OpenShift deployment pattern at a fraction of the operational cost. |
| Testing (Python) | **pytest + pytest-asyncio + httpx + testcontainers** | testcontainers spins up real Postgres+TimescaleDB+Redis+MQTT containers per test session for integration tests against real dependencies. |
| Testing (frontend) | **Vitest + React Testing Library + Playwright (E2E)** | Standard React stack; Playwright drives both web and the Next.js admin app for E2E. |
| Code quality | **ruff (lint+format) + mypy (strict) + pre-commit; eslint + prettier for TS** | Ruff replaces black/isort/flake8 in one tool. mypy strict catches integration bugs at type-check time. |
| Package manager | **uv (Python) + pnpm (TS)** | uv is 10–100x faster than pip/poetry for resolving the heavy ML dependency tree; pnpm avoids node_modules duplication across the monorepo. |
| Monorepo tooling | **Turborepo + uv workspaces** | Coordinates frontend, mobile, backend, MCP server, shared TS types, and the marketing site. |
| API documentation | **FastAPI auto-generated OpenAPI 3.1 + Redoc + ReadMe.com export** | Mandated by standards.md (OpenAPI 3.1). |
| Compliance output | **Custom audit log + electronic signature tables (Part 11) + ISO 14224-aligned failure taxonomy seed data** | Required by the README's claim of FDA 21 CFR Part 11, OSHA PSM, GMP, and ISO 55001 support. |
| Observability | **OpenTelemetry SDK → Prometheus (metrics) + Loki (logs) + Grafana** | Self-hostable, open-source, standard. |
| CI/CD | **GitHub Actions matrix (lint, type, unit, integration, e2e, docker-build, helm-lint)** | Standard for open-source projects on GitHub. |

### Project Structure

```
cmms/
├── pyproject.toml                 # uv workspace root
├── package.json                   # pnpm + Turborepo root
├── turbo.json
├── docker-compose.yml             # dev stack: postgres, timescaledb, redis, minio, emqx, mailhog
├── docker-compose.prod.yml
├── Makefile                       # `make dev`, `make test`, `make migrate`
├── .github/workflows/             # ci.yml, release.yml, docker.yml
├── deploy/
│   ├── helm/cmms/                 # Helm chart for production K8s
│   ├── docker/
│   │   ├── api.Dockerfile
│   │   ├── worker.Dockerfile
│   │   ├── ingest.Dockerfile
│   │   ├── mcp.Dockerfile
│   │   └── web.Dockerfile
│   └── terraform/                 # optional examples for AWS/GCP/Azure
├── docs/
│   ├── architecture.md
│   ├── api/openapi.json           # generated
│   └── adr/                       # architecture decision records
├── apps/
│   ├── backend/                   # FastAPI service
│   │   ├── pyproject.toml
│   │   ├── alembic.ini
│   │   ├── alembic/versions/
│   │   ├── src/cmms/
│   │   │   ├── api/
│   │   │   │   ├── v1/
│   │   │   │   │   ├── routers/   # work_orders.py, assets.py, pm.py, parts.py, sensors.py, ...
│   │   │   │   │   ├── schemas/   # Pydantic request/response models
│   │   │   │   │   └── deps.py
│   │   │   │   └── webhooks.py
│   │   │   ├── core/
│   │   │   │   ├── config.py
│   │   │   │   ├── security.py
│   │   │   │   ├── rbac.py
│   │   │   │   └── tenancy.py
│   │   │   ├── db/
│   │   │   │   ├── base.py
│   │   │   │   ├── session.py
│   │   │   │   └── models/        # SQLAlchemy models per aggregate
│   │   │   ├── domain/
│   │   │   │   ├── work_order/
│   │   │   │   ├── asset/
│   │   │   │   ├── pm/
│   │   │   │   ├── inventory/
│   │   │   │   ├── compliance/
│   │   │   │   ├── kpi/
│   │   │   │   └── ai/
│   │   │   ├── services/
│   │   │   │   ├── llm/           # provider abstraction + Claude default
│   │   │   │   ├── storage/       # S3/MinIO
│   │   │   │   ├── email/
│   │   │   │   ├── webhooks/
│   │   │   │   └── erp/           # SAP, Oracle, Dynamics adapters
│   │   │   ├── tasks/             # Celery tasks
│   │   │   ├── main.py
│   │   │   └── cli.py             # typer-based admin CLI
│   │   └── tests/
│   │       ├── unit/
│   │       ├── integration/
│   │       ├── e2e/
│   │       └── fixtures/
│   ├── ingest/                    # MQTT + OPC UA ingestion daemons
│   │   ├── pyproject.toml
│   │   └── src/cmms_ingest/
│   │       ├── mqtt_consumer.py
│   │       ├── opcua_consumer.py
│   │       ├── sparkplug.py
│   │       └── pipeline.py        # normalises → sensor_reading hypertable
│   ├── mcp/                       # MCP server exposing CMMS tools
│   │   ├── pyproject.toml
│   │   └── src/cmms_mcp/
│   │       ├── server.py
│   │       └── tools/             # get_asset.py, create_work_order.py, ...
│   ├── ml/                        # ML pipelines and model registry
│   │   ├── pyproject.toml
│   │   └── src/cmms_ml/
│   │       ├── anomaly/           # isolation forest, river streaming
│   │       ├── forecast/          # parts demand, time-to-failure
│   │       ├── rca/               # root cause analysis
│   │       └── nlp/               # procedure generation, NL→SQL
│   ├── web/                       # Next.js 15 App Router dashboard
│   │   ├── package.json
│   │   ├── next.config.ts
│   │   ├── app/
│   │   │   ├── (auth)/
│   │   │   ├── (app)/dashboard/
│   │   │   ├── (app)/work-orders/
│   │   │   ├── (app)/assets/
│   │   │   ├── (app)/pm/
│   │   │   ├── (app)/parts/
│   │   │   ├── (app)/sensors/
│   │   │   ├── (app)/ai-chat/
│   │   │   └── (app)/settings/
│   │   └── components/
│   ├── mobile/                    # Expo (React Native) field app
│   │   ├── package.json
│   │   ├── app.config.ts
│   │   ├── app/
│   │   │   ├── (tabs)/work-orders.tsx
│   │   │   ├── (tabs)/scan.tsx
│   │   │   ├── (tabs)/asset/[id].tsx
│   │   │   └── _layout.tsx
│   │   └── lib/sync/              # offline delta sync engine
│   └── docs-site/                 # Next.js marketing + docs (optional later)
├── packages/
│   ├── api-client-ts/             # generated TS client from OpenAPI
│   ├── shared-types/              # shared TS types
│   └── ui/                        # shared shadcn primitives
└── scripts/
    ├── seed-iso-taxonomies.py     # ISO 14224 / ISO 13306 reference data
    └── load-demo-data.py
```

---

## Phase 1: Foundation & Project Scaffold

### Purpose
Stand up the monorepo, dev environment, base FastAPI service, database with TimescaleDB, configuration, authentication primitives, and CI. After this phase a developer can clone the repo, run `make dev`, and hit a `/health` endpoint backed by a real Postgres+TimescaleDB+Redis stack. This phase establishes the trunk that every subsequent feature plugs into.

### Tasks

#### 1.1 — Monorepo scaffold and tooling

**What**: Initialise the uv + pnpm + Turborepo monorepo with the directory layout above and a single `make dev` entry point.

**Design**:
- `pyproject.toml` (workspace root) declares `apps/backend`, `apps/ingest`, `apps/mcp`, `apps/ml` as uv workspace members.
- `package.json` (workspace root) declares `apps/web`, `apps/mobile`, `packages/*` as pnpm workspaces.
- `turbo.json` defines `dev`, `build`, `test`, `lint` pipelines with task dependencies.
- `Makefile` targets: `dev` (boots docker-compose + runs FastAPI + worker + web), `test`, `lint`, `format`, `migrate`, `seed`, `clean`.
- `pre-commit` config runs `ruff format`, `ruff check --fix`, `mypy`, `eslint`, `prettier`.
- `.editorconfig`, `.gitignore`, `.gitattributes` for cross-platform consistency.
- `LICENCE` file — Apache 2.0 (permissive, OSS-friendly, patent grant suitable for industrial customers).

**Testing**:
- Unit (shell): `make help` lists every target.
- Integration: `make dev` brings up all containers and `curl localhost:8000/health` returns 200 within 60 seconds (CI step).
- Lint: `ruff check .` and `pnpm lint` both pass on the bare scaffold.

#### 1.2 — Dev stack docker-compose

**What**: A `docker-compose.yml` that boots PostgreSQL+TimescaleDB, Redis, MinIO, EMQX, and MailHog with sane dev defaults and persisted volumes.

**Design**:
Services (image, port, volume):
- `postgres`: `timescale/timescaledb:2.15.0-pg16` exposing 5432, volume `pgdata:/var/lib/postgresql/data`, env `POSTGRES_USER=cmms`, `POSTGRES_PASSWORD=cmms`, `POSTGRES_DB=cmms`. Init script `deploy/docker/init-timescaledb.sql` runs `CREATE EXTENSION IF NOT EXISTS timescaledb;` plus `pg_trgm`, `pgcrypto`.
- `redis`: `redis:7-alpine` exposing 6379.
- `minio`: `minio/minio:latest` exposing 9000/9001, env `MINIO_ROOT_USER=cmms`, `MINIO_ROOT_PASSWORD=cmms-dev-secret`. Buckets created by `minio-init` companion that runs `mc mb cmms-attachments cmms-imports`.
- `emqx`: `emqx/emqx:5.7` exposing 1883 (MQTT), 8083 (MQTT-WS), 18083 (dashboard).
- `mailhog`: `mailhog/mailhog` exposing 1025 (SMTP) and 8025 (UI) for dev email capture.
- Healthchecks on each service so `depends_on: { condition: service_healthy }` works.

**Testing**:
- Integration: `docker compose up -d && docker compose ps` shows all services `(healthy)` within 90s.
- Integration: `psql postgres://cmms:cmms@localhost/cmms -c "SELECT extversion FROM pg_extension WHERE extname='timescaledb'"` returns a version ≥ 2.15.
- Integration: publish to `mqtt://localhost:1883` and the EMQX dashboard reports the message.

#### 1.3 — Configuration system

**What**: Pydantic-Settings-based config loader that reads from environment variables and `.env` files, with strongly-typed sections per concern.

**Design**:
```python
# apps/backend/src/cmms/core/config.py
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import PostgresDsn, RedisDsn, AnyHttpUrl, SecretStr

class DatabaseSettings(BaseSettings):
    url: PostgresDsn
    pool_size: int = 20
    max_overflow: int = 10
    echo: bool = False

class AuthSettings(BaseSettings):
    jwt_secret: SecretStr
    jwt_algorithm: str = "HS256"
    access_token_ttl_seconds: int = 900
    refresh_token_ttl_seconds: int = 30 * 24 * 3600

class StorageSettings(BaseSettings):
    endpoint_url: AnyHttpUrl
    access_key: SecretStr
    secret_key: SecretStr
    bucket_attachments: str = "cmms-attachments"
    region: str = "us-east-1"

class MqttSettings(BaseSettings):
    broker_host: str = "localhost"
    broker_port: int = 1883
    username: str | None = None
    password: SecretStr | None = None
    base_topic: str = "cmms/+/sensors/#"

class LlmSettings(BaseSettings):
    provider: str = "anthropic"  # anthropic | openai | azure | bedrock
    anthropic_api_key: SecretStr | None = None
    openai_api_key: SecretStr | None = None
    default_model: str = "claude-sonnet-4-5"
    cheap_model: str = "claude-haiku-4-5"

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_nested_delimiter="__")
    env: str = "development"
    log_level: str = "INFO"
    cors_origins: list[str] = ["http://localhost:3000"]
    database: DatabaseSettings
    redis: RedisDsn
    auth: AuthSettings
    storage: StorageSettings
    mqtt: MqttSettings
    llm: LlmSettings
```
- Variables flatten to `DATABASE__URL`, `AUTH__JWT_SECRET`, etc.
- `.env.example` committed with placeholders; `.env` in `.gitignore`.
- Settings are singleton via `@lru_cache get_settings()`.

**Testing**:
- Unit: valid `.env` → settings parse with correct types.
- Unit: missing `DATABASE__URL` → ValidationError naming the missing field.
- Unit: nested override (`AUTH__JWT_SECRET=foo`) populates `settings.auth.jwt_secret`.

#### 1.4 — Database session, base model, and Alembic

**What**: SQLAlchemy 2.0 async engine, declarative base, session dependency, Alembic configured against the same engine.

**Design**:
- `cmms/db/base.py` defines `Base` with `MappedAsDataclass`, naming convention for constraints (`ix_%(table_name)s_%(column_0_name)s`, `fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s`, etc.) so migrations are deterministic.
- `cmms/db/session.py` exposes `async_session_maker` and FastAPI dependency `get_db()` yielding `AsyncSession` with auto-rollback on exceptions.
- Base mixin `TimestampedMixin` with `created_at`, `updated_at` (server defaults `now()`, `ON UPDATE` trigger via Alembic).
- Base mixin `TenantScopedMixin` with `tenant_id: Mapped[UUID]` + index — applied to every business model.
- Alembic env imports `Base.metadata` and uses async engine; `alembic upgrade head` runs in `make migrate`.
- First migration `0001_init.sql` enables extensions (`pgcrypto`, `timescaledb`, `pg_trgm`).

**Testing**:
- Unit: `alembic upgrade head && alembic downgrade base` round-trip succeeds on a fresh test DB (uses testcontainers Postgres+Timescale).
- Unit: TenantScopedMixin produces `tenant_id` column with index.

#### 1.5 — Tenant, User, and RBAC core tables

**What**: Initial migration creating `tenant`, `app_user`, `role`, `user_role`, `team`, `team_member`, plus seed data for system roles (`admin`, `maintenance_manager`, `planner`, `technician`, `viewer`).

**Design**: Adopt the schema from `data-model-suggestion-1.md` §"User & RBAC" verbatim with one addition: `app_user.password_hash` (`TEXT NULL` — null when SSO-only) and `app_user.preferred_locale` (`VARCHAR(10) DEFAULT 'en-US'`).

Permission model: each role's `permissions` JSONB column holds a list of strings of form `<resource>:<action>` (e.g., `work_order:create`, `asset:delete`, `*:read`). Permissions evaluated by a central `Permission` enum and `require(perm)` FastAPI dependency.

System roles seeded with deterministic UUIDs (UUID5 from `tenant_id + role_name`) so tests are reproducible.

**Testing**:
- Unit: `Permission.parse("work_order:create")` → enum value.
- Unit: `has_permission(["work_order:*"], "work_order:create")` → True; `has_permission(["asset:read"], "work_order:create")` → False.
- Integration: post-migration query returns all five system roles for a seeded tenant.

#### 1.6 — Authentication endpoints (password + JWT)

**What**: `/auth/login`, `/auth/refresh`, `/auth/logout`, `/auth/me` endpoints implementing OAuth 2.0 Password Grant with JWT access + refresh tokens, plus a `require_user`/`require_perm` dependency.

**Design**:
- Endpoints:
  - `POST /api/v1/auth/login` — body `{ email, password }` → `{ access_token, refresh_token, token_type: "Bearer", expires_in }`.
  - `POST /api/v1/auth/refresh` — body `{ refresh_token }` → new access token.
  - `POST /api/v1/auth/logout` — adds JTI to Redis denylist until natural expiry.
  - `GET /api/v1/auth/me` — returns current user, roles, effective permissions, tenant.
- Tokens: HS256 JWT signed with `auth.jwt_secret`. Access TTL 15m, refresh TTL 30d. Claims: `sub` (user UUID), `tid` (tenant UUID), `roles` (list), `perms` (list), `jti`, `exp`, `iat`.
- Password hashing: `passlib` with `argon2`.
- Dependency `require_user()` extracts and validates token, denylist-checked via Redis.
- Dependency `require_perm("work_order:create")` returns 403 with code `forbidden` if missing.
- Failed logins counted per email in Redis with sliding window (5 fails → 15m lockout) per OWASP API Security A07.

**Testing**:
- Unit: argon2 verify matches; mismatched password returns False.
- Integration: login with correct credentials → 200 + tokens; wrong password → 401; locked account → 423.
- Integration: protected endpoint without token → 401; with valid token → 200; with denylisted token → 401.
- Integration: token claims contain `tid`, `roles`, `perms`, `jti`.

#### 1.7 — Health, metrics, and structured logging

**What**: `/health`, `/health/ready`, `/metrics` endpoints; structlog JSON logging; OpenTelemetry SDK initialised; request-id middleware.

**Design**:
- `/health` — always 200 if process alive.
- `/health/ready` — checks DB, Redis, MinIO connectivity; returns 503 if any unhealthy with reason map.
- `/metrics` — Prometheus exposition via `prometheus-fastapi-instrumentator`.
- structlog renders JSON; binds `request_id`, `tenant_id`, `user_id` to context.
- OTel: traces exported via OTLP (configurable endpoint, off by default).

**Testing**:
- Integration: `/health/ready` returns 200 against the dev stack and 503 with `{"database": "down"}` after killing the Postgres container.
- Integration: log line for a request contains `request_id` matching `X-Request-ID` response header.

#### 1.8 — CI pipeline

**What**: `.github/workflows/ci.yml` running lint, type-check, unit tests, and integration tests (against testcontainers) on every push.

**Design**:
- Matrix: `python-version: ["3.12"]`, `node-version: ["20"]`.
- Jobs: `lint-py` (ruff, mypy), `lint-ts` (eslint, tsc), `test-py-unit`, `test-py-integration` (services: postgres+timescale via testcontainers), `test-ts-unit`, `docker-build` (api, worker, ingest, mcp, web), `helm-lint`.
- Cache uv venv and pnpm store keyed on lockfile.
- Required checks gate `main`.

**Testing**:
- Smoke: push a PR; all jobs green within 10 minutes.
- Negative: a deliberate `print(` in code fails the lint job.

---

## Phase 2: Core Domain — Assets, Locations, Failure Taxonomy

### Purpose
Implement the foundational catalogue: organisational sites, location hierarchy, equipment classes (ISO 14224), assets, and the failure taxonomy lookup tables (ISO 14224 / ISO 13306). Nothing in the CMMS works without something to maintain. This phase ends with a populated reference dataset, CRUD APIs for assets, and the ability to import/export asset registers — the minimum viable backbone for everything that follows.

### Tasks

#### 2.1 — Site, location, equipment_class, asset tables

**What**: Migration and SQLAlchemy models for `site`, `location`, `equipment_class`, and `asset` following data-model-suggestion-1 §"Location & Asset Hierarchy".

**Design**: Use the SQL from data-model-suggestion-1 verbatim, with three additions to `asset`:
- `qr_uuid UUID UNIQUE NOT NULL DEFAULT gen_random_uuid()` — printed on QR labels; immutable, separate from `asset_tag` (which may be renamed).
- `attributes JSONB NOT NULL DEFAULT '{}'` — for industry-specific fields per data-model-suggestion-3's hybrid approach.
- `tsv tsvector GENERATED ALWAYS AS (...) STORED` + GIN index — full-text search over name, description, asset_tag, manufacturer, model, serial_number.

Location uses materialised path (`path TEXT`) maintained by application code (Python helper `recompute_path(location)` invoked on `parent_id` change).

Equipment class uses adjacency list (`parent_class_id`) — only ~200 entries total so no materialised path needed.

**Testing**:
- Unit: `recompute_path` builds `/site-a/building-1/floor-2/zone-x` correctly across reparents.
- Unit: asset full-text search matches partial words via `tsv @@ plainto_tsquery`.
- Integration: round-trip create→read→update→delete via SQLAlchemy.

#### 2.2 — ISO 14224 / ISO 13306 taxonomy tables and seed

**What**: Migration creating `failure_mode`, `failure_cause`, `failure_mechanism`, `detection_method`, `maintenance_action_type`, `work_order_priority` tables. Seed script populates ISO 14224-conformant reference data plus default priorities.

**Design**: Schema from data-model-suggestion-1 §"Failure Taxonomy". Seed script `scripts/seed-iso-taxonomies.py`:
- Reads `seeds/iso14224-failure-modes.yaml`, `seeds/iso14224-mechanisms.yaml`, `seeds/iso13306-action-types.yaml`.
- Idempotent: uses `INSERT ... ON CONFLICT (code) DO UPDATE`.
- Loads at least: 35 failure modes (FTS, STP, ELU, AIR, BRD, UST, LCP, OHE, NOI, ERO, FOF, VBR, …), 18 failure mechanisms (COR, ERO, WEA, FAT, BLO, OVH, CRK, SPL, …), 10 detection methods, 14 action types, 4 default priorities (Emergency/SLA 4h, High/24h, Medium/72h, Low/168h).
- Run automatically on first boot of the API container if `tenant` table is empty.

**Testing**:
- Unit: parse YAML seeds → emits at least the expected counts.
- Integration: run seed twice → no duplicates, no errors.
- Fixture: `tests/fixtures/taxonomy.json` snapshot test.

#### 2.3 — Asset CRUD API

**What**: REST endpoints for asset CRUD with filtering, pagination, FTS, and tree expansion.

**Design**:
- `GET /api/v1/assets` — filter by `site_id`, `location_id`, `status`, `criticality`, `equipment_class_id`, `q` (FTS), `parent_asset_id`; cursor pagination (`cursor`, `limit`), `?include=children,location,equipment_class`.
- `POST /api/v1/assets` — body Pydantic `AssetCreate`; returns `AssetRead`.
- `GET /api/v1/assets/{id}` — `?include=` for related expansions.
- `PATCH /api/v1/assets/{id}` — partial update; emits `audit_log` entries per changed field.
- `DELETE /api/v1/assets/{id}` — soft delete (sets `status='decommissioned'`); hard delete blocked if open work orders exist.
- `GET /api/v1/assets/{id}/tree` — returns asset + descendants.
- `GET /api/v1/assets/{id}/qr` — returns SVG QR code encoding `qr_uuid`.
- Schemas:
```python
class AssetCreate(BaseModel):
    site_id: UUID
    location_id: UUID | None = None
    parent_asset_id: UUID | None = None
    equipment_class_id: UUID | None = None
    asset_tag: str = Field(min_length=1, max_length=100)
    name: str
    description: str | None = None
    serial_number: str | None = None
    manufacturer: str | None = None
    model: str | None = None
    criticality: Literal["critical", "high", "medium", "low"] = "medium"
    install_date: date | None = None
    purchase_cost: Decimal | None = None
    currency_code: str = Field("USD", pattern=r"^[A-Z]{3}$")
    has_meter: bool = False
    meter_unit: str | None = None
    attributes: dict[str, Any] = Field(default_factory=dict)
```
- Permissions: `asset:read`, `asset:create`, `asset:update`, `asset:delete`.

**Testing**:
- Unit: AssetCreate validates ISO 4217 currency codes.
- Integration: cursor pagination — request N items twice, ensure no overlap or skip.
- Integration: tree endpoint returns proper depth-first ordering with `depth` field.
- Integration: deleting an asset with an open WO → 409 with code `asset_in_use`.
- Integration: PATCH on `criticality` writes one audit_log row with old/new values.

#### 2.4 — Location, equipment-class CRUD

**What**: Sibling endpoints for `location` and `equipment_class`, plus a `GET /locations/tree?site_id=` endpoint.

**Design**: Same patterns as 2.3. Equipment-class endpoints expose the seeded ISO 14224 hierarchy plus tenant-defined custom classes.

**Testing**: As per 2.3; additionally:
- Integration: reparenting a location updates its path and all descendants' paths in one transaction.

#### 2.5 — Asset register CSV/Excel import + export

**What**: Endpoints to bulk import assets from CSV/XLSX and export the asset register.

**Design**:
- `POST /api/v1/assets/import` — multipart `file` (CSV/XLSX) → returns `import_job_id`; processing runs in a Celery task.
- `GET /api/v1/imports/{id}` — status, rows processed, errors per row.
- `GET /api/v1/assets/export.csv` and `.xlsx` — streams entire register filtered by query params.
- Importer uses `pandas` for parsing, validates each row against `AssetCreate` schema, batches inserts of 500 rows, captures per-row errors into `import_row_error` table.
- Idempotency: import file SHA-256 stored; same hash re-uploaded within 24h returns existing job.

**Testing**:
- Fixture: `tests/fixtures/assets-100-rows.xlsx` import yields 100 assets, 0 errors.
- Fixture: `tests/fixtures/assets-with-bad-rows.csv` (mixed valid/invalid) reports the exact error rows.
- Integration: export then re-import yields identical row count.

---

## Phase 3: Work Orders & Preventive Maintenance

### Purpose
Deliver the heart of the CMMS: work-order lifecycle and preventive-maintenance scheduling. After this phase a technician can be assigned a work order, fill in checklist tasks, log labour and parts, attach a photo, and close it; a planner can define PM schedules that automatically materialise work orders ahead of due dates.

### Tasks

#### 3.1 — Work order tables and lifecycle state machine

**What**: Migration for `work_order`, `work_order_task`, `work_order_labour`, `work_order_comment`, `attachment`; explicit state-machine for WO status transitions.

**Design**: SQL from data-model-suggestion-1 §"Work Order Management" verbatim. Add columns:
- `work_order.source` (`VARCHAR(30)` — `manual`, `pm`, `sensor_anomaly`, `inspection_failed`, `erp_sync`, `mcp_ai`)
- `work_order.ai_generated BOOLEAN DEFAULT FALSE`
- `work_order.ai_metadata JSONB` — sensor reading IDs, anomaly score, LLM trace ID

State machine (encoded in Python `WorkOrderStateMachine` using `transitions` library):
```
requested → approved → planned → scheduled → in_progress → completed → closed
                     ↘            ↓             ↑↓
                       cancelled  on_hold  (back to in_progress)
```
Each transition enforces:
- `approved`: requires `work_order:approve` perm.
- `in_progress`: stamps `actual_start = now()`.
- `completed`: stamps `actual_end = now()`, `actual_hours` auto-computed from labour log, all required tasks must be `completed` or `skipped`.
- `closed`: requires manager perm; freezes editing.
Invalid transitions → 409 with code `invalid_status_transition`.

**Testing**:
- Unit: WO state machine — every valid transition succeeds; every invalid one raises `InvalidTransition`.
- Unit: `completed` blocked when a required task is `pending`.
- Integration: status PATCH from `requested` → `in_progress` (skipping approved) → 409.

#### 3.2 — Work order CRUD API

**What**: Full REST surface for work orders, tasks, labour, comments, attachments.

**Design**:
- `GET /api/v1/work-orders` — filters: `status`, `wo_type`, `priority`, `assigned_to`, `assigned_team`, `asset_id`, `site_id`, `due_before`, `due_after`, `q` (FTS over title/description), `source`; sort by `due_date|created_at|priority`.
- `POST /api/v1/work-orders` — body validated by `WorkOrderCreate`; auto-generates `wo_number` via `nextval('wo_number_seq')` formatted `WO-{tenant_slug}-{site_code}-{NNNNNN}`.
- `GET/PATCH/DELETE /api/v1/work-orders/{id}`.
- `POST /api/v1/work-orders/{id}/transition` — body `{ action: "approve"|"schedule"|"start"|"hold"|"complete"|"close"|"cancel", reason?: str, scheduled_start?, scheduled_end? }`.
- `GET/POST/PATCH/DELETE /api/v1/work-orders/{id}/tasks/{task_id}` — manage checklist.
- `POST /api/v1/work-orders/{id}/labour` — log labour entries.
- `POST /api/v1/work-orders/{id}/comments` — add comment, emits webhook.
- `POST /api/v1/work-orders/{id}/attachments` — multipart upload, stores to MinIO, persists row in `attachment`.

**Testing**:
- Integration: full lifecycle create → schedule → start → complete; assertions on timestamps and audit_log entries at each step.
- Integration: complete with unfinished required task → 409.
- Integration: attachment upload writes object to MinIO and returns presigned download URL.
- Integration: list endpoint with combined filters returns correct subset.

#### 3.3 — Preventive maintenance tables

**What**: Migration for `pm_schedule`, `pm_task_template`, and supporting tables.

**Design**: SQL from data-model-suggestion-1 §"Preventive Maintenance" with additions:
- `pm_schedule.materialisation_horizon_days` — how far ahead WOs are generated (default 30).
- `pm_schedule.skip_if_open` — boolean, skips generating a new WO if a previous PM-generated WO for this schedule is still open.
- `pm_schedule.timezone` — for accurate date math across sites.

**Testing**:
- Integration: create PM with daily frequency, materialise 30 days → 30 WOs created.

#### 3.4 — PM scheduling engine

**What**: Service that computes `next_due_date` and materialises work orders ahead of due time.

**Design**:
- Module `cmms.domain.pm.scheduler`:
  - `compute_next_due(schedule: PmSchedule, from_date: date) -> date` — pure function supporting time-based (`frequency_value` × `frequency_unit`), meter-based (using latest meter reading), event-based (no automatic next date — triggered externally), condition-based (no automatic next date — sensor-triggered).
  - `materialise_due_work_orders(now: datetime, horizon: timedelta) -> list[UUID]` — finds active PMs whose `next_due_date <= now + horizon` and no open PM-generated WO already exists, then creates WOs by cloning `pm_task_template` rows into `work_order_task`.
- Celery beat task `materialise_pms` runs every 15 minutes calling `materialise_due_work_orders(now, 30 days)`.
- On WO `closed`, scheduler updates `pm_schedule.last_completed` and recomputes `next_due_date`.

**Testing**:
- Unit: compute_next_due — monthly schedule starting Jan 31 → next is Feb 28/29 (calendar-safe).
- Unit: meter-based — current meter 1230 hours, interval 500 → next at 1500, then 2000.
- Integration: `skip_if_open=True` prevents double-issue.
- Integration: closing the PM-generated WO triggers next materialisation correctly.
- Fixture-based: 50 PMs spanning all trigger types → expected WOs materialised over a simulated 90-day window.

#### 3.5 — Calendar / Gantt API endpoint

**What**: `GET /api/v1/calendar/work-orders?from=&to=&site_id=&assigned_to=` returning WOs grouped per day for drag-and-drop calendar UIs.

**Design**:
- Returns
```json
{
  "from": "2026-06-01", "to": "2026-06-30",
  "items": [
    { "id": "uuid", "wo_number": "...", "title": "...", "asset_id": "...",
      "scheduled_start": "...", "scheduled_end": "...", "status": "...",
      "priority": "high", "assigned_to_name": "...", "duration_hours": 2.5 }
  ]
}
```
- `PATCH /api/v1/work-orders/{id}` accepts `scheduled_start`/`scheduled_end` for drag updates.

**Testing**:
- Integration: items returned only intersect the window.
- Integration: drag update emits `workorder.updated` webhook with both old and new schedule.

---

## Phase 4: Spare Parts, Inventory & Procurement

### Purpose
Add the MRO inventory backbone: parts catalogue, storerooms, on-hand quantities, reorder thresholds, work-order parts usage (consumption), and a basic purchase-order workflow. After this phase a closed work order can consume parts from inventory and trigger a reorder alert when stock drops below threshold.

### Tasks

#### 4.1 — Parts and inventory tables

**What**: Migration for `part_category`, `part`, `storeroom`, `part_inventory`, `work_order_part`, `asset_part_bom`.

**Design**: SQL from data-model-suggestion-1 §"Spare Parts & Inventory" verbatim. Add `part.image_url`, `part.preferred_vendor_id` (FK introduced in 4.4).

**Testing**:
- Integration: CRUD round-trip per table.
- Integration: `part_inventory` unique constraint on `(part_id, storeroom_id)` enforced.

#### 4.2 — Parts CRUD and inventory adjustment

**What**: REST endpoints for parts, storerooms, inventory; an explicit `adjust` endpoint that produces an `inventory_transaction` audit row.

**Design**:
- `POST /api/v1/inventory/adjust` — body `{ part_id, storeroom_id, delta, reason: "stock_take"|"damage"|"transfer"|"return", reference?: { type: "work_order"|"po", id: UUID } }`.
- Creates `inventory_transaction(id, part_id, storeroom_id, delta, balance_after, reason, reference_type, reference_id, user_id, created_at)` row and updates `part_inventory.quantity_on_hand` atomically inside a single transaction with `SELECT ... FOR UPDATE`.
- Reorder trigger: after every decrement, if `quantity_on_hand <= reorder_point`, emits domain event `PartLowStock` (consumed by webhooks + notifications).

**Testing**:
- Unit: concurrent adjustments (simulated) preserve balance correctness (use a real Postgres in testcontainers with two sessions).
- Integration: dropping below reorder_point emits the event exactly once.

#### 4.3 — Work-order parts consumption

**What**: `POST /api/v1/work-orders/{id}/parts` issues parts from a storeroom against the WO and decrements inventory.

**Design**:
- Body `{ part_id, storeroom_id, quantity_used, unit_cost? }`.
- Atomic: creates `work_order_part` row, calls `inventory_adjust(delta=-quantity_used, reason='work_order', reference={wo})`, updates `work_order.actual_cost`.
- Reversal endpoint `DELETE /api/v1/work-orders/{id}/parts/{wop_id}` returns parts and updates cost.

**Testing**:
- Integration: full cycle — issue 3 parts, then reverse one → inventory balance and WO cost match expectations.
- Integration: issuing more than on-hand → 409 with code `insufficient_stock`.

#### 4.4 — Vendor and purchase-order workflow

**What**: Tables and endpoints from data-model-suggestion-1 §"Vendor & Purchase Orders" plus a "receive" workflow.

**Design**:
- Endpoints: CRUD on `vendor`, `purchase-order`, `purchase-order-line`; `POST /api/v1/purchase-orders/{id}/transition` for status changes; `POST /api/v1/purchase-orders/{id}/receive` with body `[{ line_id, quantity_received, storeroom_id }]` that calls `inventory_adjust(+quantity)` per line and updates `quantity_received`.
- PO status machine: `draft → submitted → approved → ordered → partially_received → received` (or `cancelled` from any non-final state).
- Automatic PO suggestion service: when `PartLowStock` fires, creates a `draft` PO using `preferred_vendor_id` and `reorder_qty` for each affected part (planner reviews and submits).

**Testing**:
- Integration: receive partial → status flips to `partially_received`; receive remainder → `received`.
- Integration: low-stock event creates suggestion PO with right line items.

---

## Phase 5: IoT Ingestion, Sensors, and Anomaly Detection

### Purpose
Connect the CMMS to OT/IoT data sources. After this phase the system can ingest sensor readings via MQTT and OPC UA, store them efficiently in a TimescaleDB hypertable, evaluate threshold rules and ML anomalies, and automatically generate work orders when conditions warrant. This is the project's single largest differentiator versus open-source incumbents (Atlas CMMS, openMAINT) and the cheapest entry to predictive maintenance versus IBM Maximo Monitor and eMaint AI.

### Tasks

#### 5.1 — Sensor device registry

**What**: Tables `sensor_device`, `sensor_threshold`, and association to assets.

**Design**: SQL from data-model-suggestion-1 §"IoT" plus:
- `sensor_device.measurement_unit` (`VARCHAR(50) NOT NULL`) — e.g., `degC`, `psi`, `mm/s`.
- `sensor_device.sampling_interval_seconds INTEGER` — declared rate.
- `sensor_device.wot_description JSONB` — optional W3C WoT Thing Description for self-describing devices.
- `sensor_device.tags JSONB DEFAULT '{}'` — for grouping (`{ "line": "A1", "criticality": "high" }`).
- Endpoints: full CRUD on `/api/v1/sensors/devices`, `/api/v1/sensors/devices/{id}/thresholds`.

**Testing**:
- Integration: CRUD round-trip.
- Integration: PATCH device → emits `sensor.updated` webhook.

#### 5.2 — Sensor reading hypertable

**What**: TimescaleDB hypertable for high-volume sensor readings with compression and retention policies.

**Design**:
```sql
CREATE TABLE sensor_reading (
    sensor_device_id UUID NOT NULL,
    reading_time   TIMESTAMPTZ NOT NULL,
    value          DOUBLE PRECISION NOT NULL,
    quality        SMALLINT NOT NULL DEFAULT 0,  -- 0 good, 1 suspect, 2 bad
    tenant_id      UUID NOT NULL,
    PRIMARY KEY (sensor_device_id, reading_time)
);
SELECT create_hypertable('sensor_reading', 'reading_time', chunk_time_interval => INTERVAL '1 day');
ALTER TABLE sensor_reading SET (timescaledb.compress, timescaledb.compress_segmentby = 'sensor_device_id, tenant_id');
SELECT add_compression_policy('sensor_reading', INTERVAL '7 days');
SELECT add_retention_policy('sensor_reading', INTERVAL '365 days');  -- tenant-overridable
CREATE INDEX ON sensor_reading (tenant_id, reading_time DESC);

CREATE MATERIALIZED VIEW sensor_reading_hourly
WITH (timescaledb.continuous) AS
SELECT sensor_device_id, tenant_id,
       time_bucket('1 hour', reading_time) AS bucket,
       avg(value) AS avg_value,
       min(value) AS min_value,
       max(value) AS max_value,
       count(*)   AS sample_count
FROM sensor_reading
GROUP BY sensor_device_id, tenant_id, bucket;
SELECT add_continuous_aggregate_policy('sensor_reading_hourly',
    start_offset => INTERVAL '2 days', end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '15 minutes');
```
- Retention policy per-tenant overridable via `sensor_retention_policy(tenant_id, days)` table consulted by a Celery job that adjusts policies.
- API: `GET /api/v1/sensors/devices/{id}/readings?from=&to=&bucket=raw|1m|1h|1d&agg=avg|min|max&limit=` — when `bucket=raw` queries the hypertable; otherwise queries the continuous aggregate.

**Testing**:
- Integration: insert 10k readings, query with `bucket=1h&agg=avg` returns hourly aggregates.
- Integration: chunk after 7 days is compressed (verify via `timescaledb_information.compressed_chunk_stats`).
- Performance: 100k inserts in batches of 1000 complete in <10s on dev hardware.

#### 5.3 — MQTT ingestion daemon

**What**: `apps/ingest/mqtt_consumer.py` — async MQTT client (aiomqtt) consuming from configured base topic and writing batched readings to the hypertable.

**Design**:
- Subscribes to `cmms/{tenant_slug}/sensors/{device_id}/+` for raw readings and `spBv1.0/+/+/+/+` for Sparkplug B.
- Two payload formats accepted:
  1. JSON `{ "ts": "2026-05-29T01:23:45Z", "value": 72.4, "unit": "degC", "quality": "good" }`
  2. Sparkplug B protobuf (decoded with `sparkplug_b_pb2`)
- Decoded readings batched in-memory; flushed every 1s or 1000 readings, whichever first, via `COPY` into hypertable for throughput.
- Failed batches DLQ'd to Redis stream `cmms:ingest:dlq`.
- Device-not-found readings auto-create a `sensor_device` row in `pending_registration` status (administrators review and confirm) — supports zero-config sensor onboarding.

**Testing**:
- Integration: publish 1000 messages via paho-mqtt to local EMQX → 1000 rows appear in `sensor_reading` within 5s.
- Integration: Sparkplug B encoded payload decoded and inserted correctly.
- Integration: malformed payload routes to DLQ stream with reason.

#### 5.4 — OPC UA bridge

**What**: `apps/ingest/opcua_consumer.py` — connects to a configured OPC UA server, subscribes to monitored items, writes readings.

**Design**:
- Configuration table `opcua_endpoint(id, tenant_id, name, url, security_mode, security_policy, username, password_encrypted, is_active)`.
- For each endpoint, daemon establishes connection, subscribes to nodes mapped via `opcua_node_mapping(endpoint_id, node_id, sensor_device_id)`.
- Reading writes go through the same batched pipeline as MQTT.
- Auto-discovery endpoint `POST /api/v1/sensors/opcua/{endpoint_id}/browse` returns the OPC UA address space tree.

**Testing**:
- Integration (mocked): use `asyncua` `Server` class to run an in-process mock OPC UA server in the test, then connect the consumer and assert readings flow.

#### 5.5 — Threshold rule engine and auto-WO creation

**What**: A streaming evaluator that compares each incoming reading against active thresholds and emits domain events / creates WOs.

**Design**:
- Module `cmms.domain.iot.thresholds`:
  - Loaded thresholds cached in-process and refreshed every 60s; cache invalidated on threshold CRUD via Redis pub/sub.
  - For each reading, evaluate `warning_high`, `alarm_high`, `warning_low`, `alarm_low` thresholds. Use a debounce: an alarm fires only when N consecutive readings exceed threshold within window W (defaults N=3, W=5 minutes, configurable per threshold).
  - On fire: emit `ThresholdBreached` event; if `auto_create_wo=True` and no open auto-generated WO exists for `(device, threshold)`, create a WO with:
    - `wo_type = 'condition_based'`
    - `source = 'sensor_anomaly'`
    - `ai_generated = false`
    - `ai_metadata = { device_id, threshold_id, reading_ids: [...], breach_value, threshold_value }`
    - `priority` set from `threshold.wo_priority_id`
    - Title autogenerated from a template `"{{ device.name }} {{ threshold.type }} breach ({{ value }} {{ unit }})"`.

**Testing**:
- Unit: debounce — 2 of 3 readings above threshold → no fire; 3 consecutive → fire once.
- Integration: publish a synthetic stream that breaches → WO created with correct ai_metadata.
- Integration: subsequent breaches while WO open → no duplicate WO.

#### 5.6 — ML anomaly detection (online streaming)

**What**: ML pipeline using `river` for online anomaly detection on the per-device reading stream, plus a classical isolation forest for back-filled training.

**Design**:
- Module `cmms.ml.anomaly`:
  - `OnlineAnomalyDetector` per device: wraps `river.anomaly.HalfSpaceTrees` (drift-resistant, low memory) with running mean/std normalisation.
  - State persisted to Redis hash `cmms:ml:anomaly:{device_id}` every minute; restored on restart.
  - Score threshold per device (default 0.95) trips a `MlAnomalyDetected` event.
  - Optional batch retraining nightly: pulls last 30 days from hypertable, trains `sklearn.ensemble.IsolationForest`, saves to MinIO `models/{device_id}/isolation_forest_{version}.onnx` (via skl2onnx).
- Detection consumes from the same ingestion pipeline as thresholds (sidecar in the worker).
- A new threshold type `ml_anomaly` allows operators to wire ML detections into work-order auto-creation by re-using the existing pipeline from 5.5.

**Testing**:
- Unit: feed synthetic normal stream then injected outlier → outlier scored above threshold.
- Integration: 30-day fixture replay → anomalies detected match labelled anomalies with ≥80% recall (test fixture).

---

## Phase 6: Mobile Field App & Offline Sync

### Purpose
Deliver the iOS+Android technician app required by the MVP. Technicians must work in places without connectivity (basements, factory floors, ships, remote sites) and reliably sync when reconnected. After this phase a technician can open the app offline, view assigned work orders, complete tasks with photos and voice clips, log labour, scan QR codes, and have everything sync atomically on reconnection.

### Tasks

#### 6.1 — Expo project scaffold

**What**: `apps/mobile` Expo Router app with shared types from `packages/shared-types` and the generated `packages/api-client-ts`.

**Design**:
- Tabs: Work Orders, Scan, Assets, Settings.
- Auth flow reuses `/auth/login`; tokens stored in `expo-secure-store`.
- React Query for server state; MMKV for local persistence.
- `react-native-vision-camera` for camera; `expo-av` for voice clips; `react-native-camera-kit` for QR/barcode.
- App icon, splash, and EAS Build profiles for iOS/Android prebuilt.

**Testing**:
- E2E (Detox or Maestro): smoke test boots app, logs in, lists work orders, opens detail.

#### 6.2 — Offline-first data layer

**What**: A local SQLite cache via `op-sqlite`, plus a delta-sync engine.

**Design**:
- Local tables mirror server core: `work_order`, `work_order_task`, `asset`, `part`, `attachment_pending`.
- Pull strategy: `GET /api/v1/sync/pull?since=<lamport_clock>` returns all rows updated since the last clock for the user's scope (assigned WOs, their assets and parts BOMs); responses include a new `lamport_clock`.
- Push strategy: client buffers mutations as a queue `mutation_queue(id, op, entity_type, entity_id, payload, created_at, attempts)`; on connectivity, posts to `POST /api/v1/sync/push` body `{ mutations: [...] }`. Each mutation carries a client-generated UUID as idempotency key. Server returns per-mutation result; conflicts return `409` with the server's current version; UI presents conflict resolution.
- Conflict resolution: last-writer-wins per-field for scalar updates; for status transitions the server's state-machine validates and may reject (e.g., trying to start an already-completed WO).
- Photos and voice clips: uploaded as binary multipart to the same `attachment` endpoint; the local row references the local file path until upload completes, then is replaced with the server URL.

**Testing**:
- Unit: mutation queue persists across app restarts.
- Integration: simulate airplane mode → make 10 mutations → reconnect → server reflects all 10.
- Integration: concurrent mutation from web triggers conflict → mobile shows resolution UI.

#### 6.3 — Work order execution UI

**What**: Detail screen for a work order with task checklist, labour log, parts add, photo/voice capture, and signature pad.

**Design**:
- Task checklist: list with checkbox + measurement input when `measurement_unit` is set; out-of-range measurements highlighted red.
- Labour entry: punch in/out timer using device clock; offline-safe.
- Photo: tap to capture or pick from library; thumbnails shown locally.
- Voice clip: tap-and-hold to record up to 5 minutes; stored as M4A.
- Signature pad: `react-native-signature-canvas`; result POSTed to `/work-orders/{id}/sign` (Phase 8 wires this to electronic signatures for FDA Part 11).
- Status transitions surfaced as bottom-sheet actions.

**Testing**:
- Component tests: checklist completion updates progress bar.
- E2E: complete a WO end-to-end including a measurement and a photo while offline → sync → server reflects all data.

#### 6.4 — QR / barcode scanner

**What**: Scan screen that resolves `qr_uuid` to the corresponding asset and offers quick actions.

**Design**:
- Decodes QR/barcode; lookup against local asset cache first, then API.
- On match: navigates to asset detail with "Report issue (create WO)" and "Start nearest PM" actions.
- Generates printable QR labels from web admin (Phase 7).

**Testing**:
- E2E: scan a known QR code → asset screen opens; unknown code shows fallback dialog.

---

## Phase 7: Web Dashboard (Next.js)

### Purpose
Ship the planner/manager web experience: dashboards, KPIs, work-order management, calendar scheduling, asset register browse, PM management, parts management, sensor health, and admin/settings. After this phase a maintenance manager can run an entire shift from the web UI without touching the API directly.

### Tasks

#### 7.1 — Next.js scaffold, auth, layout

**What**: `apps/web` Next.js 15 App Router project with shadcn/ui, auth wired to `/auth/login`, app shell with sidebar navigation.

**Design**:
- Route groups: `(auth)`, `(app)`.
- Auth: cookies for refresh token (HttpOnly, Secure, SameSite=Lax), access token in memory, automatic refresh interceptor in the API client.
- Layout: collapsible sidebar (Dashboard, Work Orders, Calendar, Assets, PMs, Parts, Sensors, AI Chat, Reports, Settings); top bar with tenant switcher, search, profile.
- Theme: dark/light via shadcn theme provider.

**Testing**:
- E2E (Playwright): login, navigate to each sidebar item, sign out.

#### 7.2 — Dashboard with KPI cards and charts

**What**: Home page showing MTTR, MTBF, PM compliance %, downtime cost, open WOs by priority, overdue WOs, and a 30-day downtime trend chart.

**Design**:
- Pulls from `GET /api/v1/kpi/summary?site_id=&period=30d` (added in Phase 8).
- Cards built with shadcn `Card`; charts with `recharts`.
- React Query with `staleTime: 60s`.

**Testing**:
- E2E: dashboard loads under 2s on seeded demo data; values match what the API returned.

#### 7.3 — Work order management screens

**What**: List + filter + bulk actions + detail/edit + status transition + comments.

**Design**:
- List uses `@tanstack/react-table` with column visibility persisted; filter chips for status, priority, type.
- Detail page mirrors mobile (tasks, labour, parts, attachments, comments, audit history).
- Bulk actions: assign, change priority, cancel, export selected.

**Testing**:
- E2E: create WO via UI → appears in list → transition through approved/scheduled/in-progress/completed → audit history shows each step.

#### 7.4 — Calendar / Gantt scheduling

**What**: Drag-and-drop weekly and monthly calendar view of scheduled work orders with technician swimlanes.

**Design**:
- Use `react-big-calendar` (week, month) with custom resource view (technician swimlanes).
- Drag emits PATCH to `/work-orders/{id}` updating `scheduled_start/end`.
- Conflicts (assignee already scheduled in overlapping slot) shown as red outline.

**Testing**:
- E2E: drag a WO from Monday to Wednesday → server reflects new schedule.

#### 7.5 — Asset, PM, Parts, Sensor screens

**What**: Standard CRUD UIs for each remaining catalogue plus dedicated visualisations:
- Asset tree explorer (left tree, right detail panel).
- PM list with "next due" and "compliance %" columns; PM detail shows materialised WOs.
- Parts low-stock view defaulting to filtered low-stock list.
- Sensors view with live readings sparkline (websocket subscription) and threshold management.

**Design**:
- Live readings: server endpoint `WS /api/v1/sensors/devices/{id}/stream` that pushes the last reading to subscribed clients (backed by Redis pub/sub from the ingestion pipeline).
- Asset tree uses virtualised tree from `@tanstack/react-virtual` for very large hierarchies.

**Testing**:
- E2E: scenarios per screen as above.
- E2E: open a sensor detail page, publish a reading via test MQTT client, see the sparkline update within 2s.

#### 7.6 — QR-label print and bulk import UIs

**What**: A modal for printing QR labels for selected assets (PDF) and an import dialog for asset/part CSV+XLSX.

**Design**:
- Print: client posts selection → server returns PDF (built with `reportlab`) containing N-per-page labels (configurable layout).
- Import: drag-and-drop a file → calls `POST /assets/import` → polls job status.

**Testing**:
- E2E: select 12 assets, print labels, downloaded PDF has 12 pages of labels.

---

## Phase 8: Compliance, Audit Trail, Electronic Signatures, KPIs

### Purpose
Make the system safe for regulated industries — pharma, food, chemicals, healthcare, utilities. After this phase the CMMS produces compliant audit trails (ISO 55001, FDA 21 CFR Part 11), captures electronic signatures with intent, runs inspection workflows (OSHA PSM mechanical integrity), and exposes reliability KPIs as a queryable API.

### Tasks

#### 8.1 — Universal audit log

**What**: Every mutation to a business entity (assets, WOs, PMs, parts, etc.) writes to `audit_log` with diffable old/new values.

**Design**:
- Implemented via SQLAlchemy event listeners on session flush: for every `INSERT`/`UPDATE`/`DELETE` of a `TenantScopedMixin` model, an `audit_log` row is generated.
- Diff is per-changed-field, not per-row, mirroring the schema from data-model-suggestion-1.
- Captured: `entity_type`, `entity_id`, `action`, `field_name`, `old_value`, `new_value`, `user_id` (from request context), `ip_address`, `created_at`.
- Read API: `GET /api/v1/audit?entity_type=&entity_id=&user_id=&from=&to=&limit=`.

**Testing**:
- Integration: PATCH an asset's `criticality` → exactly one audit row with right old/new.
- Integration: delete a WO → row(s) recorded with `action='deleted'`.
- Negative: changes inside a rolled-back transaction → no audit rows.

#### 8.2 — Electronic signatures (FDA 21 CFR Part 11)

**What**: Endpoints and table to capture signatures with meaning, requiring re-authentication.

**Design**:
- Table `electronic_signature` per data-model-suggestion-1.
- Endpoint `POST /api/v1/signatures` body `{ entity_type, entity_id, meaning: "approved"|"reviewed"|"completed"|"authored", password }` — re-validates the user's password (Part 11 requires non-biometric signatures use two distinct components — userid + password), then records the signature.
- Signed records become immutable for the signed fields (enforced by a check at PATCH time).
- Signature image (when captured via mobile signature pad) stored as attachment linked to the signature row.

**Testing**:
- Integration: missing password → 401; correct password → 200; record marked signed.
- Integration: attempting to PATCH a signed field on a signed entity → 409.
- Integration: signature row contains IP, user agent, timestamp.

#### 8.3 — Inspection workflow (OSHA PSM mechanical integrity)

**What**: Inspections table and endpoints for scheduling, recording, and tracking inspection due dates per asset.

**Design**: Use data-model-suggestion-1 §"OSHA PSM inspection" schema. Endpoints:
- `GET/POST/PATCH /api/v1/inspections`.
- `GET /api/v1/inspections/overdue?site_id=` for dashboards.
- Inspection completion can optionally create a follow-up WO.
- Recurrence pattern field on inspections (similar to PM scheduling) auto-creates the next inspection on completion.

**Testing**:
- Integration: complete an inspection with `findings='fail'` and follow-up flag → WO created.

#### 8.4 — KPI snapshot job and API

**What**: Daily Celery beat job that computes `asset_kpi_snapshot` rows for the previous day (and rolling 7/30/90 day windows) plus an aggregation API.

**Design**:
- Computations:
  - MTBF: total operating time between failure events / number of failure events.
  - MTTR: sum(actual_end − actual_start) for completed corrective WOs / count.
  - Availability: 1 − (downtime_minutes / period_minutes).
  - PM compliance %: completed-on-time PM WOs / total PM WOs in period.
  - Total maintenance cost: sum of `actual_cost`.
- Snapshots stored per-asset per period; aggregations rolled up at request time.
- API: `GET /api/v1/kpi/summary?site_id=&period=7d|30d|90d` → top-level rollup; `GET /api/v1/kpi/assets?...` → per-asset table.
- Downtime cost calculated as `downtime_minutes × asset.attributes.hourly_downtime_cost` when set.

**Testing**:
- Fixture-based: 30-day synthetic dataset with known WOs/failures → computed MTBF/MTTR match expected to 2 decimals.
- Integration: API filters return correct subsets.

#### 8.5 — Compliance export (ISO 55001 / OSHA PSM evidence pack)

**What**: An endpoint that produces a ZIP containing the audit log, signed work orders, inspections, and KPI report for a date range — formatted for auditor handover.

**Design**:
- `POST /api/v1/compliance/export` body `{ from, to, site_id, format: 'zip' }` → returns job id; downloadable when ready.
- ZIP contains:
  - `audit-log.csv`
  - `work-orders-signed.csv` + per-WO PDFs (rendered from template)
  - `inspections.csv` + result PDFs
  - `kpi-summary.pdf`
  - `manifest.json` with cryptographic SHA-256 of each file
- Optional GPG signing of the manifest for tamper-evidence.

**Testing**:
- Integration: export against seeded data produces a valid ZIP with all manifests; SHA-256s match contents.

---

## Phase 9: Webhooks, ERP Integration, OpenAPI SDK

### Purpose
Make the CMMS a citizen of the broader IT/OT stack: webhooks for downstream consumers, bidirectional ERP sync adapters (SAP, Oracle, Microsoft Dynamics 365), and a generated TypeScript SDK so customers can integrate quickly. The README's commitment to "REST-API-first with webhook events for work-order lifecycle changes" comes due here.

### Tasks

#### 9.1 — Webhook subscription system

**What**: Tenant-scoped webhook subscriptions with HMAC signing, retries with exponential backoff, and a delivery log.

**Design**:
- Tables: `webhook_subscription(id, tenant_id, url, secret, event_types[], is_active, created_by, created_at)`; `webhook_delivery(id, subscription_id, event_id, status, attempts, last_attempt_at, response_code, response_body, next_retry_at)`.
- Event types (initial set):
  - `workorder.created`, `workorder.updated`, `workorder.status_changed`, `workorder.completed`
  - `asset.created`, `asset.updated`, `asset.decommissioned`
  - `pm.materialised`
  - `inventory.low_stock`
  - `sensor.threshold_breached`, `sensor.ml_anomaly_detected`
  - `signature.recorded`
- Signing: `X-CMMS-Signature: sha256=hex(hmac_sha256(secret, body))` plus `X-CMMS-Timestamp` to prevent replay (consumers reject ±5 minutes).
- Delivery: Celery task `deliver_webhook` with `acks_late=True`; retries at 30s, 5m, 30m, 2h, 12h, then dead-letter.

**Testing**:
- Integration: trigger event → delivery row created → mock HTTP server receives request with valid signature.
- Integration: receiver returns 500 → retry attempted with backoff.

#### 9.2 — ERP sync framework

**What**: Pluggable adapter architecture for ERP sync (assets, parts, vendors, purchase orders) and connectors for SAP S/4HANA, Oracle Fusion, Microsoft Dynamics 365.

**Design**:
- `cmms.services.erp.base.ErpAdapter` interface:
```python
class ErpAdapter(Protocol):
    name: str
    async def list_assets(self, since: datetime | None) -> AsyncIterator[ExternalAsset]: ...
    async def upsert_asset(self, asset: Asset) -> ExternalAssetRef: ...
    async def list_parts(self, since: datetime | None) -> AsyncIterator[ExternalPart]: ...
    async def upsert_part(self, part: Part) -> ExternalPartRef: ...
    async def submit_purchase_order(self, po: PurchaseOrder) -> ExternalPoRef: ...
```
- Sync engine: scheduled Celery jobs per active `erp_connection(id, tenant_id, adapter_name, config_encrypted, last_sync_at)`.
- Idempotency: each external system's ID is mapped to local UUID in `external_id_map(local_type, local_id, system, external_id)`.
- SAP S/4HANA via OData (Business Technology Platform); Oracle Fusion via REST; Dynamics 365 via Common Data Service.
- Each adapter shipped as an optional installable extra (`pip install cmms[erp-sap]`).

**Testing**:
- Unit: each adapter mocked with `respx`; pull/push round-trip succeeds.
- Integration: a "mock-erp" adapter that talks to an in-process FastAPI server exercises the full sync engine.

#### 9.3 — OpenAPI export and TS SDK generation

**What**: `make sdk` regenerates `packages/api-client-ts` from the live OpenAPI spec; pre-commit hook fails if the SDK is out of date.

**Design**:
- `apps/backend/src/cmms/cli.py export-openapi --out docs/api/openapi.json`.
- TS client generated via `openapi-typescript-codegen`.
- Frontend imports types from the generated package only — no manual duplication.

**Testing**:
- CI: SDK regenerated; `git diff` empty → pass; any drift → fail.

#### 9.4 — Marketplace integrations (Slack, Zapier, Make)

**What**: Outgoing webhooks templated for Slack incoming webhooks; an exported OpenAPI-based Zapier app definition; documented Make connector.

**Design**:
- Slack: special webhook subscription type that formats events as Block Kit messages.
- Zapier: published via Zapier CLI with triggers (new WO, status changed) and actions (create WO, comment).

**Testing**:
- Integration: Slack subscription posts a correctly formatted message to a mock receiver.

---

## Phase 10: AI-Native Capabilities

### Purpose
Deliver the differentiators that justify the project's "AI-native" positioning. This phase wraps the data and operational primitives built in phases 1–9 with LLM-powered conversational interfaces, AI-generated procedures, automated root-cause analysis, and ML-driven forecasting. None of this is feasible without the foundation already in place.

### Tasks

#### 10.1 — LLM provider abstraction

**What**: A pluggable `LlmClient` so the CMMS works against Anthropic Claude, OpenAI, Azure OpenAI, AWS Bedrock, or a local Ollama instance with no code changes.

**Design**:
```python
class LlmClient(Protocol):
    async def chat(self, messages: list[Message], tools: list[Tool] | None = None,
                   model: str | None = None, temperature: float = 0.2) -> ChatResponse: ...
    async def stream_chat(self, ...) -> AsyncIterator[ChatChunk]: ...
    async def embed(self, texts: list[str]) -> list[list[float]]: ...
```
- Implementations: `AnthropicClient`, `OpenAiClient`, `AzureOpenAiClient`, `BedrockClient`, `OllamaClient`.
- Wraps `tenacity` for retries on rate limits.
- Per-request cost accounting (`llm_call_log` table) for budget alerts.

**Testing**:
- Unit (mocked): each provider's request mapping correct.
- Integration (recorded with `vcrpy`): chat with tool-use round-trip works for Anthropic.

#### 10.2 — Natural-language CMMS assistant (chat-with-your-CMMS)

**What**: A chat endpoint and web UI page that lets users ask questions like "show me all overdue PMs for Site A" or "create a work order to inspect pump P-101" and the assistant executes the appropriate API calls.

**Design**:
- Endpoint `POST /api/v1/ai/chat` body `{ session_id?, message, context?: { current_asset_id?, current_site_id? } }` → SSE stream.
- The LLM is given a tool catalogue (a curated subset of CMMS endpoints) via Claude's tool-use API. Tools exposed:
  - `list_work_orders(filters)`, `get_work_order(id)`, `create_work_order(...)`, `transition_work_order(...)`
  - `list_assets(filters)`, `get_asset(id)`, `get_asset_history(id)`
  - `list_pm_schedules(filters)`, `materialise_pm(id)`
  - `get_sensor_readings(device_id, from, to, bucket)`
  - `get_kpi_summary(site_id, period)`
  - `search(query)` — FTS across WOs/assets/parts
- All tool calls execute with the calling user's permissions (no privilege escalation).
- System prompt template (excerpt):
  > "You are an AI assistant embedded in a CMMS. You help maintenance managers, planners, and technicians. Always disambiguate before taking destructive actions (delete, cancel). Surface IDs and links; never invent data. If a tool returns no results, say so."
- Conversation persisted in `ai_conversation` + `ai_message` tables; user sees history.

**Testing**:
- Unit: prompt rendering with context.
- Integration (recorded): "list overdue WOs at Site A" → assistant calls `list_work_orders(status='in_progress', due_before=now, site_id=...)` and renders a table.
- Negative: a destructive request without confirmation phrase → assistant asks for confirmation.

#### 10.3 — Natural-language procedure generation

**What**: Given an asset, equipment manual PDFs, and historical WO notes, generate a draft preventive-maintenance procedure (task list).

**Design**:
- Endpoint `POST /api/v1/ai/generate-procedure` body `{ asset_id, manual_attachment_ids?: [...], procedure_type: "monthly_pm"|"annual_pm"|"startup"|"shutdown" }` → returns a draft PM schedule + task templates.
- Process:
  1. Fetch asset + equipment class + recent WO notes for the asset (last 90d) + extracted manual text (parsed via `unstructured` or `pypdf`).
  2. Embed text chunks; create a retrieval-augmented prompt for the LLM with the question "Generate a {procedure_type} maintenance procedure for this {equipment_class.name} ({manufacturer} {model})."
  3. The LLM returns a structured JSON (validated against Pydantic schema) — list of tasks with sequence, title, description, task_type, measurement_unit, expected range.
  4. Returned as a draft for a planner to review and save as a `pm_schedule` + `pm_task_template` rows.
- Provenance: every generated task carries `ai_generated=true` and a `source_chunks` reference list — planner can see which manual page each instruction came from.

**Testing**:
- Integration (recorded): generate for a seeded pump asset with a sample manual → returns ≥5 valid tasks.

#### 10.4 — AI root-cause analysis

**What**: For a recently completed corrective WO, suggest likely root causes by correlating sensor readings, prior WOs on the same asset, and ISO 14224 failure mode/mechanism reference data.

**Design**:
- Endpoint `POST /api/v1/ai/rca/{work_order_id}` → returns a ranked list of `(failure_mode, failure_mechanism, failure_cause, confidence, evidence)`.
- Process:
  1. Pull asset's last 365d of WOs, sensor readings around `actual_start` ± 7d, last inspections, BOM usage.
  2. Construct a context document with these data + ISO 14224 failure taxonomy.
  3. Ask the LLM (Claude Sonnet) to return a structured candidate list with brief evidence per candidate.
  4. Validate every returned `failure_mode` / `mechanism` against the seeded taxonomy; drop invalid suggestions.
- Output stored as `rca_suggestion` rows attached to the WO; planner can accept a suggestion which writes back to `work_order.failure_mode_id / failure_mechanism_id / failure_cause_id`.

**Testing**:
- Fixture-based: a seeded WO with synthetic sensor pre-cursor + maintenance history → top suggestion matches expected ground-truth label in 80% of fixture cases.

#### 10.5 — Spare-parts demand forecasting

**What**: Forecast consumption per part for the next 30/60/90 days using historical usage and asset failure-probability weighting.

**Design**:
- ML pipeline `cmms.ml.forecast`:
  - Inputs per `part_id`: monthly usage history from `work_order_part` joined with WO completion dates; current open WOs that may consume the part; per-asset failure probability (from anomaly model when available, else MTBF-based hazard rate).
  - Model: SARIMA via `statsmodels` for seasonal usage; weighted by sum of expected-consumption-per-asset.
  - Output: `parts_forecast(part_id, horizon_days, p50, p80, p95, computed_at)` table; Celery beat job runs weekly.
- API: `GET /api/v1/parts/forecast?horizon=30|60|90` returns recommended reorder list (p80 minus current on-hand minus on-order).
- Web UI shows it on the Parts Low-Stock view as an "AI-recommended" sort.

**Testing**:
- Fixture: 24-month synthetic usage → forecast within configured error bands.
- Integration: forecast → recommended reorder list excludes parts already on open POs.

#### 10.6 — MCP server

**What**: A Model Context Protocol server exposing CMMS data and operations as tools so any MCP-aware client (Claude Desktop, Claude Code, custom agents) can interact with the CMMS via natural language.

**Design**:
- `apps/mcp` Python package using `mcp` SDK.
- Authentication: MCP client provides a CMMS PAT (personal access token, issued from the web UI's Settings → Tokens page) via the configured transport; the MCP server exchanges it for the user's session and enforces permissions identically to REST.
- Tools (initial): `get_asset`, `search_assets`, `list_work_orders`, `get_work_order`, `create_work_order`, `add_comment`, `list_pms`, `get_sensor_readings`, `get_kpi_summary`, `record_inspection`.
- Resources: `cmms://assets/{id}`, `cmms://work-orders/{id}` — provide rich context blocks for the model.
- Two transports: stdio (Claude Desktop-style) and HTTP/SSE (for hosted use).
- Docker image and a `claude_desktop_config.json` snippet shipped in `docs/integrations/mcp.md`.

**Testing**:
- Integration: launch MCP server, send `tools/list` → expected tool catalogue.
- Integration: tool-call round-trip via stdio matches REST behaviour (same RBAC, same audit log entries).

---

## Phase 11: Hardening, Security, Performance, Multi-Tenancy

### Purpose
Take the system from "works" to "ready for production at scale". Cover OWASP API Security Top 10, multi-tenant isolation guarantees, performance under load, observability, backup/restore, and the hardening expected by enterprise security reviews.

### Tasks

#### 11.1 — Row-Level Security (RLS) for tenant isolation

**What**: Postgres RLS policies enforcing `tenant_id` filtering on every business table.

**Design**:
- Migration enables RLS on every tenant-scoped table and creates a `tenant_isolation` policy: `USING (tenant_id = current_setting('app.tenant_id')::uuid)`.
- API middleware sets `SET LOCAL app.tenant_id = <uuid>` per request transaction.
- A `bypass_rls` role used only by superadmin maintenance tasks; logged.

**Testing**:
- Integration: a request scoped to tenant A cannot read tenant B's rows even when the SQL omits a `tenant_id` filter.
- Integration: cross-tenant FK assignment rejected.

#### 11.2 — OWASP API Security audit

**What**: Implement and test mitigations for OWASP API Security Top 10 (2023): BOLA, broken auth, BOPLA, unrestricted resource consumption, function-level auth, SSRF, security misconfiguration, etc.

**Design**:
- Object-level authorisation: every `get_by_id` repository function takes a `tenant_id` and a permission check; tested with parametrised test matrices.
- Rate limiting: per-token bucket via `slowapi` + Redis; 60 req/min default, configurable per endpoint.
- Resource limits: max page size 200; max file size 50MB (configurable); max attachments per WO 50.
- CORS: explicit allowlist from settings.
- HTTPS only (TLS termination at the reverse proxy; HSTS header set).
- CSP and security headers via `secure` middleware.
- Dependency scanning: `pip-audit` + `npm audit` in CI.

**Testing**:
- E2E security: a script attempts BOLA across tenants → all attempts return 403/404; recorded as a passing security test.
- Load: 1000 concurrent requests sustained at p95 < 200ms.

#### 11.3 — Backup and restore

**What**: Documented and tested process for full backup and restore of Postgres+TimescaleDB and MinIO.

**Design**:
- `pgbackrest` or `wal-g` for continuous archiving + point-in-time recovery; cron job ships nightly base backup off-site.
- TimescaleDB compressed chunks back up at the file level — included in the base backup.
- MinIO bucket lifecycle replication to a secondary endpoint.
- `scripts/restore.sh` is a tested recovery runbook.

**Testing**:
- Quarterly DR drill scripted in CI on Sunday nightly: spin up a fresh stack from yesterday's backup, run a known-data assertion → must pass.

#### 11.4 — Observability and SLOs

**What**: Dashboards, alerts, and SLO definitions.

**Design**:
- Grafana dashboards: API latency by route, error rate, Celery queue depth and lag, sensor ingestion rate, anomaly model drift, DB connection pool saturation.
- Alerts via Alertmanager: error rate >1% for 5m, ingestion lag >5m, queue depth >10k, DB pool ≥95%.
- SLOs: API p95 ≤ 250ms, ingestion event-to-storage p95 ≤ 5s, webhook delivery p95 ≤ 30s.

**Testing**:
- Synthetic check probes hitting `/health/ready` and a sample API every 30s.

#### 11.5 — Helm chart and production deployment guide

**What**: A Helm chart in `deploy/helm/cmms` that deploys the full stack to K8s.

**Design**:
- Sub-charts: `api`, `worker`, `ingest`, `mcp`, `web`. Postgres/Redis/MinIO/EMQX referenced via Bitnami sub-charts (with note that production should use managed services).
- HPA configured per workload.
- NetworkPolicy isolating components.
- Secrets via `SealedSecrets` or `ExternalSecretsOperator` (documented options).
- `values.yaml` documents every knob; `values.production.yaml` example.

**Testing**:
- CI: `helm lint`, `helm template` against `values.yaml` and `values.production.yaml` succeed.
- Integration (optional): a kind cluster job installs the chart and runs a smoke test.

---

## Phase 12: Documentation, Demo Tenant, Release

### Purpose
Make the project genuinely consumable by the open-source community and prospective customers. Comprehensive docs, a self-service demo tenant, contribution guidelines, a release pipeline, and a getting-started experience that takes a developer from clone to running CMMS in under 5 minutes.

### Tasks

#### 12.1 — Documentation site

**What**: A docs site at `apps/docs-site` (Next.js + Nextra or Docusaurus) covering installation, configuration, API reference, integrations, and concepts.

**Design**:
- Sections: Getting Started, Concepts (assets/WOs/PMs/ISO standards), Configuration, Self-Hosting (Docker, Helm), Integrations (ERP, IoT, MCP, Zapier), API Reference (Redoc embed of generated OpenAPI), AI Features, Mobile, Contributing, Changelog.
- Versioned docs aligned to release tags.

**Testing**: Build succeeds in CI; broken-link checker passes.

#### 12.2 — Demo tenant and seed dataset

**What**: A `make demo` target that seeds a realistic tenant with assets, PMs, WOs, parts, sensor data, users, and one month of history so the UI is non-empty on first boot.

**Design**:
- `scripts/load-demo-data.py` creates:
  - Tenant "Acme Manufacturing" with one site "Plant 1"
  - 80 assets across 5 production lines using ISO 14224 equipment classes
  - 25 PM schedules with materialised WOs across the next 30 days
  - 200 historical WOs with realistic failure modes
  - 60 parts with inventory and 10 vendors
  - 12 sensor devices with 7 days of simulated readings (some with injected anomalies)
  - 8 users across roles
- `--with-sensor-stream` flag launches a publisher that streams live readings continuously.

**Testing**:
- Smoke: after `make demo`, the dashboard, calendar, asset tree, and sensor live view all show non-trivial data.

#### 12.3 — Contributing, Code of Conduct, Security Policy

**What**: `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`, `SECURITY.md`, issue/PR templates, DCO check.

**Design**: Standard OSS templates. CONTRIBUTING covers: dev setup, code style, commit conventions (Conventional Commits), test expectations, the CLA/DCO requirement, and architectural ADRs.

**Testing**: CI checks DCO sign-off on PRs.

#### 12.4 — Release pipeline

**What**: Semantic-release pipeline triggered on `main`; tags, changelog, Docker image push, Helm chart push to a registry.

**Design**:
- `release.yml` runs on tag push: builds multi-arch Docker images (`linux/amd64`, `linux/arm64`), pushes to GHCR, builds and pushes Helm chart to `ghcr.io/.../charts`.
- Changelog generated from Conventional Commits.

**Testing**: A tag bump creates artefacts and a GitHub release.

#### 12.5 — Marketing landing page and screencast

**What**: A `/` route on the docs site with a one-paragraph pitch, "Why CMMS" comparison table, demo screencast, install button, and "Star on GitHub" CTA.

**Design**: Static page with shadcn components. Comparison table sourced from `features.md`.

**Testing**: Lighthouse score ≥90 on performance, accessibility, SEO.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation               ─── required by everything
    │
Phase 2: Assets & Taxonomy        ─── requires Phase 1
    │
Phase 3: Work Orders & PMs        ─── requires Phase 2
    │
    ├── Phase 4: Parts & Inventory     ─── requires Phase 3
    │
    ├── Phase 5: IoT & Anomaly         ─── requires Phase 3 (can parallel with Phase 4)
    │
    ├── Phase 6: Mobile App            ─── requires Phase 3 (can parallel with Phase 4 & 5)
    │
    └── Phase 7: Web Dashboard         ─── requires Phase 3+4 (can parallel with Phase 5+6)
         │
Phase 8: Compliance & KPIs        ─── requires Phase 3, 4, 5
    │
Phase 9: Webhooks & ERP & SDK     ─── requires Phase 8 (webhooks rely on audit log)
    │
Phase 10: AI Capabilities         ─── requires Phase 5 (sensor data), Phase 8 (audit + KPI)
    │
Phase 11: Hardening & Production  ─── requires every functional phase
    │
Phase 12: Docs, Demo, Release     ─── final
```

**Parallelism opportunities**:
- Once Phase 3 lands, Phases 4 (Parts), 5 (IoT), 6 (Mobile), and 7 (Web) can be developed concurrently by separate contributors with light coordination on shared schemas.
- Phase 10 (AI) and Phase 11 (Hardening) can run in parallel once Phase 9 is complete.

---

## Definition of Done (per phase)

Every phase is considered complete only when **all** items below are true:

1. **All tasks implemented** as specified in this plan; deviations documented in an ADR under `docs/adr/`.
2. **All unit tests pass** locally and in CI.
3. **All integration tests pass** against testcontainers-launched real dependencies (Postgres+TimescaleDB, Redis, MinIO, EMQX where relevant).
4. **All applicable E2E tests pass** (Playwright for web, Maestro/Detox for mobile).
5. **`ruff check .` and `ruff format --check .` pass** on Python code; **`pnpm lint` and `pnpm typecheck` pass** on TS.
6. **`mypy --strict apps/backend/src` passes**.
7. **`alembic upgrade head` and `alembic downgrade base`** both succeed on a fresh database; new migrations are autogenerated only after manual review.
8. **`docker compose build`** succeeds for every Dockerfile changed; **`helm lint deploy/helm/cmms`** passes.
9. **OpenAPI spec is regenerated** (`make sdk`) and committed; the generated TS client is in sync.
10. **Every new endpoint has** request/response Pydantic schemas, permission checks, an OpenAPI summary/description, and at least one integration test exercising it.
11. **Every new domain table has** Alembic migration, SQLAlchemy model, indexes for foreseeable query patterns, and audit-log coverage (Phase 8 onward).
12. **Every new configurable behaviour is documented** in `docs/configuration.md` with default, env-var name, and acceptable range.
13. **Feature works end-to-end** demonstrated by either an E2E test or a recorded walkthrough committed to `docs/walkthroughs/`.
14. **CHANGELOG.md updated** with a Conventional-Commits-style entry per merged PR.
15. **No new HIGH or CRITICAL findings** from `pip-audit` and `npm audit`.
