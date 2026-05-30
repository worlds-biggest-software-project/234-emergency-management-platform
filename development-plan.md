# Emergency Management Platform — Phased Development Plan

> Project: `234-emergency-management-platform` · Created: 2026-05-29
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `README.md`, `research.md`, `features.md`, `standards.md`, and `data-model-suggestion-1.md` (entity-centric normalised relational, augmented selectively with JSONB extensions per `data-model-suggestion-3.md`). The model is chosen because the project's hard procurement requirements — NIMS compliance, ICS-201 through ICS-220 forms, CAP v1.2 conformance, EDXL-RM messaging, NIEM data exchange, FEMA reimbursement reporting — all reward strict referential integrity and standards-aligned reference data, while jurisdiction-level variability is best handled by JSONB extension columns rather than recurrent DDL migrations.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language (backend) | **Python 3.12** | Strongest open-source ecosystem for the project's three intersecting domains: geospatial (Shapely, GeoAlchemy2, pyproj), LLM/AI orchestration (Anthropic, OpenAI SDKs, LangGraph), and government data standards (lxml, xmlschema, dicttoxml for CAP/EDXL/NIEM). Existing reference implementation Sahana EDEN is also Python, easing community migration. |
| API framework | **FastAPI 0.115+** | Native OpenAPI 3.1 generation (required by standards.md), Pydantic v2 for typed request/response, async support for SSE streaming dashboards, WebSocket support for real-time collaboration on ICS forms. |
| Database | **PostgreSQL 16 + PostGIS 3.4** | Standards mandate GeoJSON (RFC 7946) and polygon queries for CAP `<area>` segments and incident zones — PostGIS is the only credible open-source option. JSONB enables the suggestion-3 hybrid extension pattern. Row-level security (RLS) implements multi-tenant isolation. |
| ORM | **SQLAlchemy 2.0 (async) + GeoAlchemy2** | Mature, type-annotated, supports PostGIS geometry types natively. Alembic for migrations. |
| Schema validation | **Pydantic v2 + JSON Schema 2020-12** | Single source of truth for request/response, settings, JSONB extension shapes, and OpenAPI generation. |
| Task queue | **Celery 5 + Redis 7** | Async workloads: notification dispatch, CAP/IPAWS submission, AI form drafting, data feed polling (NWS, IPAWS-OPEN, OGC WFS). Celery Beat handles scheduled polls. |
| Cache & pub/sub | **Redis 7** | Doubles as Celery broker, SSE fan-out backplane, and short-TTL cache for hot map layers. |
| Real-time transport | **Server-Sent Events (SSE)** via `sse-starlette`; **WebSockets** for collaborative form editing | SSE is the standards-recommended choice (standards.md `W3C SSE`) for one-way dashboard updates; WebSockets reserved for two-way form co-editing where SSE alone is insufficient. |
| Frontend framework | **Next.js 15 (App Router) + TypeScript** | Server components reduce client JS for low-bandwidth EOC environments; React Server Components fit ICS form rendering; widely understood by potential contributors. |
| Map library | **MapLibre GL JS** (with optional Leaflet fallback) | Open-source, vector tiles, supports GeoJSON, Mapbox-style protocols, no proprietary licence. Tiles via MapTiler self-host or OSM. |
| UI component library | **shadcn/ui + Tailwind CSS** | Accessible primitives, no runtime dependency, easy to theme for agency branding. |
| State management | **TanStack Query v5 + Zustand** | Server state via React Query (SSE-aware), local UI state via Zustand. |
| Notification providers | **Twilio** (SMS/voice), **AWS SES / SendGrid** (email), **Firebase Cloud Messaging** (push) — abstracted behind a `NotificationChannel` interface | Allows agencies to swap providers; FCM avoids per-message cost for push. |
| LLM provider | **Anthropic Claude 4.x** (default) via Vercel AI Gateway or direct SDK; **OpenAI** swappable | Default to Claude for the long-context reasoning required for situation report drafting and AAR synthesis; provider-agnostic via an `LLMClient` abstraction. |
| AI orchestration | **LangGraph 0.2+** | Required for multi-step AI flows (CAP draft → critique → revise; situational awareness aggregation → triage → COP update); supports human-in-the-loop checkpoints needed for ICS workflows. |
| MCP server | **`mcp` Python SDK** | Exposes incident state, resource availability, CAP draft generation, and ICS form completion as MCP tools (standards.md Notes §1 first-mover opportunity). |
| Geospatial libraries | **Shapely 2, GeoAlchemy2, pyproj, fiona** | Standard PostGIS-adjacent toolchain. |
| CAP/EDXL XML | **lxml, xmlschema** | Validated XML production against OASIS XSDs; IPAWS submission requires byte-exact CAP v1.2 + IPAWS profile. |
| Authentication | **Authlib** (OAuth 2.0 / OIDC) + **python3-saml** (SAML 2.0) | Both are required by standards.md for government identity provider integration. JWT issued for first-party clients. |
| Background scheduler | **Celery Beat** | Periodic polls of NWS, IPAWS-OPEN, OGC WFS, OpenFEMA, configured per `data_feed` row. |
| Containerisation | **Docker + docker-compose** (dev) / **Helm chart** (prod) | Self-hosted target audiences (county EOCs, sovereign-data agencies) expect Docker; cloud-target audiences (corporate BCP, hospitals) expect Helm/Kubernetes. |
| Object storage | **S3-compatible** (AWS S3, MinIO for self-host) | ICS form attachments, map screenshots, AAR documents, CAP resource binaries. MinIO bundled in docker-compose for offline-tolerant deployments. |
| Testing — backend | **pytest, pytest-asyncio, pytest-postgresql, testcontainers, httpx** | Industry standard; testcontainers spins up real PostGIS for integration tests. |
| Testing — frontend | **Vitest + React Testing Library + Playwright** | Vitest for components, Playwright for E2E flows on the map and ICS forms. |
| Linter / formatter | **Ruff + Black** (Python); **Biome** (TypeScript) | Modern, fast, single-config. |
| Type checker | **mypy --strict** (Python); **tsc --strict** (TypeScript) | Required given the standards-heavy schema. |
| Package manager | **uv** (Python); **pnpm** (TypeScript) | uv is fast and lockfile-deterministic; pnpm handles monorepo workspaces efficiently. |
| Monorepo layout | **Turborepo** | Coordinates backend, web, mobile, shared TypeScript schemas. |
| Mobile app (post-MVP) | **React Native + Expo** | Reuses shadcn-compatible primitives via NativeWind; offline-tolerance needed by field responders is mature on Expo. |
| Observability | **OpenTelemetry → Grafana stack (Loki + Tempo + Prometheus)** | NIST 800-53 AU control family compliance; self-hostable. |
| CI/CD | **GitHub Actions** with reusable workflows | Lint → typecheck → unit tests → integration tests (testcontainers) → docker build → smoke E2E. |
| Licence | **Apache 2.0** | Permits commercial integration (Sahana EDEN uses MIT; Apache 2.0 adds patent grant, which is reassuring for government adopters). |

### Project Structure

```
emergency-management-platform/
├── README.md
├── LICENSE
├── CONTRIBUTING.md
├── pyproject.toml                    # workspace root (uv)
├── package.json                      # pnpm root
├── turbo.json
├── docker-compose.yml                # postgres+postgis, redis, minio, mailpit, backend, web
├── docker-compose.prod.yml
├── deploy/
│   ├── helm/                         # production Helm chart
│   └── terraform/                    # optional cloud IaC
├── docs/
│   ├── architecture.md
│   ├── api/                          # generated OpenAPI artefacts
│   ├── standards-mapping.md          # CAP/NIEM/ICS field crosswalks
│   └── adr/                          # architecture decision records
├── packages/
│   ├── shared-schemas/               # TypeScript types generated from OpenAPI
│   └── ui/                           # shared shadcn components
├── apps/
│   ├── web/                          # Next.js 15 EOC dashboard
│   │   ├── app/
│   │   │   ├── (auth)/
│   │   │   ├── incidents/[id]/
│   │   │   ├── map/
│   │   │   ├── notifications/
│   │   │   ├── resources/
│   │   │   └── reports/
│   │   ├── components/
│   │   │   ├── map/                  # MapLibre wrappers
│   │   │   ├── ics-forms/            # ICS-201..220 React components
│   │   │   └── notifications/
│   │   └── lib/
│   └── mobile/                       # React Native (added in Phase 9)
├── services/
│   └── backend/
│       ├── pyproject.toml
│       ├── alembic.ini
│       ├── alembic/
│       │   └── versions/
│       ├── src/
│       │   └── emp/
│       │       ├── __init__.py
│       │       ├── main.py           # FastAPI app factory
│       │       ├── config.py         # Pydantic settings
│       │       ├── api/
│       │       │   ├── deps.py
│       │       │   ├── v1/
│       │       │   │   ├── auth.py
│       │       │   │   ├── tenants.py
│       │       │   │   ├── incidents.py
│       │       │   │   ├── ics_forms.py
│       │       │   │   ├── resources.py
│       │       │   │   ├── notifications.py
│       │       │   │   ├── cap_alerts.py
│       │       │   │   ├── situational_awareness.py
│       │       │   │   ├── after_action.py
│       │       │   │   └── sse.py
│       │       │   └── ws/           # WebSocket handlers
│       │       ├── db/
│       │       │   ├── base.py       # SQLAlchemy Base
│       │       │   ├── session.py
│       │       │   └── models/       # ORM models (one module per domain)
│       │       ├── schemas/          # Pydantic v2 request/response models
│       │       ├── services/         # business logic per domain
│       │       │   ├── ics/
│       │       │   ├── resources/
│       │       │   ├── notifications/
│       │       │   ├── cap/
│       │       │   ├── feeds/        # NWS, IPAWS, OGC, OpenFEMA pollers
│       │       │   ├── ai/           # LangGraph flows + LLM clients
│       │       │   └── reporting/
│       │       ├── integrations/
│       │       │   ├── twilio.py
│       │       │   ├── sendgrid.py
│       │       │   ├── fcm.py
│       │       │   ├── ipaws.py
│       │       │   ├── nws.py
│       │       │   ├── openfema.py
│       │       │   └── ogc_wfs.py
│       │       ├── standards/        # CAP/EDXL/NIEM serialisers + XSDs
│       │       │   ├── cap_v12.py
│       │       │   ├── edxl_rm.py
│       │       │   ├── edxl_have.py
│       │       │   ├── niem.py
│       │       │   └── schemas/      # bundled XSDs / JSON Schemas
│       │       ├── mcp_server/       # MCP server exposing incident tools
│       │       ├── tasks/            # Celery tasks
│       │       ├── auth/
│       │       │   ├── oauth.py
│       │       │   ├── oidc.py
│       │       │   ├── saml.py
│       │       │   └── rls.py        # RLS session helpers
│       │       └── observability/
│       └── tests/
│           ├── unit/
│           ├── integration/
│           ├── standards/            # CAP/EDXL XSD conformance tests
│           ├── e2e/                  # API-level happy-path scenarios
│           └── fixtures/             # sample incidents, feeds, CAP XML
└── scripts/
    ├── seed_reference_data.py        # ICS form types, ESF, NIMS resource types
    ├── seed_demo_incident.py
    └── export_openapi.py
```

---

## Phase 1: Foundation, Tenancy & Reference Data

### Purpose
Establish the project skeleton, database, authentication scaffold, configuration, observability, and the standards-aligned reference data that everything else depends on (ICS form types, ESF functions, NIMS resource typing, CAP enumerations). After Phase 1, a developer can sign up, create a tenant, log in, and inspect seeded reference data via authenticated REST endpoints.

### Tasks

#### 1.1 — Monorepo bootstrap and tooling

**What**: Initialise the Turborepo workspace with backend (uv), web app placeholder (pnpm + Next.js 15), shared schemas package, Docker Compose, CI workflow, and pre-commit hooks.

**Design**:
- `pyproject.toml` (root) declares `services/backend` as a `uv` workspace member; pins Python 3.12.
- `package.json` (root) configures pnpm workspaces; `turbo.json` defines pipelines: `lint`, `typecheck`, `test`, `build`.
- `docker-compose.yml` services:
  - `postgres` — `postgis/postgis:16-3.4`, ports `5432`, volume `pgdata`
  - `redis` — `redis:7-alpine`
  - `minio` — `quay.io/minio/minio`, console `9001`
  - `mailpit` — local SMTP capture, web UI `8025`
  - `backend` — built from `services/backend/Dockerfile`, hot-reload via `uvicorn --reload`
  - `web` — Next.js dev server
- Pre-commit hooks: `ruff format`, `ruff check --fix`, `mypy services/backend/src`, `biome check`, `pnpm typecheck`.
- GitHub Actions workflow `ci.yml`: matrix on `backend` and `web`; spins up `services: postgres` for integration tests.

**Testing**:
- `Unit: pyproject.toml parses with uv lock`
- `Unit: pnpm install completes in CI`
- `Integration: docker compose up -d && curl http://localhost:8000/health → {"status":"ok"}`
- `Integration: GitHub Actions workflow passes on a fresh PR`

#### 1.2 — FastAPI app factory and configuration

**What**: Build the FastAPI application skeleton with Pydantic-Settings configuration, structured logging, OpenTelemetry instrumentation, and a `/health`, `/healthz`, `/version` baseline.

**Design**:
- `Settings` (Pydantic):
  ```python
  class Settings(BaseSettings):
      env: Literal["dev", "staging", "prod"] = "dev"
      database_url: PostgresDsn
      redis_url: RedisDsn
      secret_key: SecretStr
      jwt_algorithm: Literal["HS256", "RS256"] = "HS256"
      jwt_access_ttl_minutes: int = 30
      jwt_refresh_ttl_days: int = 14
      cors_origins: list[AnyHttpUrl] = []
      otel_endpoint: AnyHttpUrl | None = None
      s3_endpoint: AnyHttpUrl
      s3_bucket: str
      twilio: TwilioSettings | None = None
      sendgrid: SendGridSettings | None = None
      fcm: FCMSettings | None = None
      ipaws: IPAWSSettings | None = None
      llm: LLMSettings
      class Config: env_nested_delimiter = "__"
  ```
- `create_app()` factory wires middleware in this order: request ID, OpenTelemetry, CORS, GZip, exception handler, RLS session middleware (placeholder), router includes.
- Logging via `structlog` JSON renderer; correlation ID from `X-Request-ID`.
- OpenAPI metadata set with `openapi_version="3.1.0"` and tags grouped by domain.
- `/version` returns `{"version": <pkg version>, "git_sha": <env GIT_SHA>}`.

**Testing**:
- `Unit: Settings with all envs present → loads`
- `Unit: Settings missing DATABASE_URL → ValidationError`
- `Unit: /health returns 200 with {"status":"ok"}`
- `Unit: /version returns expected fields`
- `Integration: OpenAPI spec at /openapi.json validates against OpenAPI 3.1 schema`

#### 1.3 — Database, migrations, and PostGIS

**What**: Configure SQLAlchemy 2 async session, Alembic for migrations, enable PostGIS, and create the first migration containing the `tenant`, `app_user`, `role`, `user_role`, `organisation`, and `facility` tables exactly as in `data-model-suggestion-1.md`.

**Design**:
- `db/base.py`:
  ```python
  class Base(DeclarativeBase):
      metadata = MetaData(naming_convention={
          "ix": "ix_%(table_name)s_%(column_0_name)s",
          "uq": "uq_%(table_name)s_%(column_0_name)s",
          "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
          "pk": "pk_%(table_name)s",
      })
  ```
- `db/session.py` exposes `async_engine`, `AsyncSessionLocal`, and a `get_session()` dependency that sets `SET LOCAL app.tenant_id = :tenant_id` for RLS.
- Alembic migration `0001_initial.sql.py`: enables extensions `CREATE EXTENSION IF NOT EXISTS postgis;` and `CREATE EXTENSION IF NOT EXISTS "uuid-ossp";`. Creates tenancy + organisation tables. Adds RLS policies:
  ```sql
  ALTER TABLE app_user ENABLE ROW LEVEL SECURITY;
  CREATE POLICY tenant_isolation ON app_user
      USING (tenant_id::text = current_setting('app.tenant_id', true));
  ```
- ORM model example:
  ```python
  class Tenant(Base):
      __tablename__ = "tenant"
      id: Mapped[UUID] = mapped_column(primary_key=True, server_default=text("gen_random_uuid()"))
      name: Mapped[str]
      slug: Mapped[str] = mapped_column(unique=True)
      jurisdiction: Mapped[str | None]
      tier: Mapped[str] = mapped_column(default="standard")
      settings: Mapped[dict] = mapped_column(JSONB, default=dict)
      created_at: Mapped[datetime] = mapped_column(server_default=func.now())
      updated_at: Mapped[datetime] = mapped_column(server_default=func.now(), onupdate=func.now())
  ```

**Testing**:
- `Integration (testcontainers postgis): alembic upgrade head succeeds; postgis extension present`
- `Integration: RLS prevents cross-tenant read — insert two tenants, query as tenant A returns only tenant A rows`
- `Integration: Foreign key violation when inserting app_user with unknown tenant_id`
- `Unit: Tenant.slug uniqueness enforced (IntegrityError on duplicate)`

#### 1.4 — Authentication and RLS context

**What**: Implement local password auth, JWT issuance, an `RequireUser` dependency, and propagate `tenant_id` into the PostgreSQL session for RLS. Stub SAML and OIDC routes (real integration deferred to Phase 8).

**Design**:
- `POST /v1/auth/register` — `{email, password, display_name, tenant_slug}` → creates `app_user`.
- `POST /v1/auth/login` — `{email, password, tenant_slug}` → `{access_token, refresh_token, expires_at}`.
- `POST /v1/auth/refresh` — refresh token rotation.
- `GET /v1/auth/me` — returns current user + roles + tenant.
- Password hashing: `argon2-cffi`.
- JWT claims: `sub` (user_id), `tid` (tenant_id), `roles[]`, `iat`, `exp`, `jti`.
- `Depends(get_current_user)` decodes JWT, looks up user, raises 401 on failure, sets contextvar for RLS.
- RLS session middleware: on each request, after authentication, run `await session.execute(text("SET LOCAL app.tenant_id = :tid"), {"tid": current_user.tenant_id})`.

**Testing**:
- `Unit: argon2 hash → verify(password) is True; verify(wrong) is False`
- `Integration: register → login → call /auth/me returns user data`
- `Integration: expired JWT → 401`
- `Integration: valid JWT for tenant A → cannot read tenant B's organisations (returns empty list, not 403)`
- `Integration: refresh rotation — old refresh token rejected after rotation`

#### 1.5 — Reference data tables and seeders

**What**: Create the standards-derived reference data tables (`esf_function`, `ics_form_type`, `resource_category`, `resource_type_definition`, CAP enumeration tables) and write idempotent Python seeders.

**Design**:
- Add Alembic migration `0002_reference_data.py` with table definitions from data-model-suggestion-1.md.
- `scripts/seed_reference_data.py`:
  - 15 ESF functions (ESF #1 Transportation → ESF #15 External Affairs, with lead agencies from National Response Framework).
  - 20 ICS form types (`ICS-201` through `ICS-220` with official titles).
  - NIMS resource typing seed loaded from a committed `data/nims_resource_typing.yaml` covering at minimum: fire (engine type 1-3, USAR teams), EMS (ambulance ALS/BLS), law enforcement, public works, SAR.
  - CAP enumerations as static module constants (`CAP_STATUS`, `CAP_MSG_TYPE`, `CAP_SCOPE`, `CAP_CATEGORY`, `CAP_URGENCY`, `CAP_SEVERITY`, `CAP_CERTAINTY`, `CAP_RESPONSE_TYPE`).
- Seeder is idempotent: uses `ON CONFLICT (code) DO UPDATE` for the lookup tables.

**Testing**:
- `Unit: seed_reference_data run twice → no duplicates, no errors`
- `Unit: All 15 ESFs present with correct lead agency strings`
- `Unit: ICS form types include 201, 202, 204, 205, 206, 207, 208, 209, 210, 211, 213, 214, 215, 215A, 218, 220`
- `Unit: CAP enumeration constants match OASIS CAP v1.2 §3.2.1 exact strings (case-sensitive)`
- `Integration: GET /v1/reference/ics-form-types returns seeded rows`

#### 1.6 — Audit trail middleware

**What**: Implement the generic `audit_trail` table writer and a SQLAlchemy event listener that captures all mutations on a configurable set of "auditable" tables.

**Design**:
- Add `audit_trail` table from data-model-suggestion-1.md in migration `0003_audit.py`.
- SQLAlchemy `after_insert`, `after_update`, `after_delete` hooks on registered models:
  ```python
  AUDITABLE_MODELS = {Incident, ICSForm, ResourceRequest, Notification, CAPAlert, AfterActionReport, ...}
  ```
- Captures `old_values`/`new_values` as JSONB diffs (only changed columns).
- Sources `user_id`, `ip_address`, `user_agent` from request contextvar.
- `GET /v1/audit?entity_type=X&entity_id=Y` returns paginated history (admin role only).

**Testing**:
- `Unit: update incident.status → audit_trail row created with old/new values`
- `Unit: delete row → audit row preserves old_values`
- `Integration: audit endpoint requires admin role; 403 for non-admin`
- `Integration: NIST 800-53 AU-2 — verify audit captures user_id, timestamp, action, entity, source IP`

---

## Phase 2: Incident Lifecycle & Chronology

### Purpose
Deliver the heart of the platform — creating, activating, and closing incidents — along with operational periods and the chronological log entries that drive after-action reporting. After Phase 2, an EOC operator can declare an incident, define an operational period, and log events that appear in a time-ordered incident chronology.

### Tasks

#### 2.1 — Incident CRUD

**What**: Implement `incident` and `operational_period` tables, ORM, services, and REST endpoints.

**Design**:
- Endpoints:
  - `POST /v1/incidents` — create
  - `GET /v1/incidents` — list with filters: `status`, `incident_type`, `severity`, `start_time` bbox, geo bbox
  - `GET /v1/incidents/{id}` — detail (includes current operational period, ESFs)
  - `PATCH /v1/incidents/{id}` — update fields; transitions enforced by state machine
  - `POST /v1/incidents/{id}/operational-periods` — create next period (auto-increment `period_number`)
- State machine for `incident.status`:
  ```
  draft → active → monitoring → demobilising → closed
  any → cancelled
  ```
- Validation: `incident_number` unique per tenant; `end_time > start_time` when present.
- GeoJSON serialisation for `location` and `affected_area` via `geoalchemy2.shape.to_shape` + `geojson_pydantic`.
- Request schema:
  ```python
  class IncidentCreate(BaseModel):
      incident_number: str
      name: str
      incident_type: Literal["wildfire","flood","hazmat","earthquake","sar","public_health","cyber","other"]
      severity: Literal["minor","moderate","major","catastrophic"] = "moderate"
      location: PointGeometry | None = None
      affected_area: PolygonGeometry | None = None
      start_time: datetime
      esfs: list[int] = Field(default_factory=list)
  ```

**Testing**:
- `Unit: invalid status transition (active → draft) → ValueError`
- `Unit: duplicate incident_number in same tenant → 409`
- `Integration: POST /v1/incidents with geo polygon → stored as PostGIS, returned as GeoJSON`
- `Integration: GET /v1/incidents?bbox=... returns only incidents intersecting bbox`
- `Integration: list incidents respects RLS — tenant B sees no tenant A incidents`

#### 2.2 — Operational period management

**What**: Implement operational period CRUD with auto-incrementing period numbers, objectives, and rollover semantics.

**Design**:
- `POST /v1/incidents/{id}/operational-periods` payload:
  ```python
  class OperationalPeriodCreate(BaseModel):
      start_time: datetime
      end_time: datetime
      objectives: str | None = None
      weather_forecast: str | None = None
      safety_message: str | None = None
  ```
- Server auto-assigns `period_number = max(existing) + 1`.
- Constraint: new period's `start_time` must equal previous period's `end_time` (no gaps), enforced at service layer.
- `GET /v1/incidents/{id}/operational-periods` returns ordered list.
- `GET /v1/incidents/{id}/operational-periods/current` returns the period containing `now()`.

**Testing**:
- `Unit: first operational period gets period_number=1`
- `Unit: second period gets period_number=2`
- `Unit: gap between periods → ValueError`
- `Unit: overlapping periods → ValueError`
- `Integration: GET .../current returns active period during incident`

#### 2.3 — Incident log entries (chronology)

**What**: Provide a high-throughput append-only log of incident events that backs the chronology UI, AAR generation, and audit narrative.

**Design**:
- Endpoints:
  - `POST /v1/incidents/{id}/log` — add entry
  - `GET /v1/incidents/{id}/log?since=...&type=...&limit=...&cursor=...` — cursor-paginated, time-descending
  - SSE: `GET /v1/incidents/{id}/log/stream` — appends pushed live
- Entry types: `status_change`, `resource_deployed`, `notification_sent`, `form_approved`, `note`, `communication`, `feed_event` (from data feeds), `ai_action`.
- Auto-logging: services for resource deployment, form approval, notification dispatch each emit a log entry via a shared `log_incident_event()` helper.
- SSE backplane via Redis pub/sub channel `incident:{id}:log`.

**Testing**:
- `Unit: log entry insertion publishes to Redis channel`
- `Integration: status change on incident creates a status_change log entry`
- `Integration: SSE client receives new entries within 500ms of write`
- `Integration: cursor pagination returns entries in deterministic time-desc order`
- `Integration: filter by entry_type returns matching rows only`

#### 2.4 — Real-time event bus

**What**: Provide a typed in-process event bus with a Redis pub/sub adapter so domain services can emit events (`IncidentCreated`, `ResourceDeployed`, `CAPAlertSent`, etc.) that other services subscribe to without coupling.

**Design**:
- `services/events.py`:
  ```python
  class DomainEvent(BaseModel):
      event_type: str
      tenant_id: UUID
      incident_id: UUID | None = None
      payload: dict
      occurred_at: datetime = Field(default_factory=lambda: datetime.now(UTC))

  async def publish(event: DomainEvent) -> None: ...
  def subscribe(event_type: str, handler: Callable[[DomainEvent], Awaitable[None]]): ...
  ```
- Handlers register via decorator on import.
- Each event also writes an `incident_log_entry` if `incident_id` is set.
- Redis channel naming: `events:{tenant_id}:{event_type}`.

**Testing**:
- `Unit: subscribe to event_type X → handler invoked when event published`
- `Unit: handler exception does not block other handlers`
- `Integration: cross-process event via Redis received by second worker`

---

## Phase 3: ICS Command Structure & Forms (Core Value)

### Purpose
Deliver the ICS-native heart of the platform: position hierarchy, assignments, and the digital ICS form library (ICS-201 through ICS-220). This is the primary differentiator versus mass-notification-only competitors and a procurement prerequisite (NIMS compliance). After Phase 3, an Incident Commander can build an org chart, assign people to positions, and collaboratively author ICS forms.

### Tasks

#### 3.1 — ICS position templates and hierarchy

**What**: Implement `ics_position` table with templates seeded from FEMA NIMS reference and per-incident customised trees.

**Design**:
- Seed canonical position templates per tenant on tenant creation: `IC`, `PIO`, `LO` (Liaison), `SO` (Safety), `OSC`, `PSC`, `LSC`, `FSC`, plus branches/divisions for each section (e.g., `OPS-BRANCH-1`, `OPS-DIVISION-A`).
- Hierarchical via `parent_id`; recursive CTE queries exposed via service.
- Endpoint `GET /v1/ics/positions/tree?incident_id=...` returns nested JSON tree.
- Custom positions: `POST /v1/ics/positions` allows tenant admins to add custom positions (e.g., agency-specific Deputy roles).

**Testing**:
- `Unit: tenant created → 9 canonical command positions seeded`
- `Unit: recursive descent on IC node returns full tree`
- `Integration: GET /tree returns valid nested structure with expected depth`
- `Integration: deleting a position with children → 409 (must reassign first)`

#### 3.2 — Assignment lifecycle

**What**: Implement `ics_assignment` with assign/release semantics tied to operational periods.

**Design**:
- `POST /v1/incidents/{id}/assignments`:
  ```python
  class AssignmentCreate(BaseModel):
      op_period_id: UUID
      position_id: UUID
      user_id: UUID | None = None
      organisation_id: UUID | None = None
      notes: str | None = None
  ```
- At most one active assignment per `(op_period_id, position_id)` enforced via partial unique index on `released_at IS NULL`.
- `POST /v1/assignments/{id}/release` sets `released_at = now()`.
- Span-of-control validation: warning (not blocking) if any user supervises more than 7 direct reports (NIMS recommended 1:3–1:7 ratio).

**Testing**:
- `Unit: two active assignments to same position in same period → 409`
- `Unit: assigning released position succeeds (creates new active row)`
- `Integration: span-of-control > 7 returns warning in response body`
- `Integration: position tree query returns current assignee per position`

#### 3.3 — ICS form framework

**What**: Implement the polymorphic ICS form storage — a base `ics_form` table plus one detail table per form type (ICS-201, 202, 204, 205, 206, 209, 215, 215A, 218).

**Design**:
- Use the per-form-type table approach from data-model-suggestion-1.md for strongly-typed forms (queries like "what was percent_contained at hour 24?" must be straightforward).
- Forms accessed via discriminator in `ics_form.form_type`; service layer dispatches to per-form repository.
- Endpoints (template repeated per form type):
  - `POST /v1/incidents/{id}/forms/ics-209` — create draft
  - `GET /v1/incidents/{id}/forms/ics-209/{form_id}`
  - `PATCH /v1/incidents/{id}/forms/ics-209/{form_id}` — autosave (returns new version)
  - `POST /v1/incidents/{id}/forms/ics-209/{form_id}/submit` — draft → review
  - `POST /v1/incidents/{id}/forms/ics-209/{form_id}/approve` — review → approved (requires `approver` role)
  - `POST /v1/incidents/{id}/forms/ics-209/{form_id}/supersede` — creates new version + marks old as superseded
- Form versioning: `version` int incremented; `prepared_by`, `approved_by`, `approved_at` tracked.
- JSON Schema generated per form type, exported in OpenAPI; React components use generated TypeScript types.

**Testing**:
- `Unit: state transitions: draft → review → approved (valid); draft → approved (invalid)`
- `Unit: approve requires approver role → 403 otherwise`
- `Unit: supersede creates new form with version + 1, old.status = 'superseded'`
- `Integration: ICS-209 PATCH increments version on each save`
- `Integration: GET ICS-201 returns nested map_sketch_url, situation_summary, resource_summary[]`

#### 3.4 — Collaborative form editing (CRDT-lite)

**What**: WebSocket-based collaborative editing for ICS forms with last-writer-wins per field plus presence indicators.

**Design**:
- `WS /v1/incidents/{id}/forms/{form_id}/collab`:
  - Server broadcasts `{type: "field_update", field: "situation_summary", value: "...", version, user_id}` to all subscribers.
  - Server tracks last-writer-wins per field with monotonic version.
  - Presence messages: `{type: "presence", user_id, cursor: {field, position}}`.
- Storage: every field change persisted on debounce (300ms) to per-form detail table.
- Conflict UI: if two writers edit the same field, the loser receives a `conflict` event with diff; UI prompts for manual merge.
- Connection auth: JWT in `Sec-WebSocket-Protocol` header.

**Testing**:
- `Integration: two WS clients edit different fields → both updates visible to both clients`
- `Integration: two WS clients edit same field → later write wins, earlier client receives conflict event`
- `Integration: disconnect → presence event removes user from active list`
- `Integration: unauthenticated WS connection → closed with 1008`

#### 3.5 — Incident Action Plan (IAP) compiler

**What**: Generate a printable PDF IAP composed of approved ICS-202, 203, 204, 205, 206, and 207 for a given operational period.

**Design**:
- `POST /v1/incidents/{id}/operational-periods/{op_id}/iap` triggers Celery task `compile_iap.delay(incident_id, op_id)`.
- Task renders Jinja2 HTML templates per form type, combines, converts to PDF via `weasyprint`, uploads to S3, returns presigned URL.
- IAP requires at minimum: approved ICS-202 and at least one approved ICS-204.
- `GET /v1/incidents/{id}/operational-periods/{op_id}/iap` returns latest compiled IAP metadata + URL.

**Testing**:
- `Unit: missing ICS-202 → 409 with message`
- `Integration: compile_iap produces a PDF; assert page count >= num forms`
- `Integration: regression — IAP fields populated from each form's approved version (not latest draft)`
- `Snapshot: HTML output for a sample incident matches golden file`

---

## Phase 4: Resource Management

### Purpose
Track personnel, equipment, vehicles, and teams with NIMS-aligned typing; handle resource requests and deployments tied to ICS assignments and incidents. After Phase 4, EOC operators can request resources, approve them, deploy them, and see their utilisation status.

### Tasks

#### 4.1 — Resource catalogue

**What**: Implement `resource_category`, `resource_type_definition`, and `resource` tables with NIMS resource typing.

**Design**:
- Reference data (from Phase 1.5) provides categories and type definitions.
- `Resource` lifecycle states: `available → assigned → out_of_service → reserved → available`.
- Endpoints:
  - `GET /v1/resources?type=...&status=...&near=lat,lon,km` — geo-filter by `current_location`
  - `POST /v1/resources` — register
  - `PATCH /v1/resources/{id}` — update status, location
  - `GET /v1/resources/{id}/history` — deployment history
- Capabilities and certifications stored as `TEXT[]` for fast `@>` containment queries.

**Testing**:
- `Unit: status transition out_of_service → available allowed; assigned → out_of_service requires demobilisation`
- `Integration: GET ?near=... returns resources within km using PostGIS ST_DWithin`
- `Integration: GET ?capability=swift_water returns only matching resources`
- `Integration: bulk import via POST /v1/resources/import (CSV) creates N rows`

#### 4.2 — Resource requests (EDXL-RM aligned)

**What**: Resource request workflow with approval and fulfilment, conformant with EDXL-RM v1.0 message structure.

**Design**:
- `POST /v1/incidents/{id}/resource-requests`:
  ```python
  class ResourceRequestCreate(BaseModel):
      resource_type_id: UUID
      quantity: int = 1
      priority: Literal["immediate","priority","routine"] = "routine"
      needed_by: datetime | None = None
      delivery_location: PointGeometry | None = None
      delivery_address: str | None = None
      justification: str
  ```
- State machine: `pending → approved → fulfilled` or `pending → denied` or `* → cancelled`.
- `POST /v1/resource-requests/{id}/approve` (role: logistics_chief or higher).
- `POST /v1/resource-requests/{id}/fulfill {resource_ids: [...]}` — creates `resource_deployment` rows.
- EDXL-RM XML export: `GET /v1/resource-requests/{id}/edxl` returns valid EDXL-RM 1.0 `<RequestResource>` message.

**Testing**:
- `Unit: state transition diagram exhaustively tested`
- `Standards: EDXL-RM XML output validates against bundled EDXL-RM 1.0 XSD`
- `Integration: fulfill creates resource_deployment rows and emits ResourceDeployed event`
- `Integration: deny request requires reason in payload → 422 otherwise`

#### 4.3 — Resource deployment tracking

**What**: Track resources deployed to incidents over time with operational period attribution.

**Design**:
- `resource_deployment` rows created on fulfilment; `demobilised_at` set on release.
- `GET /v1/incidents/{id}/deployments` — current deployments with resource details.
- `GET /v1/incidents/{id}/deployments?at=<timestamp>` — historical snapshot (uses `deployed_at <= t AND (demobilised_at IS NULL OR demobilised_at > t)`).
- Resource `status` automatically transitions to `assigned` on deployment, back to `available` on demobilisation.

**Testing**:
- `Unit: deploying resource sets status to assigned; demobilising sets to available`
- `Integration: historical snapshot ?at=T returns only deployments active at T`
- `Integration: deploying same resource to two active incidents → 409`

#### 4.4 — Mutual-aid agreements and cross-tenant resource visibility

**What**: Allow tenants to register mutual-aid agreements with other tenants and selectively expose resources.

**Design**:
- `mutual_aid_agreement` table per data-model-suggestion-1.md.
- New table `mutual_aid_visibility`:
  ```sql
  CREATE TABLE mutual_aid_visibility (
      agreement_id UUID NOT NULL REFERENCES mutual_aid_agreement(id),
      resource_type_id UUID REFERENCES resource_type_definition(id),
      max_quantity INTEGER,
      requires_approval BOOLEAN DEFAULT true,
      PRIMARY KEY (agreement_id, resource_type_id)
  );
  ```
- RLS extended: a tenant can read resources of partner tenants under active agreements via a view `available_mutual_aid_resources`.
- `GET /v1/mutual-aid/resources?type=...` returns own + partners.
- Requesting partner resources: `POST /v1/mutual-aid/requests` creates a `resource_request` on the partner tenant with `requires_approval` honoured.

**Testing**:
- `Integration: tenant A creates agreement with tenant B → A can see B's published resources`
- `Integration: tenant B receives mutual-aid request from tenant A; appears in their inbox`
- `Integration: revoking agreement removes cross-tenant visibility immediately`

---

## Phase 5: GIS, Situational Awareness & Data Feeds

### Purpose
Aggregate external real-time data (weather, IPAWS, OGC feature services, OpenFEMA) into the platform's common operating picture, render incidents and resources on a map, and let operators define markers and zones. After Phase 5, the map dashboard displays live incidents with weather overlays and external data layers.

### Tasks

#### 5.1 — Map markers and zones

**What**: Implement `incident_marker` and `incident_zone` CRUD plus the GeoJSON feature endpoints feeding the map UI.

**Design**:
- Endpoints:
  - `POST /v1/incidents/{id}/markers` — add point feature (POI, hazard, road closure, staging area, command post, helispot)
  - `POST /v1/incidents/{id}/zones` — add polygon (evacuation, hot, warm, cold, exclusion, shelter-in-place)
  - `GET /v1/incidents/{id}/features` — combined FeatureCollection (markers + zones + incident location + deployed resources)
  - `GET /v1/map/features?bbox=...&type=...` — global map view filtered by bbox and feature type
- Response is RFC 7946-compliant GeoJSON FeatureCollection with `properties` carrying entity metadata.

**Testing**:
- `Unit: marker creation requires valid Point geometry → 422 on bad coords`
- `Unit: zone creation requires valid Polygon`
- `Standards: GeoJSON output validates against RFC 7946 JSON Schema`
- `Integration: bbox query returns only intersecting features`

#### 5.2 — Data feed framework

**What**: Generic poller framework — each `data_feed` row drives a Celery Beat schedule that polls a source, normalises into events, and writes log entries.

**Design**:
- `services/feeds/base.py`:
  ```python
  class FeedAdapter(Protocol):
      feed_type: str
      async def poll(self, feed: DataFeed) -> list[FeedEvent]: ...

  class FeedEvent(BaseModel):
      feed_id: UUID
      external_id: str
      occurred_at: datetime
      location: PointGeometry | None
      area: PolygonGeometry | None
      summary: str
      raw: dict
      severity: str | None
  ```
- Registry maps `feed_type → adapter`.
- Celery Beat reads active feeds and schedules polls per `polling_interval_seconds`.
- Dedupe via `(feed_id, external_id)` unique index on a new `feed_event` table.
- Configurable per-tenant per-feed.

**Testing**:
- `Unit: registering a new adapter → discoverable from registry`
- `Integration: poll task writes feed_event rows; dedupes second run`
- `Integration: disabled feed not polled by beat`

#### 5.3 — NWS Weather adapter

**What**: Adapter for the NOAA NWS API at `api.weather.gov`.

**Design**:
- Endpoint `https://api.weather.gov/alerts/active?area={state}` returns GeoJSON FeatureCollection.
- Adapter maps each alert to `FeedEvent`; preserves CAP-equivalent fields (`event`, `severity`, `urgency`, `certainty`).
- Required `User-Agent` header per NWS API rules: `"emergency-management-platform ({contact_email})"`.
- Per-tenant config: state(s) of interest; severity threshold.

**Testing**:
- `Unit: parse sample NWS GeoJSON fixture → N FeedEvents with correct fields`
- `Unit: missing User-Agent → adapter raises ConfigurationError`
- `Integration (mocked HTTP): poll generates expected events`

#### 5.4 — IPAWS-OPEN Atom feed adapter

**What**: Subscribe to FEMA's IPAWS-OPEN All-Hazards Information Feed and ingest CAP alerts.

**Design**:
- Feed URL configurable; default `https://apps.fema.gov/IPAWSOPEN_EAS_SERVICE/rest/recent/...`.
- Parses RFC 4287 Atom; each `<entry>` contains a CAP v1.2 XML payload.
- Stores each alert as a `cap_alert` (with full `cap_info`, `cap_area`, `cap_resource` segments) AND emits a `FeedEvent`.
- IPAWS Collaborative Operating Group (COG) ID stored on tenant; alerts targeted at that COG are tagged.

**Testing**:
- `Standards: parse bundled IPAWS sample Atom + CAP → fields populated per CAP v1.2 spec`
- `Unit: invalid CAP XML → adapter records error, does not crash poller`
- `Integration: same alert polled twice → only one cap_alert row`

#### 5.5 — OGC API Features / WFS adapter

**What**: Consume authoritative government GIS layers (flood zones, fire perimeters, critical infrastructure).

**Design**:
- Supports both OGC API - Features (modern JSON) and legacy WFS 2.0 (XML).
- Per-feed config: collection ID, bbox filter, attribute mappings.
- Periodically refreshes a `gis_layer_cache` table that the map service serves as MVT tiles via `pg_tileserv`-compatible queries (using PostGIS `ST_AsMVT`).

**Testing**:
- `Standards: parse OGC API Features GeoJSON response → cache rows match`
- `Standards: parse WFS 2.0 GetFeature XML response → cache rows match`
- `Integration: MVT endpoint returns binary protobuf for a tile (x,y,z)`

#### 5.6 — OpenFEMA adapter

**What**: Pull active disaster declarations and historical Public Assistance data for situational context.

**Design**:
- Endpoint `https://www.fema.gov/api/open/v2/DisasterDeclarationsSummaries?$filter=...`.
- Polls daily; stores in `fema_declaration` cache table.
- `GET /v1/situational-awareness/fema?state=...&since=...` exposes filtered view.

**Testing**:
- `Unit: parse OpenFEMA JSON response → correct field mapping`
- `Integration: poll task respects $filter and pagination`

#### 5.7 — Common Operating Picture (COP) endpoint

**What**: Unified read endpoint feeding the map dashboard with everything needed in one round-trip.

**Design**:
- `GET /v1/cop?bbox=...&incident_ids=...` returns:
  ```json
  {
    "incidents": [...],
    "resources": [...],
    "markers": [...],
    "zones": [...],
    "active_alerts": [...],
    "feed_events": [...],
    "weather": {...}
  }
  ```
- SSE channel `GET /v1/cop/stream?bbox=...` emits `cop_update` events on any change in the bbox.

**Testing**:
- `Integration: COP response respects bbox; entities outside bbox excluded`
- `Integration: SSE receives update within 1s of a new feed_event in bbox`
- `Integration: response shape validates against OpenAPI schema`

---

## Phase 6: Mass Notification & CAP Alert Origination

### Purpose
Deliver multi-channel mass notification (SMS, email, voice, mobile push) with audience targeting, plus CAP-compliant alert origination targeted at IPAWS. After Phase 6, an operator can compose a notification, target an audience, dispatch it, and observe delivery status. CAP-compliant XML can be produced (IPAWS submission proper is Phase 7).

### Tasks

#### 6.1 — Notification templates

**What**: CRUD for `notification_template` with category-based prebuilt library (150+ scenarios).

**Design**:
- Endpoints standard CRUD.
- Seed templates from a YAML catalogue covering: weather (tornado warning, flash flood, winter storm), fire (wildfire evacuation, structure fire), hazmat (shelter-in-place, evacuation), shelter (open, close, capacity update), medical (mass casualty staging, hospital divert), public safety (active shooter, missing person), infrastructure (power outage, water main).
- Templates support placeholders: `{{incident_name}}`, `{{location}}`, `{{instruction}}` etc.

**Testing**:
- `Unit: render template with context → placeholders substituted`
- `Unit: missing required placeholder in context → ValueError`
- `Integration: seed templates count >= 150`

#### 6.2 — Audience targeting

**What**: Audience model supporting roles, groups, geographic targeting (polygon and radius), and contact lists.

**Design**:
- `target_type` field on `notification` plus `target_criteria` JSONB:
  - `all`: tenant-wide
  - `role`: `{role_ids: [...]}`
  - `group`: `{group_ids: [...]}`
  - `geo_polygon`: uses `notification.geo_target` polygon
  - `geo_radius`: `{lat, lon, radius_km}`
  - `incident`: all assigned to incident
- `_resolve_audience(notification) -> list[NotificationRecipient]` materialises recipients at send time.
- Geo targeting uses PostGIS: `ST_DWithin(user.location, polygon, 0)` against opted-in mobile users' last known location.

**Testing**:
- `Unit: role targeting resolves only users with role`
- `Unit: geo polygon targeting matches users inside polygon`
- `Unit: geo radius targeting matches users within km`
- `Integration: notification with 0 resolved recipients → 422 "no audience"`

#### 6.3 — Notification dispatch with channel abstraction

**What**: Channel-agnostic dispatch service that fans out to per-channel adapters.

**Design**:
- `NotificationChannel` protocol:
  ```python
  class NotificationChannel(Protocol):
      channel: str  # 'sms', 'email', 'voice', 'push'
      async def send(self, recipient: NotificationRecipient, body: str, subject: str | None) -> ChannelResult: ...
  ```
- Adapters: `TwilioSMS`, `TwilioVoice`, `SendGridEmail`, `FCMPush`, `MailpitEmail` (dev), `LogChannel` (test).
- Channel selected per recipient by available contact info + tenant config.
- Celery task `dispatch_notification.delay(notification_id)`:
  1. Resolves recipients.
  2. Inserts `notification_recipient` rows in pending state.
  3. Dispatches per-recipient subtasks `send_to_recipient.delay(recipient_id)`.
  4. Each subtask invokes channel adapter, updates `status`, `delivered_at`, `read_at` (via webhook).
- Twilio status webhooks: `POST /v1/webhooks/twilio/status` updates `notification_recipient`.
- Rate limiting: configurable per-tenant cap on sends/minute.

**Testing**:
- `Unit: channel selection prefers SMS if phone present, else email`
- `Integration (mocked Twilio): send 100 SMS → 100 recipient rows updated`
- `Integration: Twilio status webhook with delivered status → recipient row updated`
- `Integration: rate limit triggers backoff; no sends dropped`
- `Standards: OWASP A03 — webhook signature validated using Twilio X-Twilio-Signature`

#### 6.4 — Two-way responses and confirmations

**What**: Inbound SMS/email replies map back to recipient row; structured response codes for confirmations and surveys.

**Design**:
- Inbound SMS webhook `POST /v1/webhooks/twilio/inbound` correlates by recipient phone + most recent notification.
- Inbound email via SendGrid Inbound Parse webhook.
- Templates can include response prompts: `Reply YES to confirm, NO to decline, HELP to request assistance`.
- Free-text responses stored in `notification_recipient.response`.
- Help requests automatically create an `incident_log_entry` with type `communication` and emit `HelpRequested` event for operator alerting.

**Testing**:
- `Integration: inbound SMS "YES" → recipient response captured + responded_at set`
- `Integration: inbound SMS "HELP" → HelpRequested event emitted; incident log entry created`
- `Integration: inbound email reply maps to correct recipient via subject-line token`

#### 6.5 — CAP alert authoring and serialisation

**What**: Author CAP v1.2 alerts and produce validated CAP XML (full IPAWS submission handled in Phase 7).

**Design**:
- `POST /v1/incidents/{id}/cap-alerts`:
  ```python
  class CAPAlertCreate(BaseModel):
      status: Literal["Actual","Exercise","System","Test","Draft"]
      msg_type: Literal["Alert","Update","Cancel","Ack","Error"]
      scope: Literal["Public","Restricted","Private"]
      restriction: str | None = None
      info: list[CAPInfoCreate]  # at least one
      references: list[CAPReference] | None = None
  ```
- `CAPInfoCreate` includes all required CAP fields per OASIS spec.
- `services/cap/serialiser.py` produces CAP v1.2 XML using `lxml`, validated against the OASIS XSD `CAP-v1.2.xsd` bundled in `standards/schemas/`.
- IPAWS profile validation: additional rules enforced (sender format, expires required, response_type required for Public scope).
- `GET /v1/cap-alerts/{id}/xml` returns the validated XML; `GET /v1/cap-alerts/{id}.json` returns JSON serialisation.
- `identifier` auto-generated: `{tenant.slug}-{incident.incident_number}-{seq}-{yyyymmddTHHMMSS}`.

**Testing**:
- `Standards: XML output validates against CAP-v1.2.xsd`
- `Standards: IPAWS profile validation — Public scope without response_type → 422`
- `Unit: cancel message includes references to prior alert identifier`
- `Unit: round-trip CAP XML → parse → serialise produces equivalent XML`
- `Integration: CAPAlertSent event emitted on successful publish`

---

## Phase 7: AI-Native Capabilities

### Purpose
Activate the differentiating AI features: situation summary generation, ICS form auto-drafting from narrative, CAP alert draft generation, AI-curated common operating picture, and AAR drafting. After Phase 7, operators can dictate or paste a narrative and receive structured drafts they approve or revise.

### Tasks

#### 7.1 — LLM client abstraction and AI Gateway

**What**: Pluggable LLM client supporting Anthropic Claude (default), OpenAI, and Vercel AI Gateway, with prompt caching and audit logging of all calls.

**Design**:
- `services/ai/client.py`:
  ```python
  class LLMClient(Protocol):
      async def complete(
          self,
          system: str,
          messages: list[Message],
          tools: list[ToolSpec] | None = None,
          response_schema: type[BaseModel] | None = None,
          cache_breakpoints: list[int] | None = None,
      ) -> LLMResponse: ...
  ```
- Implementations: `AnthropicClient`, `OpenAIClient`, `GatewayClient`.
- All calls logged to `ai_call_log` table (tenant_id, model, tokens_in, tokens_out, latency_ms, prompt_hash, response_hash, user_id, intent).
- Prompt caching enabled for static system prompts and reference data (ICS form schemas).
- Per-tenant budget caps; surfaced via `GET /v1/tenants/{id}/ai-usage`.

**Testing**:
- `Unit: structured output via response_schema → returns validated Pydantic model`
- `Unit: cache_breakpoints applied to Anthropic call`
- `Integration (recorded responses via vcrpy): each implementation passes a shared contract test`
- `Integration: budget exceeded → service raises BudgetExceeded; no further calls`

#### 7.2 — Natural-language incident logger

**What**: Convert dictated/typed responder narrative into a structured `incident_log_entry` plus suggested updates to ICS-201/202/209 fields.

**Design**:
- `POST /v1/incidents/{id}/ai/narrate`:
  ```python
  class NarrationCreate(BaseModel):
      narrative: str
      audio_url: str | None = None  # if audio, transcribe first
      preview_only: bool = True
  ```
- If audio: transcribe via Whisper / OpenAI Audio API.
- LangGraph flow:
  1. `classify` — determine intent (situation update, resource request, status change, etc.).
  2. `extract` — structured extraction against schema (location, severity, resource needs, casualties).
  3. `suggest_form_updates` — proposed deltas to ICS-201 situation summary, ICS-209 status fields.
  4. `review` — human approves/edits in UI before commit.
- If `preview_only=true`, returns proposals; if false, commits log entry + form updates after explicit approval.
- All AI proposals tagged `source='ai'` in log entries.

**Testing**:
- `Unit (mocked LLM): narrative "Engine 51 arrived on scene at 14:23" → extracts resource arrival event`
- `Unit (mocked LLM): narrative with casualties → ICS-209 injuries field proposed`
- `Integration: preview_only=true returns proposals without persisting`
- `Integration: AI-generated log entries have source='ai' and link to ai_call_log`

#### 7.3 — AI-assisted ICS form drafting

**What**: Per-form-type drafting endpoints that take context (incident state, prior forms, latest situational updates) and produce a draft form.

**Design**:
- `POST /v1/incidents/{id}/forms/ics-209/draft` — generates draft ICS-209 from current incident state + prior approved ICS-209 (delta-aware).
- LangGraph flow per form type:
  1. `gather_context` — fetch incident, last operational period, latest situation report, deployed resources, log entries since last form version.
  2. `draft` — LLM call with form's JSON Schema as structured output target.
  3. `validate` — schema validation; if invalid, retry once with error feedback.
  4. `return draft` — never auto-submits.
- Drafts have `status='draft'` and `source='ai'` (new column on `ics_form`).
- Operator reviews/edits, then transitions normally via existing form endpoints.

**Testing**:
- `Unit (mocked LLM): ICS-209 draft includes percent_contained extracted from fire perimeter feed`
- `Standards: draft output validates against the form's JSON Schema`
- `Integration: ICS-202 draft includes objectives generated from incident type + prior objectives`
- `Integration: invalid LLM output → retried once with feedback`

#### 7.4 — CAP alert draft generation

**What**: Generate CAP-compliant alert drafts in multiple languages from a single operator narrative.

**Design**:
- `POST /v1/incidents/{id}/ai/cap-draft`:
  ```python
  class CAPDraftRequest(BaseModel):
      narrative: str
      languages: list[str] = ["en-US"]  # ISO 639 + region
      response_type: list[str] | None = None  # auto if absent
      channels: list[Literal["wea","eas","web","mobile"]] = ["wea","web"]
  ```
- LangGraph flow:
  1. `classify_hazard` — extract event type, severity, urgency, certainty.
  2. `infer_area` — extract or geocode the affected area.
  3. `draft_per_language` — generate one `cap_info` block per language.
  4. `validate` — produce CAP XML, validate against XSD + IPAWS profile.
  5. `return` — returns draft `cap_alert` with `status='Draft'`.
- Operator reviews; on approve, status transitions and (Phase 7.5) IPAWS submission triggers.

**Testing**:
- `Unit (mocked LLM): narrative for tornado warning → CAP event="Tornado Warning", severity="Extreme", urgency="Immediate"`
- `Standards: generated XML for each language passes XSD validation`
- `Integration: multi-language draft has cap_info rows per requested language`
- `Integration: AAR generated from prior incident does not duplicate prior approved drafts`

#### 7.5 — IPAWS submission

**What**: Submit approved CAP alerts to IPAWS-OPEN, handle responses, and track lifecycle.

**Design**:
- IPAWS submission uses HTTPS POST to FEMA's IPAWS-OPEN service with SOAP envelope containing CAP XML plus IPAWS COG credentials.
- `services/ipaws/client.py` handles auth (mutual TLS with COG certificate), submission, and response parsing.
- New table `cap_submission`:
  ```sql
  CREATE TABLE cap_submission (
      id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
      alert_id UUID NOT NULL REFERENCES cap_alert(id),
      destination TEXT NOT NULL,  -- 'IPAWS', 'NWEM', 'WEA', etc.
      status TEXT NOT NULL,       -- 'submitted', 'accepted', 'rejected', 'expired'
      submitted_at TIMESTAMPTZ NOT NULL,
      ipaws_response JSONB,
      error_message TEXT
  );
  ```
- Test mode: tenant config flag `ipaws_test_mode=true` routes to IPAWS test environment.
- All submissions audited; rejection reasons surfaced to operator.

**Testing**:
- `Integration (mocked IPAWS): approve CAP alert → submission row created with status='submitted'`
- `Integration (mocked IPAWS): IPAWS rejects with error → status='rejected', error_message captured`
- `Integration: IPAWS test mode flag selects test endpoint`
- `Manual checklist: real-world IPAWS-OPEN sandbox submission (gated by tenant config)`

#### 7.6 — AAR auto-generation

**What**: Draft After-Action Report from incident chronology, forms, resource deployments, and log entries.

**Design**:
- `POST /v1/incidents/{id}/ai/aar-draft` — only allowed when `incident.status='closed'`.
- LangGraph flow:
  1. `gather` — chronology (all log entries), all approved forms, resource deployments, notifications sent, CAP alerts issued, cost records.
  2. `summarise_timeline` — narrative timeline.
  3. `extract_strengths_improvements` — structured extraction; identifies decisions, delays, resource gaps.
  4. `recommendations` — generates actionable recommendations.
- Produces an `after_action_report` row + `improvement_plan_item` rows (status='open') for review.

**Testing**:
- `Unit (mocked LLM): closed incident → AAR draft with non-empty timeline + 3+ recommendations`
- `Unit: incident.status != 'closed' → 409`
- `Integration: improvement plan items created and visible via GET /v1/incidents/{id}/aar/items`

#### 7.7 — MCP server

**What**: Expose incident state and AI tools via Model Context Protocol so external AI agents (Claude Desktop, Cursor, custom agents) can query and act.

**Design**:
- `mcp_server/` package implements an MCP server using the official Python SDK.
- Tools:
  - `list_active_incidents() -> [IncidentSummary]`
  - `get_incident(id) -> IncidentDetail`
  - `query_resources(filters) -> [Resource]`
  - `draft_cap_alert(incident_id, narrative, language) -> CAPDraft`
  - `draft_ics_form(incident_id, form_type) -> FormDraft`
  - `add_log_entry(incident_id, narrative) -> LogEntry`
- Resources (read-only context): incident chronology, current COP snapshot.
- Auth via tenant-scoped MCP API token; bound to a service account user.
- Transport: stdio (local) and SSE (network); both authenticated.
- Documented at `docs/mcp.md` with Claude Desktop config example.

**Testing**:
- `Integration: MCP client lists tools and resources`
- `Integration: invoke list_active_incidents returns expected JSON`
- `Integration: MCP token scoped to tenant — cannot read other tenants`
- `Integration: invoke draft_cap_alert returns valid CAP draft`

---

## Phase 8: Identity Federation, Compliance & Hardening

### Purpose
Add the enterprise/government-grade identity, security, and compliance features required for procurement: SAML/OIDC federation, role-based access control refinement, FedRAMP-aligned NIST 800-53 controls, SOC 2 readiness instrumentation, and OWASP-aligned hardening.

### Tasks

#### 8.1 — SAML 2.0 federation

**What**: Integrate `python3-saml` for SAML SSO with state and federal identity providers.

**Design**:
- `GET /v1/auth/saml/{tenant_slug}/metadata` — service provider metadata XML.
- `GET /v1/auth/saml/{tenant_slug}/login` — initiates SAML AuthnRequest, redirects to IdP.
- `POST /v1/auth/saml/{tenant_slug}/acs` — assertion consumer service; validates SAML response, provisions or updates user, issues JWT.
- Per-tenant `saml_config` table: IdP metadata URL, entity ID, attribute mappings, signing certs.
- Just-in-time provisioning with attribute-to-role mapping.

**Testing**:
- `Integration: configure mock IdP (saml2-test); full login flow → JWT issued`
- `Integration: signed assertion with tampered payload → 401`
- `Standards: SAML response validates against XSD; signature verification mandatory`

#### 8.2 — OIDC federation

**What**: Add OIDC support alongside SAML for modern IdPs (Azure Entra ID, Okta, Google Workspace).

**Design**:
- `Authlib`-based OIDC client.
- Per-tenant `oidc_config` table: issuer URL, client ID/secret, scopes.
- Standard authorization code + PKCE flow.
- Token introspection; userinfo endpoint queried for role mapping.

**Testing**:
- `Integration (mock IdP): authorization code flow → JWT issued`
- `Integration: id_token signature validated against JWKS`
- `Integration: refresh of OIDC token rotates session`

#### 8.3 — Fine-grained RBAC

**What**: Replace coarse role checks with permission-based authorisation enforced via a single `require_permission()` dependency.

**Design**:
- Permissions encoded as `domain.action` strings: `incident.create`, `incident.update.status`, `cap.publish`, `resource.request.approve`, `ics-form.approve`, `audit.read`, `tenant.admin`.
- Roles map to permission sets; tenants can define custom roles.
- Some permissions context-scoped (e.g., `ics-form.approve` requires the user to hold an active assignment to a position with approval authority for the form type).
- Endpoint dependency:
  ```python
  @router.post("/cap-alerts/{id}/publish", dependencies=[Depends(require_permission("cap.publish"))])
  ```

**Testing**:
- `Unit: user with permission → allowed; without → 403`
- `Unit: context-scoped permission — approve ICS-204 requires Operations Section Chief assignment`
- `Integration: permission audit log records authorisation decisions`

#### 8.4 — Encryption, secrets, and key management

**What**: At-rest encryption for sensitive columns, integration with a secrets manager, and key rotation hooks.

**Design**:
- Sensitive columns (`data_feed.auth_config`, `saml_config.signing_cert_private`, `oidc_config.client_secret`, `notification_recipient.destination` for unsubscribed users) encrypted using `cryptography` Fernet with a tenant-scoped data key.
- Data keys envelope-encrypted by a KMS master key (AWS KMS, GCP KMS, HashiCorp Vault, or local Vault dev).
- `secrets.py` provides `get_secret(name) -> SecretStr`; backends pluggable.
- DB connection strings, JWT secrets, provider credentials never logged.

**Testing**:
- `Unit: round-trip encryption per tenant; tenant A key cannot decrypt tenant B ciphertext`
- `Integration: secret rotation re-wraps DEKs without data loss`
- `Standards: NIST 800-53 SC-13 (cryptographic protection) controls satisfied per checklist`

#### 8.5 — OWASP hardening pass

**What**: Systematic OWASP Top 10 review with mitigations and tests.

**Design**:
- A01 (Broken Access Control): RLS + permission tests; IDOR audit (all `:id` paths verify tenant ownership).
- A02 (Cryptographic Failures): TLS-only; HSTS; modern cipher suites.
- A03 (Injection): SQLAlchemy parameterised queries; XML parsed with `defusedxml`; XSS escaping in templates.
- A04 (Insecure Design): rate limits per endpoint via `slowapi`; multi-factor auth for tenant admins.
- A05 (Security Misconfiguration): production checks at boot — fail if `secret_key` is default, if CORS is `*`, if debug enabled.
- A07 (ID/Auth Failures): argon2 with rate-limited login; account lockout after 10 failures; password complexity policy.
- A08 (Data Integrity Failures): signed JWTs; webhook signature verification (Twilio, SendGrid, FCM).
- A09 (Logging Failures): structured logs of auth events; tamper-evident audit trail via append-only table + periodic hash chain.
- A10 (SSRF): outbound HTTP via allowlist for feed adapters; private IP block.

**Testing**:
- `Security: ZAP baseline scan in CI`
- `Security: bandit + ruff security rules in pre-commit`
- `Unit: IDOR test for every `:id` endpoint (parametrised test generator)`
- `Unit: rate limit triggers 429 after N requests`
- `Integration: outbound HTTP to 169.254.169.254 blocked`

#### 8.6 — Observability and compliance reporting

**What**: OpenTelemetry traces, metrics, logs; SOC 2 / NIST 800-53 evidence collection.

**Design**:
- OpenTelemetry SDK exports to OTLP collector (Tempo/Loki/Prometheus).
- Per-tenant dashboards: notification delivery success rate, AI cost, feed health, form approval latency.
- Compliance evidence endpoints (admin-only):
  - `GET /v1/compliance/audit-summary?since=...` — counts of auth events, permission denials, admin actions
  - `GET /v1/compliance/data-export?tenant_id=...` — full tenant data export (GDPR/CCPA support)
- Backup and restore procedure documented; automated nightly logical backups.

**Testing**:
- `Integration: traces propagate through API → service → DB`
- `Integration: audit summary returns counts consistent with audit_trail rows`
- `Integration: data export produces a tarball with all tenant entities`

---

## Phase 9: Web Dashboard & Mobile Field App

### Purpose
Deliver the user-facing experiences: a Next.js web EOC dashboard with the map-first common operating picture, ICS form editors, notification composer, and reports; plus a React Native field app for responders with offline tolerance.

### Tasks

#### 9.1 — Auth and shell (web)

**What**: Login (local, SAML, OIDC), role-aware navigation shell, tenant switcher.

**Design**:
- App Router layout: `(auth)` group for unauthenticated routes; `(app)` for authenticated.
- Auth via NextAuth or custom JWT flow; refresh token rotation in HTTP-only cookie.
- Layout: left sidebar (Incidents, Map, Notifications, Resources, Reports, Admin), top bar (tenant switcher, user menu, status indicators).
- React Server Components used for initial dashboard render; client components for interactive map and forms.

**Testing**:
- `E2E (Playwright): login → land on dashboard → see correct nav for role`
- `E2E: refresh expired token in background; user not logged out mid-session`
- `Unit: permission-based nav items hidden for users without permission`

#### 9.2 — Incident dashboard

**What**: Main dashboard listing active incidents with summary cards.

**Design**:
- Card per incident: name, type, severity, status, IC assigned, last update.
- Real-time updates via SSE on `/v1/incidents/stream`.
- Quick-create incident dialog with templates per incident type.
- Filter sidebar: status, type, severity, time range, jurisdiction.

**Testing**:
- `E2E: create incident from dashboard → appears in list within 1s`
- `E2E: filter by severity → only matching cards visible`
- `Unit: incident card snapshot matches design`

#### 9.3 — Common Operating Picture map

**What**: Full-screen MapLibre GL map showing incidents, resources, zones, markers, feed events; with layer toggles.

**Design**:
- MapLibre with vector basemap (configurable tile URL); GeoJSON sources fetched from `/v1/cop?bbox=...`.
- Layer panel: Incidents (markers), Resources (clustered), Zones (semi-transparent fills), Weather (NWS alerts), External GIS (WFS layers), CAP alerts (geo-targeted areas).
- Click incident marker → side panel with incident detail, link to full view.
- Drawing tools: polygon (zone), point (marker), measure distance.
- SSE-driven layer refresh on COP updates.

**Testing**:
- `E2E: map renders with seed incident; click marker shows side panel`
- `E2E: toggle Weather layer → NWS alert polygon visible`
- `E2E: draw evacuation zone → POSTs polygon; appears as new layer`
- `Unit: GeoJSON parsing handles malformed input gracefully`

#### 9.4 — ICS form editors

**What**: Per-form-type editor components for ICS-201, 202, 204, 205, 206, 209, 215, 215A, 218.

**Design**:
- Each form is a React Server Component with embedded client component for editing.
- WebSocket-based collaborative editing per Phase 3.4.
- Auto-save with debouncing; presence avatars in form header.
- "Generate Draft with AI" button → calls `/v1/.../draft` endpoint → diff view for accept/reject.
- "Submit for Review" → "Approve" buttons gated by permissions.
- Print/Export PDF → calls IAP compile.

**Testing**:
- `E2E: open ICS-209 → edit field → reload → field persisted`
- `E2E: two browser windows editing same form → updates sync in real time`
- `E2E: click "Generate Draft" → diff view appears; accept → form updated`
- `E2E: submit for review → reviewer sees pending in queue`

#### 9.5 — Notification composer

**What**: UI for selecting templates, targeting audience (including drawing geo polygon on map), previewing, and dispatching.

**Design**:
- Wizard: template selection → message customisation → audience targeting → review → send.
- Audience targeting tab embeds a MapLibre instance for drawing polygons or radius.
- Preview shows estimated recipient count and per-channel breakdown.
- Send button gated by `notification.send` permission; confirm dialog for sends >1000 recipients.
- Live status panel after send: delivery progress bar, per-channel success rates, opt-out tracking.

**Testing**:
- `E2E: pick template → fill placeholders → target by role → send → live status shows progress`
- `E2E: draw polygon → estimated recipient count > 0`
- `E2E: send with 0 recipients → 422 surfaced as inline error`

#### 9.6 — CAP alert composer

**What**: Specialised UI for authoring CAP alerts with IPAWS submission flow.

**Design**:
- Single-page editor with CAP field groups: alert metadata, info per language, area (polygon drawn on map), response type.
- AI assist sidebar: paste narrative → "Draft CAP" generates structured fields.
- Validation badges: CAP XSD valid, IPAWS profile valid.
- Submit flow: Draft → Approver review → Approve → Submit to IPAWS (if configured) → confirmation.

**Testing**:
- `E2E: paste narrative → AI draft → review fields → approve → submission row created`
- `E2E: invalid CAP (missing required field) → submit button disabled with error tooltips`
- `E2E: multi-language draft → tabs per language with separate edit views`

#### 9.7 — Resource management UI

**What**: Resource catalogue list/grid with map view, request workflow, and deployment tracking.

**Design**:
- Resource list with filters: type, status, capability, location radius.
- Map toggle plotting resource current locations.
- Request workflow: select resource type, quantity, justification, delivery location.
- Pending approvals queue for Logistics Section Chief role.
- Mutual-aid view: own resources + partner agencies, with badge for cross-tenant origin.

**Testing**:
- `E2E: file resource request → approver receives in queue → approve → resource status updates`
- `E2E: filter by capability shows only matching resources`

#### 9.8 — Reports and AAR view

**What**: Incident chronology, situation reports, AAR drafting/editing, compliance exports.

**Design**:
- Chronology view: time-descending log entries, filterable by entry type.
- AAR editor: AI-generated draft with edit-in-place sections (Executive Summary, Timeline, Strengths, Improvements, Recommendations).
- Improvement Plan tracker with assignee, due date, status.
- FEMA reimbursement export: cost records summarised by FEMA category, PDF + CSV.

**Testing**:
- `E2E: closed incident → generate AAR draft → edit recommendations → publish`
- `E2E: export FEMA reimbursement PDF → file downloads with expected sections`

#### 9.9 — Mobile field app (React Native)

**What**: Expo-based app for field responders with offline tolerance for log entries, resource requests, location sharing, and form view.

**Design**:
- Tabs: Active Incident, Map, Resources, Notifications.
- Offline queue: SQLite-backed outbox; syncs when connectivity returns. Idempotency keys on all POST requests.
- Location sharing: background location updates posted to `/v1/users/me/location`.
- Push notifications via FCM/APNs.
- Quick log entry: voice memo → backend transcription → narrative entry.
- Read-only ICS form viewer (full editing reserved for web).

**Testing**:
- `E2E (Expo Detox): create log entry offline → toggle online → entry appears in backend within 5s`
- `E2E: receive push notification → tap → opens relevant incident`
- `Unit: outbox replays in order; failed requests retried with backoff`

---

## Phase 10: Packaging, Documentation & Open-Source Release

### Purpose
Make the platform genuinely usable by external adopters: production-ready Docker/Helm packaging, comprehensive documentation, sample data and tutorials, contribution guides, security disclosure policy, and a public reference deployment.

### Tasks

#### 10.1 — Production Docker and Helm

**What**: Production-grade container images and Helm chart suitable for self-hosted deployment.

**Design**:
- Multi-stage Dockerfile producing distroless runtime images.
- Helm chart with: postgres + postgis (Bitnami subchart or external), redis, minio, backend (HPA-ready), web, celery workers, celery beat, MCP server, OpenTelemetry collector, ingress.
- Values file documents all configurable knobs (LLM provider, notification providers, IPAWS COG cert mount, etc.).
- Database migration `Job` runs `alembic upgrade head` on chart install/upgrade.
- Backup CronJob: nightly logical pg_dump to S3.

**Testing**:
- `Integration: helm install on kind cluster; smoke check /health passes`
- `Integration: helm upgrade between two versions; migrations applied; no downtime for stateless services`
- `Integration: backup CronJob produces a restorable dump`

#### 10.2 — Documentation site

**What**: Docusaurus or MkDocs site covering architecture, deployment, API reference, standards mapping, and runbooks.

**Design**:
- Sections: Getting Started, Concepts (Incidents, ICS, CAP), Deployment (Docker, Helm, FedRAMP considerations), Configuration, API Reference (rendered from OpenAPI), Standards Mapping (per spec → field mapping), MCP integration, AI Configuration, Operations Runbooks (backups, incident response, scaling).
- Hosted on GitHub Pages; auto-built on main branch push.

**Testing**:
- `Integration: docs site builds without errors in CI`
- `Integration: OpenAPI page reflects latest spec`

#### 10.3 — Demo seed and tutorial scenario

**What**: One-command demo deployment that seeds a realistic wildfire incident with prepopulated forms, resources, and a guided walkthrough.

**Design**:
- `make demo` runs docker-compose and executes `scripts/seed_demo_incident.py`.
- Seeds: Larimer County tenant, sample wildfire incident (Glacier Creek Fire), 3 operational periods, approved ICS-201/202/204/209, deployed engines + crews, two CAP test alerts, simulated NWS feed events.
- Tutorial walkthrough document with screenshots showing: log in, view COP, draft ICS-209 with AI, dispatch notification, generate AAR.

**Testing**:
- `Integration: make demo completes; visiting localhost:3000 shows seeded data`
- `Manual: tutorial walkthrough completes end-to-end without errors`

#### 10.4 — Open-source governance

**What**: License (Apache 2.0), CONTRIBUTING, CODE_OF_CONDUCT, SECURITY.md, issue/PR templates, semantic-release.

**Design**:
- `LICENSE`: Apache 2.0 text.
- `CONTRIBUTING.md`: dev setup, commit conventions (Conventional Commits), DCO sign-off, code review process.
- `CODE_OF_CONDUCT.md`: Contributor Covenant 2.1.
- `SECURITY.md`: GitHub security advisory disclosure process, PGP key, 90-day disclosure window.
- Issue templates: Bug, Feature Request, Security (private), Standards Compliance Issue.
- `semantic-release` for backend and web with conventional commits triggering version bumps.

**Testing**:
- `Manual: contribution flow validated by external dry-run contributor`
- `Integration: semantic-release dry run produces expected version bumps`

#### 10.5 — Reference deployment and benchmark

**What**: Public reference deployment (read-only demo) and load-test benchmark.

**Design**:
- Reference deployment hosted on Fly.io / Render / DigitalOcean App Platform with seeded demo data.
- `k6` or `locust` script simulates: 100 concurrent operators, 1000 incidents, 10k notifications/min, 50 simultaneous SSE COP subscribers.
- Documented hardware sizing recommendations per agency tier (small county, large state, federal).

**Testing**:
- `Performance: 95p API latency < 250ms at target load`
- `Performance: notification dispatch throughput >= 5000/min on single worker`
- `Performance: SSE backplane sustains 1000 concurrent subscribers without dropped events`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation, Tenancy, Reference Data
    │
    ├── Phase 2: Incident Lifecycle & Chronology
    │       │
    │       ├── Phase 3: ICS Command Structure & Forms      ─┐
    │       │                                                │
    │       ├── Phase 4: Resource Management                 ├── can be developed in parallel
    │       │                                                │
    │       ├── Phase 5: GIS & Situational Awareness         ─┘
    │       │
    │       ├── Phase 6: Mass Notification & CAP             ── requires Phase 5 (geo targeting)
    │       │           │
    │       │           └── Phase 7: AI-Native Capabilities  ── requires Phases 3, 4, 5, 6
    │       │
    │       └── Phase 8: Identity, Compliance, Hardening     ── partially parallel with 3-7 (RBAC needed by 6.x)
    │
    ├── Phase 9: Web Dashboard & Mobile App                  ── can begin once Phase 2 APIs stable;
    │                                                            mobile sub-phase can wait until Phase 6
    │
    └── Phase 10: Packaging, Docs, Open-Source Release       ── final; spans all prior phases
```

Parallelism opportunities:
- After Phase 2, three teams can work in parallel on Phase 3 (ICS forms), Phase 4 (resources), and Phase 5 (GIS/feeds).
- Phase 8 RBAC scaffolding (8.3) should land before Phase 6 to avoid retrofitting permissions; SAML/OIDC (8.1, 8.2) can run later.
- Phase 9 web work can begin in parallel with Phase 3+ once Phase 2 APIs are stable (against staging fixtures).
- Phase 10 packaging can begin in parallel with Phase 8/9 if it tracks the current main branch.

---

## Definition of Done (per phase)

Every phase must satisfy all of the following before being considered complete:

1. All tasks in the phase implemented per their Design section.
2. All unit, integration, and standards conformance tests for the phase pass locally and in CI.
3. `ruff format`, `ruff check`, and `mypy --strict` pass on `services/backend`.
4. `biome check` and `tsc --strict` pass on `apps/web` (and `apps/mobile` from Phase 9.9 onward).
5. `pytest --cov` overall coverage for the phase's modules >= 80%; standards-critical modules (CAP, EDXL, NIEM) >= 95%.
6. Docker images build cleanly; `docker compose up` boots the affected services.
7. Each new API endpoint appears in `/openapi.json` with example request/response and is reflected in the generated TypeScript client.
8. New configuration options (env vars, settings) documented in `docs/configuration.md`.
9. Database migrations created via Alembic, reversible (down-migration tested), and applied cleanly to a fresh DB.
10. New JSONB or extension fields backed by a JSON Schema in `standards/schemas/` with validation enforced.
11. New external integrations documented with required credentials, rate limits, and failure modes.
12. Phase-level smoke test passes end-to-end (e.g., for Phase 3: create incident → assign IC → create ICS-201 → approve → IAP renders).
13. ADR (Architecture Decision Record) added to `docs/adr/` for any non-obvious technical decision made during the phase.
14. Security review item: any new auth-touching code paths reviewed against OWASP Top 10; any new outbound HTTP added to allowlist.
15. Changelog entry under `## Unreleased` per Keep a Changelog format.
