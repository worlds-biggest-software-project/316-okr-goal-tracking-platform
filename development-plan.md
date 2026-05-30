# OKR & Goal Tracking Platform — Phased Development Plan

> Project: 316-okr-goal-tracking-platform · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the three `data-model-suggestion-*.md` files. The data model follows **Suggestion 1 (Entity-Centric Normalized Relational)** as the structural backbone — chosen because the MVP requires database-enforced cascading alignment, RBAC, and SCIM provisioning, all of which benefit from explicit foreign keys. It selectively borrows the **JSONB hybrid** approach from Suggestion 2 for fast-evolving data (AI analysis output, provider-specific integration config), keeping those columns schema-free to avoid migration churn.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | Python 3.12 | The differentiating features (AI authoring, quality scoring, risk prediction, narrative generation, MCP server) are LLM-heavy. Python has the richest LLM/AI ecosystem (Anthropic SDK, official MCP SDK) and strong async support for connector polling. |
| API framework | FastAPI | Generates OpenAPI 3.1 automatically (required by standards.md), native Pydantic v2 validation, async-first for connector and webhook workloads, and integrates cleanly with the MCP server. |
| Data validation | Pydantic v2 | Request/response schemas double as the JSON Schema source for the OpenAPI spec and as validators for JSONB payloads (AI output, integration config). |
| Database | PostgreSQL 16 | The normalized model needs recursive CTEs for alignment trees, `JSONB` + GIN indexes for AI/integration fields, `TIMESTAMPTZ`, array columns (scopes, tags), and partial indexes. SQLite cannot support these. |
| ORM / migrations | SQLAlchemy 2.0 (async) + Alembic | Mature async ORM; Alembic gives reproducible, reviewable migrations — one per phase. |
| Task queue | Celery + Redis | Connector polling, scheduled check-in reminders, AI analysis, and narrative generation are async/long-running. Celery beat handles the cron-style reminder cadence. |
| Cache / broker | Redis 7 | Doubles as Celery broker/result backend and a cache for dashboard aggregates and rate-limit counters. |
| LLM provider | Anthropic Claude (via `anthropic` SDK) | AI-native positioning. Claude Sonnet for authoring/quality/risk; abstracted behind an `LLMClient` interface so the provider is swappable. |
| MCP server | `mcp` (official Python SDK) | standards.md identifies an MCP server as a key differentiator — lets Claude/Copilot read OKRs, surface at-risk KRs, and draft check-ins. |
| Auth (app) | OAuth 2.0 / OIDC (Authlib) + JWT (PyJWT) | RFC 6749/6750/7519 compliance; enterprise SSO via OIDC. |
| Auth (enterprise SSO) | SAML 2.0 (`python3-saml`) | Legacy enterprise IdPs (per standards.md) require SAML; complements OIDC. |
| Provisioning | SCIM 2.0 (RFC 7643/7644) custom endpoints | "The enterprise gate" per standards.md notes — must be designed early. |
| Frontend | React 18 + TypeScript + Vite, TanStack Query, Tailwind | SPA dashboard consuming the REST API. Alignment trees and progress charts are highly interactive; SPA is the right fit. Recharts for charts, react-flow for the alignment tree. |
| Notifications | Slack incoming webhooks + SMTP (email) | features.md MVP requires Slack + email; follows Slack's documented incoming-webhook JSON format. |
| Containerisation | Docker + docker-compose | Self-hosted deployment mode (README). One compose file brings up API, worker, Postgres, Redis, frontend. |
| Testing | pytest + pytest-asyncio + httpx + testcontainers | Async test support; testcontainers spins up real Postgres/Redis for integration tests. Vitest + Playwright for frontend. |
| Code quality | ruff (lint+format) + mypy (strict) | Single fast linter/formatter; static typing catches schema drift. |
| Package manager | uv | Fast, reproducible Python dependency resolution and locking. |
| Key libraries | `anthropic`, `mcp`, `celery`, `redis`, `sqlalchemy`, `alembic`, `authlib`, `pyjwt`, `python3-saml`, `jira`, `PyGithub`, `simple-salesforce`, `openpyxl` (Excel), `weasyprint` (PDF), `slack_sdk` | Domain-specific: official connector SDKs, export libraries, notification SDK. |

### Project Structure

```
okr-platform/
├── pyproject.toml
├── uv.lock
├── Dockerfile
├── docker-compose.yml
├── alembic.ini
├── .env.example
├── README.md
├── openapi.json                      # generated, committed for SDK/clients
├── migrations/                       # Alembic
│   ├── env.py
│   └── versions/
├── src/okr/
│   ├── __init__.py
│   ├── main.py                       # FastAPI app factory, router mounting
│   ├── config.py                     # Pydantic Settings (env-driven)
│   ├── db.py                         # async engine, session dependency
│   ├── celery_app.py                 # Celery app + beat schedule
│   ├── models/                       # SQLAlchemy ORM models
│   │   ├── org.py  team.py  user.py  cycle.py
│   │   ├── objective.py  key_result.py  check_in.py  initiative.py
│   │   ├── comment.py  data_source.py  ai.py  notification.py  api_key.py
│   ├── schemas/                      # Pydantic request/response + JSONB payloads
│   ├── api/                          # FastAPI routers
│   │   ├── deps.py                   # auth, db, rbac dependencies
│   │   ├── objectives.py  key_results.py  check_ins.py
│   │   ├── cycles.py  teams.py  users.py  comments.py
│   │   ├── alignment.py  dashboards.py  exports.py
│   │   ├── integrations.py  webhooks.py  ai.py
│   │   ├── auth.py  scim.py  saml.py  api_keys.py
│   ├── services/                     # business logic (framework-agnostic)
│   │   ├── progress.py               # progress/status computation
│   │   ├── alignment.py              # tree traversal + roll-up
│   │   ├── rbac.py                   # permission checks
│   │   ├── notifications.py
│   │   ├── export_pdf.py  export_excel.py
│   ├── integrations/                 # connector framework
│   │   ├── base.py                   # Connector protocol
│   │   ├── jira.py  github.py  salesforce.py  webhook.py
│   │   ├── registry.py  sync.py      # Celery sync tasks
│   ├── ai/                           # LLM features
│   │   ├── client.py                 # LLMClient abstraction
│   │   ├── authoring.py  quality.py  risk.py
│   │   ├── narrative.py  dependency.py  retrospective.py
│   │   └── prompts/                  # prompt templates (.md or .py)
│   ├── mcp/
│   │   └── server.py                 # MCP server exposing OKR tools
│   ├── auth/                         # JWT, OIDC, password, sessions
│   ├── security/                     # encryption (Fernet) for stored credentials
│   └── tasks/                        # Celery tasks (reminders, ai, sync)
├── tests/
│   ├── conftest.py                   # fixtures: db, client, factories
│   ├── factories/                    # test data builders
│   ├── unit/  integration/  e2e/
│   └── fixtures/                     # sample payloads (jira, salesforce, sarif-like)
└── frontend/
    ├── package.json  vite.config.ts  tsconfig.json
    └── src/
        ├── api/                      # generated client from openapi.json
        ├── components/  pages/  hooks/  routes.tsx
```

---

## Phase 1: Foundation & Data Model

### Purpose
Establish the project skeleton, configuration, database schema, and ORM models so every later phase has a stable persistence layer and app factory. After this phase the app boots, connects to Postgres/Redis, runs migrations, and serves a health check. No business logic yet — this de-risks the schema, which all other phases depend on.

### Tasks

#### 1.1 — Project scaffolding & configuration

**What**: Create the Python project, dependency manifest, Docker/compose setup, and environment-driven settings.

**Design**:
- `pyproject.toml` with dependencies grouped (`api`, `worker`, `ai`, `dev`). Tooling config for `ruff` and `mypy --strict`.
- `src/okr/config.py` using `pydantic_settings.BaseSettings`:

```python
class Settings(BaseSettings):
    database_url: str
    redis_url: str = "redis://redis:6379/0"
    jwt_secret: str
    jwt_alg: str = "HS256"
    access_token_ttl_min: int = 30
    refresh_token_ttl_days: int = 14
    credential_encryption_key: str          # Fernet key, base64
    anthropic_api_key: str | None = None
    llm_model: str = "claude-sonnet-4-6"
    slack_default_webhook: str | None = None
    smtp_host: str | None = None
    smtp_port: int = 587
    environment: Literal["dev", "test", "prod"] = "dev"
    model_config = SettingsConfigDict(env_prefix="OKR_", env_file=".env")
```

- `src/okr/main.py`: `create_app()` factory mounting routers, adding CORS, exception handlers, and `GET /healthz` returning `{"status":"ok","db":bool,"redis":bool}`.
- `docker-compose.yml`: services `api`, `worker`, `beat`, `db` (postgres:16), `redis` (redis:7), `frontend`. `.env.example` enumerates every `OKR_*` var.

**Testing**:
- `Unit: Settings loads from env vars with OKR_ prefix → correct typed values`
- `Unit: missing required jwt_secret → ValidationError naming the field`
- `Integration (real, testcontainers): GET /healthz → 200, db=true, redis=true`
- `E2E: docker compose up → api container healthy, /healthz reachable on host port`

#### 1.2 — Database engine, session, migration harness

**What**: Async SQLAlchemy engine, request-scoped session dependency, and Alembic wired to the ORM metadata.

**Design**:
- `src/okr/db.py`: `create_async_engine(settings.database_url)`, `async_sessionmaker`, and `get_session()` FastAPI dependency (`async with` yielding `AsyncSession`).
- `migrations/env.py` imports `Base.metadata` from `src/okr/models` for autogenerate.
- Convention: every phase adds exactly one Alembic revision; `alembic upgrade head` runs in container entrypoint before the API starts.

**Testing**:
- `Integration (real): get_session yields a working session, rolls back on exception`
- `Integration (real): alembic upgrade head on empty db → all tables created; alembic downgrade base → clean`

#### 1.3 — Core ORM models & initial migration

**What**: Implement all tables from Suggestion 1, plus selective JSONB columns from Suggestion 2, as SQLAlchemy models and one migration.

**Design**:
Tables (PostgreSQL DDL, condensed; full DDL per Suggestion 1):
- `organisations` (id, name, slug UNIQUE, billing_plan, fiscal_year_start_month, config JSONB `{sso, scim, slack, branding}`, timestamps)
- `teams` (id, org_id FK, parent_team_id self-FK, name, slug, description; UNIQUE(org_id, slug))
- `users` (id, org_id FK, email UNIQUE, name, role CHECK in owner/admin/manager/member/viewer, timezone, scim_external_id, password_hash NULLABLE, is_active, timestamps)
- `team_members` (team_id, user_id, role CHECK lead/member; PK composite)
- `cycles` (id, org_id FK, name, cycle_type, start_date, end_date, is_active; UNIQUE(org_id,name); CHECK end>start)
- `objectives` (id, org_id, cycle_id, owner_id, team_id NULL, parent_objective_id self-FK, title, description, level CHECK company/team/individual, status CHECK on_track/at_risk/off_track/completed/cancelled, progress REAL 0..1, is_stretch, tags TEXT[], `ai` JSONB, sort_order, timestamps)
- `key_results` (id, objective_id FK CASCADE, owner_id, title, description, metric_type CHECK number/percentage/currency/boolean/custom, unit, start_value, current_value, target_value, progress, status, confidence REAL NULL, data_source_id FK NULL, `integration` JSONB NULL, `ai` JSONB, sort_order, timestamps)
- `check_ins` (id, key_result_id FK CASCADE, user_id, previous_value, new_value, confidence CHECK on_track/at_risk/off_track, note, source CHECK manual/integration/ai_suggested, created_at)
- `initiatives` (id, key_result_id FK CASCADE, owner_id, title, description, status, priority, due_date, external_id, external_source, completed_at, timestamps)
- `comments` (id, user_id, commentable_type CHECK objective/key_result/initiative/check_in, commentable_id, body, timestamps) — polymorphic
- `data_sources` (id, org_id, provider, name, status, credentials_enc TEXT, config JSONB, last_synced_at, timestamps; UNIQUE(org_id,provider,name))
- `data_source_mappings` (id, data_source_id FK, key_result_id FK, query_expression, transform_expression, last_value, last_synced_at; UNIQUE(data_source_id,key_result_id))
- `ai_goal_analyses` (id, objective_id FK NULL, key_result_id FK NULL, analysis_type CHECK, score, summary, details JSONB, model_version, created_at)
- `ai_narratives` (id, org_id, cycle_id, narrative_type CHECK, scope_team_id NULL, content, model_version, status CHECK draft/reviewed/published, published_by NULL, created_at)
- `notification_preferences` (user_id, channel, event_type, is_enabled; PK composite)
- `api_keys` (id, org_id, user_id, name, key_hash UNIQUE, key_prefix, scopes TEXT[], is_active, created_at)

Indexes per Suggestion 1 (org+cycle, owner+cycle, team+cycle, parent, partial status), plus GIN on `objectives.tags` and `objectives.ai`.

**Testing**:
- `Unit: progress CHECK rejects value 1.5 → IntegrityError`
- `Unit: objectives.level not in enum → IntegrityError`
- `Integration (real): insert org→team→user→cycle→objective→key_result→check_in chain succeeds`
- `Integration (real): delete objective cascades to key_results and check_ins`
- `Integration (real): duplicate (org_id, slug) team → unique violation`
- `Integration (real): recursive CTE over parent_objective_id returns full tree for seeded fixture`

---

## Phase 2: Identity, AuthN/AuthZ & RBAC

### Purpose
Add users, sessions, password and token auth, organisation membership, and role-based access control. Every subsequent endpoint depends on "who is the caller and what may they do." Ships local email/password login plus the JWT bearer scheme; enterprise SSO (SAML/OIDC) and SCIM come in Phase 8.

### Tasks

#### 2.1 — Password auth, JWT issuance & refresh

**What**: Email/password registration (org bootstrap), login, and JWT access/refresh tokens (RFC 7519/6750).

**Design**:
- `src/okr/auth/passwords.py`: `hash_password`/`verify_password` (argon2 via `passlib`).
- `src/okr/auth/tokens.py`: `create_access_token(sub, org_id, role, scopes)` → JWT with `exp`; `create_refresh_token`; `decode_token`.
- Endpoints:
  - `POST /auth/register` `{org_name, email, name, password}` → creates org + owner user → `{access_token, refresh_token, token_type:"bearer"}`
  - `POST /auth/login` `{email, password}` → 200 tokens | 401
  - `POST /auth/refresh` `{refresh_token}` → new access token | 401
- Bearer transmitted as `Authorization: Bearer <jwt>` (RFC 6750).

**Testing**:
- `Unit: hash then verify round-trips; wrong password → False`
- `Unit: decode_token on expired token → raises ExpiredSignatureError`
- `Integration (real): register → login with same creds → 200 with valid JWT`
- `Integration (real): login wrong password → 401, no token`
- `Integration (real): refresh with revoked/expired refresh token → 401`

#### 2.2 — Auth dependencies & RBAC service

**What**: FastAPI dependencies resolving the current user and enforcing role/object-level permissions (OWASP API1 — Broken Object Level Authorization).

**Design**:
- `deps.get_current_user(token) -> User` (decodes JWT, loads user, 401 if inactive).
- `src/okr/services/rbac.py`:

```python
class Action(StrEnum):
    READ = "read"; WRITE = "write"; DELETE = "delete"; ADMIN = "admin"

ROLE_RANK = {"viewer":0,"member":1,"manager":2,"admin":3,"owner":4}

def can(user: User, action: Action, obj: Objective | KeyResult | None) -> bool:
    # 1. obj.org_id must equal user.org_id  (tenant isolation)
    # 2. READ: any member of org
    # 3. WRITE: owner of obj, team lead of obj.team, manager+ of org
    # 4. DELETE/ADMIN: admin+ or owner
```

- `deps.require(action, loader)` dependency factory raising 403 on failure, 404 if object's org_id != caller's org (avoid leaking existence).

**Testing**:
- `Unit: viewer can READ but not WRITE own org objective`
- `Unit: member WRITE on objective they don't own and aren't team lead of → False`
- `Unit: user from org A requesting org B objective → treated as 404`
- `Integration (real): manager PATCHes a team member's objective → 200; member PATCHes manager's → 403`

#### 2.3 — Teams, users, membership management

**What**: CRUD for teams (with parent hierarchy) and user management within an org.

**Design**:
- `GET/POST /teams`, `GET/PATCH/DELETE /teams/{id}`, `POST /teams/{id}/members` `{user_id, role}`, `DELETE /teams/{id}/members/{user_id}`.
- `GET /users`, `POST /users` (admin invites), `PATCH /users/{id}` (role change — admin+ only), `DELETE /users/{id}` (soft delete `is_active=false`).
- Team tree returned via recursive CTE; cycle detection prevents `parent_team_id` loops.

**Testing**:
- `Unit: setting parent_team_id to a descendant → ValidationError (cycle)`
- `Integration (real): create nested teams Eng→Backend→API → GET /teams returns tree`
- `Integration (real): non-admin POST /users → 403`

---

## Phase 3: Core OKR Engine — Objectives, Key Results, Check-ins

### Purpose
The heart of the product. Implement objective/key-result CRUD, the progress computation engine, status derivation, and the check-in workflow with confidence ratings. After this phase a team can author OKRs and track them manually — the table-stakes MVP capability that every competitor has.

### Tasks

#### 3.1 — Objective & Key Result CRUD

**What**: Full lifecycle endpoints for objectives and nested key results within a cycle.

**Design**:
- Pydantic schemas: `ObjectiveCreate`, `ObjectiveOut`, `KeyResultCreate`, `KeyResultOut`.

```python
class KeyResultCreate(BaseModel):
    title: str
    metric_type: Literal["number","percentage","currency","boolean","custom"] = "number"
    unit: str | None = None
    start_value: float = 0
    target_value: float
    owner_id: UUID | None = None      # defaults to objective owner

class ObjectiveCreate(BaseModel):
    cycle_id: UUID
    title: str
    description: str | None = None
    level: Literal["company","team","individual"] = "team"
    team_id: UUID | None = None
    parent_objective_id: UUID | None = None
    owner_id: UUID | None = None       # defaults to caller
    is_stretch: bool = False
    tags: list[str] = []
    key_results: list[KeyResultCreate] = []
```

- Endpoints: `POST /objectives`, `GET /objectives?cycle_id=&team_id=&owner_id=&status=`, `GET/PATCH/DELETE /objectives/{id}`, `POST /objectives/{id}/key-results`, `PATCH/DELETE /key-results/{id}`. All gated by RBAC (3.x of Phase 2).
- `parent_objective_id` validated to belong to same org and same cycle.

**Testing**:
- `Unit: ObjectiveCreate with parent in different cycle → 422 from service validation`
- `Integration (real): create objective with 3 nested key results → all persisted, returned with computed progress 0`
- `Integration (real): GET /objectives?status=at_risk filters correctly`

#### 3.2 — Progress & status computation engine

**What**: Deterministic computation of key-result progress, objective progress roll-up, and status derivation.

**Design**:
- `src/okr/services/progress.py`:

```python
def kr_progress(start: float, current: float, target: float, metric: str) -> float:
    if metric == "boolean":
        return 1.0 if current >= target else 0.0
    if target == start:
        return 1.0 if current >= target else 0.0
    return max(0.0, min(1.0, (current - start) / (target - start)))

def objective_progress(krs: list[KeyResult]) -> float:
    return mean(kr.progress for kr in krs) if krs else 0.0

def derive_status(progress: float, cycle_elapsed: float,
                  confidence: str | None) -> str:
    # confidence override: an explicit off_track check-in wins
    if confidence == "off_track": return "off_track"
    expected = cycle_elapsed                 # linear pace expectation
    if progress >= expected - 0.10: return "on_track"
    if progress >= expected - 0.25: return "at_risk"
    return "off_track"
```

- `cycle_elapsed = (today - start_date) / (end_date - start_date)`, clamped 0..1.
- Recomputation triggered on check-in, KR update, and on a daily Celery task (status can drift as the cycle elapses even with no new check-in).

**Testing**:
- `Unit: kr_progress(0,5,10,"number") == 0.5`
- `Unit: kr_progress(100,80,50,"number") == 0.6 (descending target)`
- `Unit: kr_progress(0,1,1,"boolean") == 1.0`
- `Unit: derive_status: progress 0.2 at cycle_elapsed 0.5 → off_track`
- `Unit: explicit off_track confidence overrides high progress → off_track`
- `Unit: objective_progress averages KR progress; empty → 0.0`

#### 3.3 — Check-in workflow

**What**: Record periodic check-ins that update a key result's current value, set confidence, and roll up to the parent objective.

**Design**:
- `POST /key-results/{id}/check-ins` `{new_value, confidence, note?}`:
  1. Load KR, capture `previous_value`.
  2. Insert `check_ins` row (source=`manual`).
  3. Update `kr.current_value`, recompute `kr.progress` and `kr.status`.
  4. Recompute parent objective progress/status.
  5. Emit `goal.checked_in` internal event (consumed by notifications, Phase 6).
- `GET /key-results/{id}/check-ins` → chronological history for trend charts.
- Confidence enum: `on_track | at_risk | off_track`.

**Testing**:
- `Integration (real): check-in moves KR 0→5 (target 10) → progress 0.5, objective progress recomputed`
- `Integration (real): check-in with confidence off_track → KR status off_track regardless of progress`
- `Integration (real): check-in history endpoint returns rows newest-first`
- `Unit: check-in on KR in another org → 404`

---

## Phase 4: Alignment, Dashboards & Search

### Purpose
Make the cascading hierarchy visible and queryable — the procurement criterion called out in research.md ("visualisation of cross-team alignment has become a key procurement criterion"). Adds the alignment tree, roll-up dashboards, status aggregation, and filtered search across the hierarchy.

### Tasks

#### 4.1 — Alignment tree API

**What**: Return the full company→team→individual objective tree for a cycle, with progress and status at each node.

**Design**:
- `src/okr/services/alignment.py`: `build_tree(org_id, cycle_id) -> list[TreeNode]` using a recursive CTE (Suggestion 1 example query), then nests rows in memory.

```python
class TreeNode(BaseModel):
    id: UUID; title: str; level: str; status: str; progress: float
    owner_name: str; team_name: str | None
    key_results: list[KeyResultSummary]
    children: list["TreeNode"]
```

- `GET /alignment/tree?cycle_id=` → root nodes (level=company) with nested children.
- `GET /alignment/objectives/{id}/lineage` → ancestor chain (for "what does this support?").

**Testing**:
- `Integration (real): seeded 3-level hierarchy → tree has correct nesting and depth`
- `Integration (real): orphan objective (no parent) appears as its own root`
- `Unit: lineage returns ancestors in order child→...→company`

#### 4.2 — Dashboards & aggregation

**What**: Org- and team-level dashboards summarising status distribution, average progress, and check-in cadence compliance.

**Design**:
- `GET /dashboards/org?cycle_id=` →

```python
class DashboardOut(BaseModel):
    total_objectives: int; total_key_results: int
    avg_progress: float
    status_counts: dict[str, int]      # {on_track, at_risk, off_track, ...}
    at_risk_objectives: list[ObjectiveOut]
    checkin_compliance: float          # % of KRs checked in within cadence
```

- `GET /dashboards/team/{team_id}?cycle_id=` — same shape scoped to a team subtree.
- Aggregates cached in Redis (60s TTL), invalidated on check-in.

**Testing**:
- `Integration (real): dashboard counts match seeded fixture (e.g. 15 on_track, 6 at_risk)`
- `Integration (real): checkin_compliance reflects KRs checked in within last 7 days`
- `Integration (real): cache hit returns identical payload; check-in invalidates cache`

#### 4.3 — Filtering & search

**What**: Advanced filtering and full-text-ish search across objectives and key results.

**Design**:
- `GET /search?q=&cycle_id=&level=&status=&owner_id=&team_id=&tags=` — `q` matches title/description via Postgres `ILIKE` / `tsvector` (add a `search_vector` generated column + GIN index in this phase's migration).
- Pagination via `limit`/`offset`; results scoped to caller's org (tenant isolation enforced in query, not just RBAC).

**Testing**:
- `Integration (real): q="revenue" returns only matching objectives in caller org`
- `Integration (real): tags filter uses GIN index, returns objectives with all requested tags`
- `Integration (real): search never returns another org's rows`

---

## Phase 5: Public REST API, OpenAPI Spec & API Keys

### Purpose
Expose the platform as an integration backbone (README: "REST API as the integration backbone"). Adds API-key auth for machine clients, the published OpenAPI 3.1 spec, and standard pagination/error envelopes. This is also the contract the frontend and MCP server consume.

### Tasks

#### 5.1 — API-key authentication

**What**: Issue and verify scoped API keys for programmatic access (alongside JWT for interactive users).

**Design**:
- `POST /api-keys` `{name, scopes}` → returns the full key **once** (`okr_live_<random>`), stores only `key_hash` (sha256) + `key_prefix`.
- Auth dependency accepts either `Authorization: Bearer <jwt>` or `Authorization: Bearer okr_live_...`; the latter resolves the org/scopes from `api_keys`.
- Scopes: `okr:read`, `okr:write`, `checkins:write`, `integrations:manage`, `admin`.

**Testing**:
- `Unit: stored key_hash never equals plaintext; prefix matches first 12 chars`
- `Integration (real): request with valid key + okr:read scope → 200; missing scope → 403`
- `Integration (real): revoked key (is_active=false) → 401`

#### 5.2 — OpenAPI 3.1 spec, error & pagination conventions

**What**: Standardise error responses (RFC 7231 status codes) and pagination, and commit the generated OpenAPI 3.1 document.

**Design**:
- Global exception handler → RFC 7807-style `{type, title, status, detail}` JSON problem.
- `PageParams(limit<=100, offset>=0)`; list responses wrapped as `{items, total, limit, offset}`.
- `scripts/export_openapi.py` writes `openapi.json`; CI fails if it drifts from the committed copy.

**Testing**:
- `Unit: validation error → 422 problem JSON with field path in detail`
- `Integration: GET /openapi.json → valid OpenAPI 3.1 (validate with openapi-spec-validator)`
- `Integration: list endpoint respects limit/offset, total accurate`

---

## Phase 6: Notifications & Check-in Cadence

### Purpose
Drive the check-in habit (Tability/Weekdone's differentiator) and surface status changes. Adds Slack + email notifications following Slack's incoming-webhook format (standards.md) and a Celery-beat reminder cadence with timezone-aware timing.

### Tasks

#### 6.1 — Notification service & channels

**What**: Deliver notifications for check-in reminders, status changes, comment mentions, and at-risk goals across Slack/email/in-app, honouring per-user preferences.

**Design**:
- `src/okr/services/notifications.py`: `notify(user, event_type, context)` checks `notification_preferences`, fans out to enabled channels.
- Slack payload follows the documented incoming-webhook JSON (`{blocks:[...]}`); email via SMTP with a templated body.
- In-app notifications stored in a `notifications` table (added here): `(id, user_id, event_type, body, read_at, created_at)`.
- `event_type` enum matches `notification_preferences`: `checkin_reminder, status_change, comment_mention, goal_at_risk, cycle_starting, cycle_ending`.

**Testing**:
- `Unit: user disabled slack for goal_at_risk → slack channel skipped, email still sent`
- `Integration (mocked Slack): goal_at_risk → POST to webhook URL with valid blocks payload`
- `Integration (mocked SMTP): status_change email rendered with objective title`

#### 6.2 — Reminder cadence (Celery beat)

**What**: Scheduled reminders for due check-ins, timed to each user's timezone and the org's cadence.

**Design**:
- Celery-beat hourly task `enqueue_due_reminders` scans active cycles, finds KRs whose owners haven't checked in within the cadence window (org `checkin_cadence`: weekly/monthly), and—if local time is the configured send hour (e.g. 09:00 in `user.timezone`)—enqueues `notify(..., checkin_reminder)`.
- Cycle start/end reminders fire from `cycle_starting`/`cycle_ending` based on `cycles.start_date`/`end_date`.

**Testing**:
- `Unit: user in America/New_York at 09:00 local → eligible; at 14:00 → not`
- `Unit: KR checked in 2 days ago with weekly cadence → no reminder; 8 days ago → reminder`
- `Integration (mocked clock + notifications): beat run enqueues exactly the overdue owners`

---

## Phase 7: Data Source Integrations & Automated Progress

### Purpose
Auto-update key results from external systems — the headline differentiator (Profit.co, Quantive). Builds a connector framework and Jira/GitHub/Salesforce connectors plus a generic inbound webhook, all writing check-ins with `source=integration`. Uses OAuth 2.0 (RFC 6749) for delegated access and encrypts stored credentials.

### Tasks

#### 7.1 — Connector framework & credential security

**What**: A pluggable connector protocol, encrypted credential storage, and the data-source/mapping management API.

**Design**:
- `src/okr/security/crypto.py`: Fernet encrypt/decrypt using `credential_encryption_key`; `data_sources.credentials_enc` stores ciphertext only.
- `src/okr/integrations/base.py`:

```python
class Connector(Protocol):
    provider: str
    def validate_config(self, config: dict) -> None: ...
    async def test_connection(self, creds: dict, config: dict) -> bool: ...
    async def fetch_value(self, creds: dict, config: dict,
                          mapping: DataSourceMapping) -> float: ...
```

- `registry.py` maps provider → Connector. Endpoints: `POST /integrations` (connect, validates + stores encrypted creds), `POST /integrations/{id}/test`, `POST /key-results/{id}/mapping` `{data_source_id, query_expression, transform_expression?}`.

**Testing**:
- `Unit: credentials_enc round-trips through Fernet; plaintext never stored`
- `Unit: invalid Jira config (missing base_url) → validate_config raises`
- `Integration (mocked): POST /integrations/{id}/test → calls connector.test_connection, returns status`

#### 7.2 — Jira, GitHub, Salesforce connectors

**What**: Concrete connectors that fetch a metric value for a mapped key result.

**Design**:
- **Jira** (`jira` SDK): `query_expression` is JQL; `fetch_value` returns matching issue count (e.g. `project = PLAT AND status = Done`). OAuth 2.0 access/refresh tokens with auto-refresh.
- **GitHub** (`PyGithub`): `query_expression` is a search query (e.g. PRs merged in repo within cycle); returns count. PAT or OAuth.
- **Salesforce** (`simple-salesforce`): `query_expression` is SOQL; `fetch_value` returns `totalSize` or a SUM aggregate. OAuth 2.0.
- `transform_expression` (optional) is a safe arithmetic expression applied to the raw value (e.g. `value / 1000`), evaluated via a restricted AST evaluator (no `eval`).

**Testing**:
- `Integration (mocked Jira API): JQL returns 22 issues → fetch_value == 22`
- `Integration (mocked GitHub): search returns 7 PRs → fetch_value == 7`
- `Integration (mocked Salesforce): SOQL totalSize 28 → fetch_value == 28; transform "/1000" → 0.028`
- `Unit: transform_expression with disallowed call (e.g. __import__) → rejected`

#### 7.3 — Sync engine & inbound webhooks

**What**: Scheduled connector sync that creates integration check-ins, plus a signed inbound webhook for custom sources.

**Design**:
- Celery task `sync_data_source(data_source_id)`: for each mapping, `fetch_value`, apply transform, and if changed, create a `check_in` with `source=integration`, then recompute progress/status (reuses Phase 3 engine). Append a sync-log entry to `data_sources.config->sync_log` (JSONB).
- Celery beat schedules sync per data source (default every 6h).
- `POST /webhooks/{data_source_id}` verifies an HMAC signature (`X-OKR-Signature`) against the stored `webhook_secret_hash`; body `{mapping_id, value}` creates a check-in. Invalid signature → 401, no check-in (mirrors OWASP guidance).

**Testing**:
- `Integration (real db + mocked connector): sync creates integration check-in, updates KR progress`
- `Integration (real): sync with unchanged value → no duplicate check-in`
- `Integration (mocked): webhook with valid HMAC → 200, check-in created`
- `Integration (mocked): webhook with bad signature → 401, no check-in`

---

## Phase 8: Enterprise SSO & SCIM Provisioning

### Purpose
Clear "the enterprise gate" (standards.md): SAML 2.0 + OIDC SSO and SCIM 2.0 provisioning. Without these, enterprises (500+ users) cannot adopt. Designed to slot onto the Phase 2 identity model without rework.

### Tasks

#### 8.1 — OIDC & SAML SSO

**What**: Federated login against enterprise IdPs (Okta, Azure AD, Google Workspace).

**Design**:
- OIDC (Authlib): `GET /auth/sso/oidc/login?org=` → redirect to IdP; `GET /auth/sso/oidc/callback` → validate ID token (JWT), match/create user by email + `org.config.sso`, issue platform JWT.
- SAML 2.0 (`python3-saml`): `GET /auth/sso/saml/login?org=`, `POST /auth/sso/saml/acs` (assertion consumer service) validating the signed XML assertion, mapping NameID→user.
- SSO settings (entity_id, sso_url, certificate) stored in `organisations.config.sso` JSONB.

**Testing**:
- `Integration (mocked IdP): OIDC callback with valid ID token → platform JWT, user upserted`
- `Integration (mocked): SAML ACS with valid signed assertion → session; tampered assertion → 401`
- `Unit: SSO user email not in org domain allowlist → rejected`

#### 8.2 — SCIM 2.0 endpoints

**What**: RFC 7643/7644 endpoints for IdP-driven user and group provisioning.

**Design**:
- Bearer-token-secured (SCIM token in `org.config.scim`) endpoints:
  - `GET/POST /scim/v2/Users`, `GET/PUT/PATCH/DELETE /scim/v2/Users/{id}`
  - `GET/POST /scim/v2/Groups`, `PATCH /scim/v2/Groups/{id}` (membership)
- Map SCIM User→`users` (externalId→`scim_external_id`, active→`is_active`), SCIM Group→`teams` + `team_members`.
- Responses use SCIM schemas/`meta` envelope; `PATCH` supports the SCIM PatchOp format.

**Testing**:
- `Integration (real): POST /scim/v2/Users (valid SCIM) → user created with scim_external_id`
- `Integration (real): PATCH active=false → user.is_active false (de-provisioning)`
- `Integration (real): Group PATCH add member → team_members row created`
- `Integration: missing/invalid SCIM bearer token → 401`

---

## Phase 9: AI-Native Features

### Purpose
Deliver the AI-native advantage that distinguishes this platform: authoring, quality scoring, risk flagging, retrospectives, cross-team dependency detection, and board-ready narratives — gaps the research identified as underserved. All run through a swappable `LLMClient` and persist to `ai_goal_analyses` / `ai_narratives` and the JSONB `ai` columns.

### Tasks

#### 9.1 — LLM client abstraction

**What**: A single interface for all LLM calls, with the Anthropic implementation.

**Design**:
- `src/okr/ai/client.py`:

```python
class LLMClient(Protocol):
    async def complete(self, system: str, user: str,
                       response_schema: type[BaseModel] | None = None) -> Any: ...

class AnthropicClient(LLMClient):
    # uses anthropic SDK, model=settings.llm_model
    # when response_schema given, instructs JSON output and validates with Pydantic
```

- Prompt templates live in `src/okr/ai/prompts/`. All AI tasks run as Celery jobs (avoid blocking requests) and record `model_version`.

**Testing**:
- `Unit (mocked SDK): complete returns parsed Pydantic model when schema given`
- `Unit: malformed JSON from model → retried once, then ValidationError surfaced`

#### 9.2 — OKR authoring & quality scoring

**What**: Suggest well-formed objectives/KRs from context, and score existing OKRs for measurability, ambition, and alignment with coaching feedback.

**Design**:
- `POST /ai/authoring` `{team_id, context}` → suggested `{objective, key_results[]}` draft (not persisted until accepted).
- `POST /ai/quality/{objective_id}` → `AiGoalAnalysis(analysis_type="quality_score")` with `score 0..1` + `details.feedback[]`; also written to `objectives.ai.quality_*`.
- Quality prompt rubric: specific & measurable KRs, appropriate ambition (stretch vs sandbagging), KRs prove the objective, time-bound within the cycle.

**Testing**:
- `Integration (mocked LLM): authoring returns draft with >=2 key results, not persisted`
- `Integration (mocked LLM): quality score persists ai_goal_analyses row + updates objectives.ai`
- `Unit: vague KR ("improve things") fixture → low score with measurability feedback`

#### 9.3 — Risk scoring, forecasting & cross-team dependency detection

**What**: Flag goals trending to failure, forecast outcomes, and detect when one team's off-track goal endangers another's — the open-gap differentiator.

**Design**:
- Daily Celery task `analyze_risks(cycle_id)`: for each active objective, combine deterministic signals (progress vs `cycle_elapsed`, days since last check-in, KR confidence) with an LLM judgement → `risk_score` + `risk_factors[]` (analysis_type=`risk_prediction`).
- Dependency detection: LLM analyses objective titles/descriptions/tags across teams to propose `dependencies:[{objective_id, team, risk}]` (analysis_type=`dependency_detection`); flags when a depended-on objective is at_risk/off_track → triggers `goal_at_risk` notification to the dependent owner.
- Forecast: predict final value/status per KR from check-in trend (analysis_type=`progress_forecast`).

**Testing**:
- `Unit: objective at 0.2 progress, 0.7 cycle elapsed, 21 days no check-in → high deterministic risk before LLM`
- `Integration (mocked LLM): dependency detection links two seeded objectives, notifies dependent owner when source goes off_track`
- `Integration (mocked LLM): forecast persists predicted_value + predicted_status`

#### 9.4 — Quarterly retrospectives & board narratives

**What**: Generate plain-language executive summaries, quarterly retrospectives, and board decks from structured OKR + check-in data.

**Design**:
- `POST /ai/narrative` `{cycle_id, narrative_type, scope_team_id?}` → Celery job aggregates cycle metrics + check-in trends, prompts the LLM, stores `ai_narratives` row (`status=draft`).
- `narrative_type`: `executive_summary | quarterly_retrospective | team_report | board_deck | all_hands`.
- `POST /ai/narrative/{id}/publish` (manager+) sets `status=published`, `published_by`.
- Board deck export reuses Phase 10 PDF renderer.

**Testing**:
- `Integration (mocked LLM): executive_summary narrative persisted as draft with model_version`
- `Integration (real): publish narrative → status published, published_by set; member publish → 403`
- `Unit: aggregation feeds correct metrics snapshot (on_track/at_risk counts) into the prompt`

---

## Phase 10: Exports, Reporting & MCP Server

### Purpose
Round out reporting (PDF/Excel for all-hands and board decks — an MVP requirement) and ship the MCP server that lets AI assistants read/write OKRs (the standards.md differentiator). These are independent capabilities that can be built in parallel once the core API exists.

### Tasks

#### 10.1 — PDF & Excel export

**What**: Export OKRs, dashboards, and narratives to PDF and Excel.

**Design**:
- `GET /exports/okrs.xlsx?cycle_id=&team_id=` (`openpyxl`): one sheet of objectives, one of key results with progress/status.
- `GET /exports/okrs.pdf?cycle_id=` (`weasyprint`): renders an HTML template (alignment tree + status summary). Narrative PDFs render an `ai_narratives` row.
- Streamed responses with correct `Content-Type`/`Content-Disposition`.

**Testing**:
- `Integration (real): xlsx export opens with openpyxl, rows match seeded objectives`
- `Integration (real): pdf export returns non-empty application/pdf; contains objective titles (text-extract assert)`

#### 10.2 — MCP server

**What**: An MCP server exposing OKR read/write tools to LLM assistants (Claude, Copilot).

**Design**:
- `src/okr/mcp/server.py` (official `mcp` SDK) exposing tools:
  - `list_okrs(team_id?, owner_id?, cycle_id?)` → current objectives + KRs
  - `get_at_risk(cycle_id?)` → at-risk/off-track goals
  - `record_check_in(key_result_id, new_value, confidence, note?)`
  - `draft_check_in(key_result_id)` → suggested update from recent activity
  - `query_check_in_history(key_result_id)`
- Authenticated with an API key (Phase 5) mapped to scopes; all calls go through the same RBAC + service layer (no bypass).

**Testing**:
- `Integration (mocked MCP client): list_okrs returns caller-org objectives only`
- `Integration (real): record_check_in via MCP creates a check_in row + recomputes progress`
- `Unit: MCP tool with key lacking checkins:write scope → permission error`

---

## Phase 11: Frontend SPA

### Purpose
Deliver the web UI competitors consider table stakes: dashboards, the alignment tree, goal authoring, and the check-in flow. Built last so it consumes a stable, fully specified API (OpenAPI from Phase 5). Can be developed in parallel with Phases 7–10 against the spec.

### Tasks

#### 11.1 — App shell, auth & generated API client

**What**: React+Vite app with routing, login/SSO, and a typed client generated from `openapi.json`.

**Design**:
- `frontend/src/api/` generated via `openapi-typescript`; TanStack Query hooks wrap it. Auth context stores JWT, refresh on 401. SSO buttons hit `/auth/sso/*`.
- Routes: `/login`, `/dashboard`, `/alignment`, `/objectives/:id`, `/cycles`, `/teams`, `/settings/integrations`.

**Testing**:
- `Unit (Vitest): auth context refreshes token on 401 then retries`
- `E2E (Playwright, mocked API): login → redirected to /dashboard`

#### 11.2 — Dashboards, alignment tree & check-in UI

**What**: The core interactive screens.

**Design**:
- Dashboard: status donut + avg-progress + at-risk list (Recharts), consuming `/dashboards/org`.
- Alignment tree: `react-flow` rendering `/alignment/tree`, color-coded by status, expandable nodes; risk badges from `objectives.ai`.
- Objective detail: KR progress bars, check-in form (modal), comment thread, AI quality/risk panel.

**Testing**:
- `E2E (Playwright, mocked API): submit check-in → KR progress bar updates`
- `E2E: alignment tree renders seeded hierarchy with correct node count and status colors`
- `E2E: at-risk objective shows risk badge from ai field`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Data Model            ─── required by everything
    │
Phase 2: Identity, Auth & RBAC              ─── requires 1
    │
Phase 3: Core OKR Engine                    ─── requires 2
    │
    ├── Phase 4: Alignment, Dashboards, Search   ─── requires 3
    ├── Phase 5: Public REST API + OpenAPI       ─── requires 3 (parallel w/ 4)
    │       │
    │       ├── Phase 6: Notifications & Cadence      ─── requires 3
    │       ├── Phase 7: Integrations & Auto-Progress ─── requires 3,5
    │       ├── Phase 8: Enterprise SSO & SCIM        ─── requires 2 (parallel)
    │       ├── Phase 9: AI-Native Features           ─── requires 3,4
    │       └── Phase 10: Exports & MCP Server        ─── requires 5,9(narrative PDF)
    │
Phase 11: Frontend SPA                      ─── requires 5 spec; parallel w/ 7–10
```

**Parallelism opportunities**
- After Phase 3: Phases 4 and 5 can proceed concurrently.
- After Phase 5: Phases 6, 7, 8, 9 are largely independent and can be split across developers.
- Phase 11 (frontend) can start as soon as the Phase 5 OpenAPI spec is stable, running in parallel with Phases 7–10.
- Phase 8 (SSO/SCIM) depends only on Phase 2 and can begin early in parallel with the Phase 3+ track.

---

## Definition of Done (per phase)

A phase is complete only when all of the following hold:

1. All tasks implemented as specified.
2. All unit and integration tests pass; new tests cover happy-path and the listed edge cases.
3. `ruff check` and `ruff format --check` pass with no errors.
4. `mypy --strict src/okr` passes.
5. Exactly one Alembic migration added for schema changes; `alembic upgrade head` then `downgrade base` runs cleanly on an empty database.
6. The feature works end-to-end via `docker compose up` (API + worker + db + redis).
7. New `OKR_*` config options added to `.env.example` and documented.
8. New/changed endpoints appear in the regenerated `openapi.json` (committed; CI drift check passes).
9. Tenant isolation verified: no endpoint returns or mutates another organisation's data (OWASP API1/API3).
10. Stored secrets (credentials, SCIM/API tokens) are encrypted or hashed at rest — never persisted in plaintext.
11. Frontend phases additionally: `vitest` unit tests and `playwright` E2E pass; `tsc --noEmit` clean.
```
