# Predictive HR Analytics — Phased Development Plan

> Project: 488-predictive-hr-analytics · Created: 2026-05-31
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and `data-model-suggestion-1.md` (Normalized Relational, PostgreSQL — selected as the foundation; the JSONB and event-sourcing variants are noted as future migration paths). The product is an AI-native, open-source, **self-hostable** platform that predicts flight risk, promotion readiness, and team conflict from continuous workforce signals, with first-class explainability, fairness auditing, and a prescriptive action layer with outcome tracking.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | **Python 3.12** | The product is ML/LLM-heavy (flight-risk models, SHAP explainability, transformer-based survey NLP, LLM narrative generation). Python is the lingua franca of these libraries (scikit-learn, XGBoost, SHAP, transformers, fairlearn). Avoids a polyglot split between API and model layers. |
| API framework | **FastAPI** | Async-first, generates **OpenAPI 3.1** automatically (required by standards.md for enterprise gateway compatibility), Pydantic-native request/response validation, and first-class dependency injection for auth and tenancy scoping. |
| Data validation | **Pydantic v2** | Enforces the canonical HR ontology at the application boundary; JSON Schema export aligns with standards.md (JSON Schema 2020-12) and lets connectors validate against a single contract. |
| ORM / migrations | **SQLAlchemy 2.0 + Alembic** | Mature, supports the 3NF schema from data-model-suggestion-1, async engine for FastAPI, and Alembic gives reproducible, reviewable migrations needed to absorb HRIS schema drift safely. |
| Database | **PostgreSQL 16** | The selected data model is relational 3NF; Postgres provides ACID writes for sync transactions, native range partitioning for high-volume tables (`prediction_scores`, `survey_responses`, `collaboration_snapshots`), JSONB escape hatch for SHAP payloads, and `pgvector` later for survey-text embeddings. SQLite is used only for unit-test fixtures. |
| Task queue | **Celery + Redis** | HRIS syncs, model training, batch scoring, and fairness audits are long-running and must not block API requests. Celery gives retry/backoff and scheduled beat tasks (configurable scoring cadence per README). Redis doubles as result backend and cache. |
| ML — tabular models | **scikit-learn + XGBoost** | XGBoost is the strongest off-the-shelf model for tabular attrition/flight-risk with limited data; scikit-learn pipelines standardise preprocessing and enable calibrated probabilities (needed for honest risk bands). |
| Explainability | **SHAP** | Provides per-prediction feature attributions (the `prediction_explanations.shap_value` column in the data model). Mandatory for GDPR Art. 22 / EU AI Act explainability and the "top 3 contributing features" MVP requirement. |
| Fairness | **Fairlearn** | Computes demographic parity difference and equalized odds difference across protected attributes — exactly the `fairness_audits` metrics in the data model. Open-source, scikit-learn compatible. |
| Survey NLP | **sentence-transformers (local) + LLM provider (pluggable)** | Transformer embeddings cluster open-text survey responses (features.md flags incumbent rule-based approaches as a weakness). Sentiment/theme labelling and narrative generation go through a pluggable LLM gateway. |
| LLM gateway | **Pluggable provider abstraction (Anthropic / OpenAI / local Ollama)** | Self-hosters must be able to run without sending employee data to a third party. A thin `LLMProvider` interface keeps narrative generation, compliance-doc drafting, and the NL query feature provider-agnostic. |
| MCP server | **Model Context Protocol server (optional component)** | standards.md identifies MCP as the open path to the "ask a question about your workforce" feature. Exposes read-only, tenancy-scoped workforce data to an LLM assistant. |
| Frontend | **React 18 + TypeScript + Vite + TanStack Query + Recharts + shadcn/ui** | The product needs role-based dashboards (HR admin / manager / employee). React SPA against the FastAPI OpenAPI; Recharts for risk/trend visualisations; shadcn/ui for accessible, themeable components. |
| Auth | **OAuth 2.0 + OIDC (Authlib), JWT sessions** | standards.md requires OIDC SSO (Okta/Azure AD/Google Workspace) for the platform and OAuth 2.0 + PKCE for HRIS connectors. RBAC enforced server-side to defend against OWASP API #1 (BOLA). |
| HRIS connectors | **Direct connectors (BambooHR, CSV) for MVP; Merge.dev unified API adapter as optional accelerator** | BambooHR (HTTP Basic) and CSV are the lowest-effort, highest-coverage MVP sources per features.md MVP scope. Workday (OAuth2 + REST/SOAP) and SuccessFactors (OData v4) are later, larger phases. A Merge.dev adapter is offered as an alternative to building 10+ connectors. |
| Containerisation | **Docker + docker-compose** | README mandates self-hosted deployment; compose bundles api, worker, beat, postgres, redis, and frontend for one-command local/self-host startup. |
| Testing | **pytest + pytest-asyncio + httpx + factory_boy + testcontainers** | Standard Python stack; testcontainers spins ephemeral Postgres/Redis for integration tests; factory_boy builds employee fixtures with longitudinal history. |
| Frontend testing | **Vitest + React Testing Library + Playwright** | Unit/component tests plus E2E flows for the three dashboards. |
| Code quality | **ruff (lint+format), mypy (strict), pre-commit; eslint + prettier (frontend)** | Type safety matters for a data-correctness-critical product. |
| Package mgmt | **uv (Python), pnpm (frontend)** | Fast, reproducible lockfiles. |
| Provenance / audit | **W3C PROV-DM-aligned `prediction_runs` + audit log** | standards.md: provenance of every model output is a GDPR Art. 22 / EU AI Act requirement. Every score links to the model version, feature snapshot, and run that produced it. |
| Standards alignment | ISO 30414 metric definitions, ISO/IEC 42001 AIMS process docs, SARIF-style structured findings, OpenAPI 3.1, OData v4 (SuccessFactors), JSON Schema 2020-12, OWASP API Top 10 | Drives metric naming, model-governance documentation, connector query semantics, and API security review. |

### Project Structure

```
predictive-hr-analytics/
├── pyproject.toml
├── uv.lock
├── README.md
├── Dockerfile
├── docker-compose.yml
├── .env.example
├── alembic.ini
├── migrations/                      # Alembic migrations
│   └── versions/
├── src/
│   └── phra/
│       ├── __init__.py
│       ├── main.py                  # FastAPI app factory
│       ├── config.py                # Pydantic Settings (env-driven)
│       ├── db.py                    # async engine, session, base
│       ├── celery_app.py            # Celery + beat config
│       ├── ontology/                # Canonical HR ontology
│       │   ├── models.py            # SQLAlchemy ORM models
│       │   ├── schemas.py           # Pydantic canonical schemas (JSON Schema export)
│       │   └── enums.py
│       ├── auth/
│       │   ├── oidc.py              # OIDC login (Authlib)
│       │   ├── jwt.py               # session token issue/verify
│       │   ├── rbac.py              # roles, scopes, dependency guards
│       │   └── tenancy.py           # organisation-scoping dependency
│       ├── ingestion/
│       │   ├── base.py              # Connector ABC + normaliser hooks
│       │   ├── normaliser.py        # source field -> canonical ontology
│       │   ├── csv_connector.py
│       │   ├── bamboohr_connector.py
│       │   ├── merge_connector.py   # optional unified-API adapter
│       │   ├── workday_connector.py # later phase
│       │   ├── successfactors_connector.py  # later phase
│       │   ├── quality.py           # data-quality assessment
│       │   └── tasks.py             # Celery sync tasks
│       ├── features/
│       │   ├── store.py             # feature extraction (proj_ml_feature_vectors)
│       │   └── definitions.py       # ISO 30414-aligned feature defs
│       ├── models/                  # ML models
│       │   ├── registry.py          # train/version/activate models
│       │   ├── flight_risk.py
│       │   ├── promotion_readiness.py
│       │   ├── team_health.py
│       │   ├── workforce_forecast.py
│       │   ├── explain.py           # SHAP -> explanations + plain language
│       │   └── tasks.py             # training + batch scoring tasks
│       ├── fairness/
│       │   └── audit.py             # Fairlearn demographic parity / eq. odds
│       ├── llm/
│       │   ├── provider.py          # LLMProvider ABC + impls
│       │   ├── narratives.py        # risk/succession narrative generation
│       │   └── compliance.py        # GDPR Art.22 / EU AI Act doc drafting
│       ├── actions/
│       │   ├── recommend.py         # action recommendation engine
│       │   └── outcomes.py          # outcome tracking + feedback loop
│       ├── compensation/
│       │   └── benchmark.py         # market benchmark ingestion + comp gap
│       ├── api/
│       │   ├── routes/              # one module per resource
│       │   ├── deps.py              # shared FastAPI dependencies
│       │   └── errors.py            # error envelope + handlers
│       ├── mcp/
│       │   └── server.py            # MCP server exposing workforce data
│       └── observability/
│           ├── audit_log.py         # PROV-DM-aligned audit trail
│           └── logging.py
├── frontend/
│   ├── package.json
│   ├── vite.config.ts
│   └── src/
│       ├── api/                     # generated OpenAPI client
│       ├── routes/
│       │   ├── admin/               # HR admin dashboards
│       │   ├── manager/             # manager dashboards
│       │   └── employee/            # employee self-service transparency
│       ├── components/
│       └── lib/
└── tests/
    ├── conftest.py                  # testcontainers Postgres/Redis fixtures
    ├── fixtures/                    # sample CSVs, BambooHR responses, longitudinal data
    ├── unit/
    ├── integration/
    └── e2e/
```

The structure groups by concern (ontology, ingestion, models, fairness, actions, api) so each phase adds modules without restructuring.

---

## Phase 1: Foundation — Project, Config, Database, Tenancy, Auth

### Purpose
Establish the runnable skeleton: a FastAPI app, Postgres connection, Alembic migrations, configuration, multi-tenant organisation scoping, OIDC login, and RBAC. After this phase the platform boots, authenticates a user against an identity provider, enforces role- and organisation-scoped access, and exposes a health endpoint and an empty OpenAPI 3.1 spec. Every later phase relies on the tenancy and RBAC primitives built here to satisfy OWASP API #1 (BOLA).

### Tasks

#### 1.1 — Project scaffolding & configuration

**What**: Create the `phra` package, `pyproject.toml` (uv), Docker assets, and a typed settings object.

**Design**:
```python
# src/phra/config.py
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_prefix="PHRA_")
    database_url: str = "postgresql+asyncpg://phra:phra@localhost:5432/phra"
    redis_url: str = "redis://localhost:6379/0"
    secret_key: str                          # JWT signing; no default — must be set
    oidc_issuer: str | None = None
    oidc_client_id: str | None = None
    oidc_client_secret: str | None = None
    llm_provider: str = "none"               # none | anthropic | openai | ollama
    llm_api_key: str | None = None
    scoring_cron: str = "0 2 * * 1"          # weekly Monday 02:00 (configurable cadence)
    environment: str = "development"

def get_settings() -> Settings: ...          # cached via lru_cache
```
- `docker-compose.yml` services: `postgres:16`, `redis:7`, `api`, `worker`, `beat`, `frontend`.
- `Dockerfile` multi-stage (uv install → slim runtime).
- `main.py` exposes `create_app()` returning a FastAPI instance with CORS, error handlers, and `/healthz`.

**Testing**:
- `Unit: get_settings() with env vars set → fields parsed with correct types and defaults.`
- `Unit: missing PHRA_SECRET_KEY → ValidationError naming secret_key.`
- `Integration: GET /healthz → 200 {"status":"ok","db":"up","redis":"up"} when services reachable.`
- `Integration: docker compose up → all services healthy (smoke).`

#### 1.2 — Database engine, base models, Alembic

**What**: Async SQLAlchemy engine/session, declarative base with `id/created_at/updated_at` mixins, and Alembic wired to autogenerate.

**Design**:
```python
# src/phra/db.py
class Base(DeclarativeBase): ...
class UUIDMixin:    id: Mapped[UUID] = mapped_column(primary_key=True, default=uuid4)
class TimestampMixin:
    created_at: Mapped[datetime]; updated_at: Mapped[datetime]
async def get_session() -> AsyncIterator[AsyncSession]: ...   # FastAPI dependency
```
- Alembic `env.py` reads `Base.metadata`; `alembic revision --autogenerate` produces reviewable migrations.

**Testing**:
- `Unit: UUIDMixin generates a UUID on insert without explicit id.`
- `Integration (testcontainers): alembic upgrade head → empty schema applies cleanly; downgrade base reverts.`

#### 1.3 — Organisation & tenancy scoping

**What**: `organisations` table and a `TenantContext` dependency that scopes every query to the caller's organisation.

**Design**:
- Implements `organisations` from data-model-suggestion-1.
```python
# src/phra/auth/tenancy.py
@dataclass
class TenantContext:
    organisation_id: UUID
    user_id: UUID
    role: Role
def require_tenant() -> TenantContext: ...   # raises 401/403; reads from JWT
```
- A reusable `scoped(query, ctx)` helper appends `WHERE organisation_id = :org` and is mandatory for all list/detail queries.

**Testing**:
- `Unit: scoped() appends organisation filter to a select.`
- `Integration: user from org A requesting org B resource id → 404 (not 403, to avoid existence leak).`

#### 1.4 — OIDC login + JWT sessions

**What**: OIDC Authorization Code + PKCE login (RFC 6749/7636) issuing platform JWT sessions (RFC 7519).

**Design**:
- Routes: `GET /auth/login` (redirect to IdP), `GET /auth/callback` (exchange code, upsert user, set session), `POST /auth/logout`.
- Dev fallback: `POST /auth/dev-login` (only when `environment=development`) issues a JWT for a seeded user.
- JWT claims: `sub`, `org`, `role`, `exp`, `iat`.

**Testing**:
- `Unit: JWT round-trip → issued token verifies; tampered signature → InvalidToken.`
- `Integration (mocked IdP): /auth/callback with valid code → user upserted, session cookie set, 302.`
- `Integration: expired token on protected route → 401.`

#### 1.5 — RBAC

**What**: Four roles with scope-based guards.

**Design**:
```python
# src/phra/auth/rbac.py
class Role(StrEnum): HR_ADMIN="hr_admin"; MANAGER="manager"; EMPLOYEE="employee"; ANALYST="analyst"
def require_role(*allowed: Role) -> Callable: ...   # FastAPI dependency factory
```
- Matrix: HR_ADMIN=all org data; MANAGER=own reporting tree only; EMPLOYEE=self only; ANALYST=read aggregate + model mgmt, no individual PII export.
- Manager tree resolved via recursive CTE on `employees.manager_id`.

**Testing**:
- `Unit: require_role(HR_ADMIN) with manager token → 403.`
- `Integration: manager requests a direct report's score → 200; requests a non-report's score → 404.`
- `Integration: employee requests another employee's data → 404.`

---

## Phase 2: Canonical HR Ontology & Core Domain Model

### Purpose
Implement the canonical HR data model — the single source of truth that insulates ML models from HRIS schema drift (the central technical risk in research.md). After this phase, all core entities (employees, departments, positions, competencies, compensation, performance, surveys, collaboration metadata) exist as ORM models with Pydantic schemas and CRUD endpoints, metric definitions align to ISO 30414, and JSON Schemas are exported for connectors to validate against.

### Tasks

#### 2.1 — Core organisational entities

**What**: ORM models + migrations for `departments`, `locations`, `positions`, `employees`.

**Design**: Implement exactly the DDL from data-model-suggestion-1 (§ Core Entities) including all CHECK constraints (`employment_type`, `status`), self-referential `manager_id`, and the five `employees` indexes. Pydantic schemas:
```python
class EmployeeIn(BaseModel):
    external_hris_id: str | None
    first_name: str; last_name: str; email: EmailStr | None
    hire_date: date; termination_date: date | None = None
    department_id: UUID | None; position_id: UUID | None
    location_id: UUID | None; manager_id: UUID | None
    employment_type: EmploymentType = EmploymentType.FULL_TIME
    status: EmployeeStatus = EmployeeStatus.ACTIVE
class EmployeeOut(EmployeeIn): id: UUID; created_at: datetime; updated_at: datetime
```

**Testing**:
- `Unit: EmployeeIn with employment_type='alien' → ValidationError.`
- `Integration: insert employee with manager_id forming a cycle → allowed at DB level; resolve_tree() detects/breaks cycle.`
- `Integration: terminating an employee sets status='terminated' and termination_date.`

#### 2.2 — Competency & skill framework

**What**: `competency_frameworks`, `competencies`, `position_competency_requirements`, `employee_competency_assessments`.

**Design**: DDL per suggestion-1 (§ Competency & Skill Framework); `required_level`/`assessed_level` 1–5 CHECK; `source` enum (self/manager/peer/360/system). Helper `competency_gap(employee_id, position_id) -> list[Gap]` computes per-competency `required - latest_assessed`.

**Testing**:
- `Unit: assessed_level=6 → ValidationError.`
- `Integration: competency_gap returns gaps only where required>assessed using the latest assessment per competency.`

#### 2.3 — Compensation & market benchmarks

**What**: `compensation_records` (with generated `total_comp`) and `market_benchmarks`.

**Design**: DDL per suggestion-1 (§ Compensation). `comp_to_market_ratio(employee_id) = latest base_salary / matching benchmark percentile_50`, matched by `position_id` + `location_id`, falling back to position-only.

**Testing**:
- `Unit: total_comp computed = base + base*bonus%/100 + equity.`
- `Integration: comp_to_market_ratio with salary 15% below P50 → 0.85.`
- `Integration: no matching benchmark → ratio None, no exception.`

#### 2.4 — Performance & engagement surveys

**What**: `review_cycles`, `performance_reviews`, `survey_campaigns`, `survey_questions`, `survey_responses`.

**Design**: DDL per suggestion-1 (§ Performance, § Engagement). Engagement scoring helper aggregates Likert responses per category (engagement/manager/culture/growth) into 0–100 scores. Open-text responses stored in `text_value` for Phase 7 NLP.

**Testing**:
- `Unit: overall_rating=0 → ValidationError (1–5).`
- `Integration: engagement score = mean of normalised Likert values per category.`
- `Integration: survey with zero responses → score None (not 0), to avoid false signal.`

#### 2.5 — Collaboration metadata

**What**: `collaboration_snapshots` (metadata only — no message content, per GDPR Art. 6 constraint in research.md).

**Design**: DDL per suggestion-1 (§ Collaboration Metadata); unique `(employee_id, snapshot_date, source)`. Ingestion validates that no free-text/content fields are present (schema rejects unknown fields).

**Testing**:
- `Unit: snapshot payload containing a 'message_body' field → rejected (extra='forbid').`
- `Integration: duplicate (employee, date, source) → upsert, not duplicate row.`

#### 2.6 — Canonical schema export (JSON Schema 2020-12)

**What**: CLI `phra export-schemas` writing JSON Schemas for every canonical entity.

**Design**: Iterate Pydantic models, emit `schemas/*.json` (Draft 2020-12). These are the contract connectors normalise toward and align with HR Open Standards vocabulary where applicable.

**Testing**:
- `Unit: exported employee schema validates a known-good payload and rejects a known-bad one.`
- `Fixture: committed golden schemas; test fails if drift is uncommitted.`

---

## Phase 3: Data Ingestion & Semantic Normalisation

### Purpose
Connect external HRIS sources to the canonical ontology through a normalisation layer that absorbs schema drift. After this phase, an organisation can import its workforce via CSV and BambooHR (the MVP connectors from features.md), runs are logged, data quality is assessed, and the canonical tables are populated — the prerequisite for any modelling. The connector ABC makes Workday/SuccessFactors/Merge additive later.

### Tasks

#### 3.1 — Connector ABC & normaliser

**What**: A connector interface and a declarative field-mapping normaliser.

**Design**:
```python
# src/phra/ingestion/base.py
class Connector(ABC):
    provider: str
    @abstractmethod
    async def fetch_employees(self) -> Iterable[dict]: ...
    @abstractmethod
    async def fetch_compensation(self) -> Iterable[dict]: ...
    @abstractmethod
    async def fetch_performance(self) -> Iterable[dict]: ...

# normaliser.py — declarative mapping insulates models from schema drift
@dataclass
class FieldMap: source: str; target: str; transform: Callable | None = None
def normalise(record: dict, mappings: list[FieldMap]) -> dict: ...
```
- Each connector ships a default `list[FieldMap]`; admins can override per connection (stored in `hris_connections.config`, encrypted at app layer).
- Normalised records are validated against the Phase 2 Pydantic schemas before upsert; failures recorded, not fatal.

**Testing**:
- `Unit: normalise() applies rename + transform (e.g. 'Hire Dt' "03/15/2024" → hire_date date).`
- `Unit: unmapped required field → record flagged invalid with field name.`

#### 3.2 — CSV connector

**What**: Upload-based CSV ingestion with a column-mapping wizard payload.

**Design**: `POST /connections/csv/import` (multipart) with a `mapping` JSON body of `FieldMap`s. Streams rows, normalises, validates, upserts by `external_hris_id`. Returns a `SyncResult{records_synced, records_failed, errors[]}`.

**Testing**:
- `Integration: 100-row valid CSV → 100 synced, 0 failed.`
- `Integration: CSV with 5 rows missing hire_date → 95 synced, 5 failed with row+field in errors.`
- `Fixture: committed sample_employees.csv used across tests.`

#### 3.3 — BambooHR connector

**What**: REST connector using HTTP Basic auth (API key as username, RFC 7617).

**Design**: Implements `Connector` against `https://api.bamboohr.com/api/gateway.php/{subdomain}/v1`. Endpoints: employee directory, custom report for compensation/performance. Pagination + rate-limit backoff. Default `FieldMap`s map BambooHR fields to canonical.

**Testing**:
- `Integration (mocked httpx): directory response → employees normalised and upserted.`
- `Integration (mocked): 429 then 200 → retried with backoff, succeeds.`
- `Integration (mocked): invalid API key → connection status set 'error', sync_log failed.`

#### 3.4 — Sync orchestration & logging

**What**: `hris_connections`, `sync_logs`, and a Celery task per sync.

**Design**: DDL per suggestion-1 (§ Data Ingestion). `run_sync(connection_id)` Celery task writes a `sync_logs` row (running→success/partial/failed), updates `last_sync_at`. `POST /connections/{id}/sync` enqueues; `GET /connections/{id}/syncs` lists history. Beat schedules periodic syncs.

**Testing**:
- `Integration: triggering sync enqueues task and returns 202 with sync_log id.`
- `Integration: partial failures → status 'partial', records_failed>0.`

#### 3.5 — Data quality assessment

**What**: Pre-modelling completeness/longitudinality report (mitigation for the "insufficient data quality" risk in research.md).

**Design**:
```python
@dataclass
class DataQualityReport:
    employee_count: int
    months_of_history: float
    completeness: dict[str, float]   # e.g. {'compensation':0.92,'performance':0.61}
    individual_model_eligible: bool  # >=500 active employees AND >=18 months history
    cohort_model_eligible: bool
    warnings: list[str]
```
`GET /organisations/me/data-quality` surfaces this; the modelling layer reads `individual_model_eligible` to decide individual vs cohort scope honestly (README/features.md sub-500 requirement).

**Testing**:
- `Unit: 300 employees, 24mo → individual_model_eligible False, cohort True, warning about scale.`
- `Unit: 600 employees, 10mo → individual False (history), warning about history depth.`
- `Integration: report reflects seeded fixture counts.`

---

## Phase 4: Feature Store & Flight-Risk Model (Core Value Proposition)

### Purpose
Ship the heart of the product: the flight-risk model with explainability. After this phase the platform extracts longitudinal feature vectors, trains a calibrated flight-risk model, scores employees (individual or cohort depending on data eligibility), persists scores with risk bands, and produces SHAP-based top-3 plain-language explanations — satisfying the MVP must-haves and the GDPR Art. 22 explainability requirement.

### Tasks

#### 4.1 — Feature store

**What**: Deterministic feature extraction into a `ml_feature_vectors` table.

**Design**: Implements the feature set from suggestion-2's `proj_ml_feature_vectors` as a relational table populated by `features/store.py`:
```python
def extract_features(org_id: UUID, as_of: date) -> pd.DataFrame:
    # columns: tenure_months, comp_to_market_ratio, months_since_last_raise,
    # months_since_promotion, manager_tenure_months, manager_changes_2yr,
    # perf_rating_current, perf_rating_previous, perf_trend, engagement_score,
    # engagement_delta_90d, meeting_count_avg, cross_team_interactions_avg,
    # response_time_avg, competency_gap_count, department_attrition_rate_12m
```
- All features are point-in-time `as_of` to prevent target leakage (no post-termination data).
- Feature definitions documented with ISO 30414 metric references where applicable.

**Testing**:
- `Unit: tenure_months computed from hire_date to as_of.`
- `Unit: as_of excludes events after the cutoff (no leakage).`
- `Integration: extract over fixture org → DataFrame with all 16 columns, no NaN in required fields.`

#### 4.2 — Flight-risk model training

**What**: `prediction_models` registry + XGBoost training pipeline with probability calibration.

**Design**: DDL per suggestion-1 (§ Prediction Models). Label = voluntary termination within horizon (default 180 days) derived from `EmployeeTerminated`/termination fields. Pipeline: impute → scale → XGBoost → `CalibratedClassifierCV`. Persists model artifact, `auc_score`, `precision_score`, `recall_score`, `feature_list`, sets `is_active`. Training is a Celery task.
```python
def train_flight_risk(org_id: UUID, horizon_days: int = 180) -> ModelMetrics: ...
```

**Testing**:
- `Unit: label builder marks employees terminated within horizon as positive, voluntary only.`
- `Integration: train on synthetic separable fixture → AUC > 0.8, model row persisted is_active.`
- `Integration: insufficient positives (<20) → raises InsufficientDataError, no model activated.`

#### 4.3 — Risk bands & calibration

**What**: Map calibrated probability to `risk_band` (low/moderate/high/critical) with org-configurable thresholds.

**Design**: Defaults: <0.15 low, <0.35 moderate, <0.60 high, else critical. Thresholds stored per org. `classify_risk(score) -> RiskBand`.

**Testing**:
- `Unit: 0.62 → critical; 0.15 → moderate (boundary).`
- `Unit: custom thresholds override defaults.`

#### 4.4 — Batch scoring (individual & cohort)

**What**: Score active employees and persist to `prediction_scores`.

**Design**: DDL per suggestion-1 (§ prediction_scores). If `individual_model_eligible`, score per employee (`score_type='individual'`); otherwise aggregate to department/role cohort scores (`score_type='cohort'`/`'role'`) — the honest sub-500 offering. `score_employees(org_id)` Celery task scheduled by `scoring_cron`.

**Testing**:
- `Integration: eligible org → one individual score per active employee.`
- `Integration: ineligible org → cohort scores per department, no individual rows.`
- `Integration: terminated employees excluded from scoring.`

#### 4.5 — SHAP explanations & plain language

**What**: Per-score top-N feature attributions with human-readable text.

**Design**: DDL per suggestion-1 (§ prediction_explanations). `explain(model, feature_vector) -> list[Explanation]` via SHAP; rank by |shap_value|; map feature→template:
```python
TEMPLATES = {
 "comp_to_market_ratio": "Compensation is {pct}% {dir} market median",
 "engagement_delta_90d": "Engagement has {dir} {pts} points over 90 days",
 "months_since_promotion": "{n} months since last promotion",
 ...
}
```
Stores `feature_name`, `feature_value`, `shap_value`, `contribution_rank`, `plain_language`. Top 3 surfaced by default (MVP requirement).

**Testing**:
- `Unit: explanation list sorted by |shap_value| desc; ranks 1..N.`
- `Unit: comp ratio 0.85 → "Compensation is 15% below market median".`
- `Integration: every persisted score has >=3 explanations.`

#### 4.6 — Flight-risk API

**What**: Endpoints exposing scores and explanations (OpenAPI 3.1, RBAC-scoped).

**Design**:
- `GET /flight-risk?department_id=&risk_band=&page=` → paginated scored employees (HR_ADMIN org-wide; MANAGER tree-only).
- `GET /flight-risk/{employee_id}` → score + ranked explanations + factor narrative slot.
- `POST /models/flight-risk/train` (ANALYST/HR_ADMIN) → enqueues training.

**Testing**:
- `Integration: HR admin lists with risk_band=critical → only critical rows.`
- `Integration: manager scoping enforced (only reports returned).`
- `Integration: response validates against generated OpenAPI schema.`

---

## Phase 5: Role-Based Dashboards (Frontend MVP)

### Purpose
Deliver the three role-based UIs (HR admin, manager, employee) that make the predictions usable — the MVP front door. After this phase, HR admins see org-wide flight risk, managers see their team, and employees see their own transparency view (the differentiating employee-facing explainability from features.md). Can be developed in parallel with Phase 6 once Phase 4 APIs exist.

### Tasks

#### 5.1 — Frontend scaffold & generated API client

**What**: Vite + React + TS app with an OpenAPI-generated typed client and auth flow.

**Design**: `openapi-typescript` generates `api/types.ts` from the FastAPI spec; TanStack Query wraps fetches. Auth via OIDC redirect; role drives routing. shadcn/ui + Tailwind theme.

**Testing**:
- `Component (RTL): unauthenticated user → redirected to /auth/login.`
- `Component: role=employee → admin routes not rendered in nav.`

#### 5.2 — HR admin dashboard

**What**: Org-wide flight-risk overview with segmentation and drill-down.

**Design**: Risk-band distribution (Recharts), filter by department/location/risk band, sortable table, click-through to employee detail. Progressive disclosure (summary cards → detail), mirroring incumbent UX patterns from features.md.

**Testing**:
- `Component: filter critical → table shows only critical rows (mocked API).`
- `E2E (Playwright): admin logs in → sees distribution → filters → opens detail.`

#### 5.3 — Manager dashboard

**What**: Team-scoped view with action prompts.

**Design**: "N people on your team are at risk" summary + per-report cards showing band, top-3 factors, and recommended action slot (Phase 6). Designed for non-HR users (Peakon-style).

**Testing**:
- `Component: renders only direct reports (mocked tree).`
- `E2E: manager cannot navigate to a non-report detail (404 view).`

#### 5.4 — Employee self-service transparency

**What**: Employee view of the factors used in their own assessment.

**Design**: Plain-language explanation of top contributing factors, what data is used and why, and a "flag inaccuracy" form (research.md trust mitigation; EU AI Act worker transparency). No raw score shown unless org enables it.

**Testing**:
- `Component: employee sees own factors, not numeric model internals when disclosure disabled.`
- `E2E: employee submits inaccuracy flag → recorded, confirmation shown.`

---

## Phase 6: Prescriptive Action Layer & Outcome Tracking

### Purpose
Build the genuine differentiator identified across research.md and features.md: not just scores, but prioritised recommended actions with owners and deadlines, plus systematic tracking of whether actions were taken and whether outcomes improved — feeding back into the model. Requires Phase 4 (scores) and benefits from Phase 5 (UI surfaces).

### Tasks

#### 6.1 — Recommendation engine

**What**: Generate prioritised `recommended_actions` from a score + its explanations.

**Design**: DDL per suggestion-1 (§ Prescriptive Actions). Rule-driven mapping from dominant explanation factor → action type:
```python
RULES = [
 ("comp_to_market_ratio<0.9", ActionType.COMP_ADJUSTMENT, priority=1),
 ("engagement_delta_90d<-5", ActionType.STAY_INTERVIEW, priority=1),
 ("months_since_promotion>24", ActionType.DEV_PLAN, priority=2),
 ("manager_changes_2yr>=2", ActionType.MANAGER_CHECKIN, priority=2),
]
def recommend(score_id) -> list[RecommendedAction]: ...  # assigned_to defaults to manager
```
Status lifecycle: `pending → approved → in_progress → completed | declined`. Human approval gate enforced before "acting" (research.md mitigation; EU AI Act human oversight).

**Testing**:
- `Unit: comp ratio 0.8 dominant → COMP_ADJUSTMENT priority 1 assigned to manager.`
- `Integration: actions created in 'pending'; cannot transition to in_progress without approved_by.`

#### 6.2 — Action APIs & approval gate

**What**: CRUD + state transition endpoints.

**Design**: `GET /actions?assigned_to=&status=`, `POST /actions/{id}/approve`, `POST /actions/{id}/decline`, `PATCH /actions/{id}` (status). Approval requires HR_ADMIN or MANAGER in scope; transitions validated against the state machine.

**Testing**:
- `Integration: approve by manager of the employee → 200; by unrelated manager → 404.`
- `Integration: illegal transition (completed→pending) → 409.`

#### 6.3 — Outcome tracking & feedback loop

**What**: `action_outcomes` + a job measuring score delta 90 days post-action.

**Design**: DDL per suggestion-1 (§ action_outcomes). Scheduled task computes `prediction_score_after_90d - score_at_action` per completed action, records `outcome_type`. An `action_effectiveness` aggregate (by action_type) feeds Phase 8 model retraining as a signal (which interventions reduce risk).

**Testing**:
- `Integration: completed action with later improved score → outcome_type='positive', score_delta negative (risk down).`
- `Integration: effectiveness aggregate groups by action_type with mean delta.`

---

## Phase 7: Fairness, Compliance & Explainability Governance

### Purpose
Make the platform defensible under the EU AI Act (high-risk Annex III), GDPR Article 22, ISO/IEC 42001, and NIST AI RMF. After this phase every model run is fairness-audited across protected attributes, blocked from activation if it fails, every prediction is provenance-tracked, and GDPR Art. 22 explanation requests can be fulfilled with auto-drafted documentation. This is a hard gate, not optional, per standards.md.

### Tasks

#### 7.1 — Fairness audit (Fairlearn)

**What**: `fairness_audits` computed per model across protected attributes.

**Design**: DDL per suggestion-1 (§ fairness_audits). For each of `gender/ethnicity/age_band/disability` (where consented data exists), compute `demographic_parity_difference` and `equalized_odds_difference` via Fairlearn; `pass = max(diff) <= threshold` (default 0.1). Runs automatically after every training.

**Testing**:
- `Unit: synthetic biased predictions → demographic_parity_diff above threshold, pass=False.`
- `Unit: balanced predictions → pass=True.`
- `Integration: audit rows persisted per protected attribute after training.`

#### 7.2 — Activation gate

**What**: Block activation of a model that fails fairness.

**Design**: `activate_model` checks the latest audit; if any `pass=False`, refuses activation and records a governance note. Override requires HR_ADMIN with a documented justification (audited).

**Testing**:
- `Integration: model failing audit → activation refused with 409 + reason.`
- `Integration: documented override → activated, override logged.`

#### 7.3 — Provenance & audit log (W3C PROV-DM aligned)

**What**: `prediction_runs` linking each score to model version + feature snapshot + run metadata, plus an append-only audit log.

**Design**: Each scoring run writes a `prediction_runs` row (run_id, model_id, feature_as_of, triggered_by, started/completed). Scores reference run_id. Audit log records who accessed/exported individual PII (OWASP API #1 monitoring + EU AI Act traceability).

**Testing**:
- `Integration: a score can be traced to its run, model version, and feature as_of date.`
- `Integration: exporting an individual's data writes an audit entry with actor + timestamp.`

#### 7.4 — GDPR Art. 22 explanation requests + LLM compliance drafting

**What**: `explanation_requests` workflow with auto-drafted plain-language explanations.

**Design**: DDL per suggestion-1 (§ explanation_requests). `POST /explanation-requests` (employee or HR on behalf) → pulls the relevant score + explanations + provenance → `llm/compliance.py` drafts a GDPR Art. 22 / EU AI Act response (human-reviewed before sending). Works with `llm_provider=none` by falling back to a templated explanation.

**Testing**:
- `Integration: request for a known score → draft contains top factors + plain-language logic.`
- `Integration: llm_provider=none → templated (non-LLM) explanation still produced.`

#### 7.5 — Survey open-text NLP

**What**: Transformer-based sentiment + theme clustering of survey free-text (features.md AI-augmentation candidate).

**Design**: `sentence-transformers` embeds `survey_responses.text_value`; cluster (HDBSCAN) into themes; LLM labels clusters and assigns sentiment. Results stored as JSONB analytics, surfaced as engagement drivers. Metadata-only / aggregate to preserve anonymity.

**Testing**:
- `Unit: embedding + clustering on fixture responses → stable cluster count.`
- `Integration: clusters labelled; per-team aggregation suppresses groups < k respondents (anonymity).`

---

## Phase 8: Promotion Readiness, Team Health & Workforce Forecasting

### Purpose
Round out the predictive suite beyond flight risk: promotion readiness/succession, team-conflict signals from collaboration metadata, and 6–18 month workforce forecasting. Reuses the Phase 4 model registry, explainability, and Phase 7 fairness gates. Requires Phases 2–4 and 7.

### Tasks

#### 8.1 — Promotion readiness & succession

**What**: Readiness model + succession bench.

**Design**: DDL per suggestion-1 (§ Succession). Readiness combines performance trajectory, competency coverage vs target position, cross-functional experience, peer/manager feedback → `readiness_score` and band (ready_now/ready_1yr/ready_2yr/developmental). `succession_plans`/`succession_candidates` rank a bench per critical position; LLM drafts a per-candidate rationale (features.md). Fairness-audited like flight risk.

**Testing**:
- `Unit: candidate meeting all competency requirements with rising perf → ready_now.`
- `Integration: bench ranked by readiness_score desc per plan.`
- `Integration: succession model passes through the Phase 7 activation gate.`

#### 8.2 — Team health / conflict signals

**What**: Aggregate collaboration-metadata model flagging rising team risk.

**Design**: Features from `collaboration_snapshots` aggregated to team: declining cross-team interactions, rising response latency, meeting overload. Outputs team-level `prediction_scores` (`score_type='cohort'`). Strictly metadata-only (GDPR Art. 6).

**Testing**:
- `Unit: team with rising latency + falling cross-team interaction → elevated conflict score.`
- `Integration: no individual-level conflict scores produced (team aggregate only).`

#### 8.3 — Workforce capacity forecasting

**What**: 6–18 month headcount/skill-gap forecasts.

**Design**: DDL per suggestion-1 (§ workforce_forecasts). Project headcount by department/location from current headcount, predicted attrition (from flight-risk aggregates), and internal mobility trends; identify `hiring_need` and `skill_gap_areas`; attach `confidence`. `GET /forecasts?department_id=&horizon_months=`.

**Testing**:
- `Unit: headcount_predicted = current - predicted_attrition + planned_hires.`
- `Integration: forecast rows persisted per department for requested horizon.`

---

## Phase 9: Advanced HRIS Connectors & Unified-API Option

### Purpose
Extend ingestion to the enterprise sources (Workday, SAP SuccessFactors) and offer the Merge.dev unified-API adapter as an alternative to maintaining many connectors. Additive to Phase 3; the connector ABC means no rework. Requires Phase 3.

### Tasks

#### 9.1 — Workday connector

**What**: OAuth 2.0 (Authorization Code) connector over Workday REST + RaaS, with SOAP fallback for objects missing from REST.

**Design**: ISU/OAuth2 client per standards.md; default `FieldMap`s map worker/org/comp objects to canonical; tolerant of per-release field renames (drift handled by overridable mappings + validation, not code changes).

**Testing**:
- `Integration (mocked): RaaS report → employees normalised; renamed field handled via mapping override.`
- `Integration (mocked): token refresh on 401 → retried.`

#### 9.2 — SAP SuccessFactors connector

**What**: OData v4 connector over Employee Central.

**Design**: Implements OData query semantics ($select/$filter/$expand/$top/$skip) for paginated extraction; OAuth 2.0 SAML Bearer / client credentials.

**Testing**:
- `Integration (mocked): paged OData response ($top/$skip) → all pages consumed.`
- `Integration (mocked): $expand on employment nav property → nested data normalised.`

#### 9.3 — Merge.dev unified adapter

**What**: Optional single adapter mapping Merge's unified HRIS model to canonical.

**Design**: One `MergeConnector` covering 200+ providers via Merge's normalised schema; account-token per linked integration. Offered as the fast path for self-hosters who prefer not to run direct connectors.

**Testing**:
- `Integration (mocked): Merge employees payload → canonical employees.`
- `Integration: switching a connection from BambooHR-direct to Merge yields equivalent canonical records (fixture parity test).`

---

## Phase 10: Compensation Benchmarking, NL Query (MCP) & Packaging

### Purpose
Complete the v1.1 scope and harden for release: live compensation benchmarking as a predictive feedback into flight risk, a natural-language workforce query interface via an MCP server, and full self-host packaging with governance documentation. Requires Phases 4, 7, and 9.

### Tasks

#### 10.1 — Compensation benchmarking integration

**What**: Ingest market salary data (Levels.fyi API / Radford/Mercer CSV) and surface comp drift as a flight-risk driver.

**Design**: Connector populates `market_benchmarks`; a scheduled job flags employees below a configurable market percentile; the comp gap already flows into the feature store (§4.1), closing the loop features.md flagged as missing in incumbents.

**Testing**:
- `Integration: ingesting benchmark CSV updates market_benchmarks; comp_to_market_ratio recomputed.`
- `Integration: employee crossing below P25 → flagged in a comp-drift report.`

#### 10.2 — MCP server for NL workforce queries

**What**: Read-only, tenancy-scoped MCP server exposing aggregate workforce data to an LLM assistant.

**Design**: Implements MCP (standards.md) with tools like `query_flight_risk(filters)`, `get_team_health(team_id)`, `get_forecast(dept, horizon)`. Enforces the same RBAC/tenancy as the API; never exposes individual PII to managers outside their tree. Powers the "ask a question about your workforce" feature provider-agnostically.

**Testing**:
- `Integration: MCP tool call scoped to caller's org only; cross-org query returns empty.`
- `Integration: manager-scoped session cannot retrieve out-of-tree individuals via MCP.`

#### 10.3 — Self-host packaging & governance docs

**What**: Production docker-compose, `.env.example`, backup/restore notes, and ISO/IEC 42001 + NIST AI RMF documentation templates.

**Design**: One-command `docker compose up` brings up the full stack with migrations auto-applied; a `docs/governance/` set covers AIMS risk assessment, bias-testing procedure (Phase 7), and human-oversight policy, mapping to EU AI Act high-risk obligations.

**Testing**:
- `E2E: fresh `docker compose up` → migrate → seed → CSV import → train → score → dashboard loads (full smoke).`
- `Integration: OpenAPI spec served at /openapi.json validates as OAS 3.1.`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (config, db, tenancy, OIDC, RBAC)   ─── required by everything
    │
Phase 2: Canonical HR Ontology                          ─── requires Phase 1
    │
Phase 3: Ingestion & Normalisation (CSV, BambooHR)      ─── requires Phase 2
    │
Phase 4: Feature Store + Flight-Risk Model + SHAP       ─── requires Phase 3   ★ core value
    ├── Phase 5: Role-Based Dashboards (FE MVP)          ─── requires Phase 4 (parallel with 6)
    └── Phase 6: Prescriptive Actions + Outcomes         ─── requires Phase 4 (parallel with 5)
         │
Phase 7: Fairness, Compliance, Provenance, Survey NLP   ─── requires Phase 4 (gates 4 & 8)
         │
Phase 8: Promotion / Team Health / Forecasting          ─── requires Phases 2–4, 7
         │
Phase 9: Advanced Connectors (Workday, SF, Merge)       ─── requires Phase 3 (parallel with 5–8)
         │
Phase 10: Comp Benchmarking + MCP NL Query + Packaging  ─── requires Phases 4, 7, 9
```

**Parallelism opportunities:**
- Phases 5 and 6 can be built concurrently once Phase 4 lands.
- Phase 9 (additional connectors) can proceed in parallel with Phases 5–8 since the connector ABC is fixed in Phase 3.
- Phase 7's fairness gate should land before Phase 8 ships any new model to production, but its survey-NLP task (7.5) is independent and can be parallelised.

---

## Definition of Done (per phase)

A phase is complete only when all of the following hold:

1. All tasks in the phase are implemented.
2. All unit and integration tests pass; new code paths are covered.
3. `ruff` lint and format pass; `mypy --strict` passes (frontend: eslint + prettier + tsc).
4. `docker compose up` builds and starts all affected services.
5. The feature works end-to-end against a seeded fixture organisation.
6. New configuration options are documented in `.env.example`.
7. New API endpoints appear in the auto-generated OpenAPI 3.1 spec and validate.
8. Alembic migrations are created, applied cleanly, and reversible.
9. Any new model is fairness-audited and passes the Phase 7 activation gate before being marked `is_active` (Phases 4, 8 onward).
10. Any feature touching individual PII writes to the provenance/audit log and enforces tenancy + RBAC scoping (OWASP API #1).
