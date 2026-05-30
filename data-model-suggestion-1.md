# Data Model Suggestion 1: Normalized Relational Model (PostgreSQL)

## Overview

A traditional Third Normal Form (3NF+) relational schema in PostgreSQL. Every entity
occupies its own table with strict foreign-key constraints, composite indexes tuned for
the dominant query patterns (flight-risk scoring, promotion readiness, team-health
dashboards), and check constraints that enforce business invariants at the database level.

## Why This Approach Suits Predictive HR Analytics

HR data is inherently relational: employees belong to departments, report to managers,
hold positions defined by competency frameworks, receive performance ratings on review
cycles, and respond to engagement surveys over time. A normalized model captures these
relationships explicitly, prevents update anomalies (e.g. changing a department name in
one place), and makes ad-hoc analytical queries straightforward with standard SQL joins.

Predictive models need clean, deduplicated feature tables. A 3NF schema provides a
single source of truth for each attribute, which simplifies the feature-extraction
pipelines that feed ML models for flight-risk and promotion-readiness scoring.

## Trade-offs

**Strengths:**
- Strong referential integrity ensures data quality -- critical when model accuracy
  depends on clean inputs.
- Mature tooling: every BI platform, ETL tool, and ML pipeline speaks SQL.
- ACID transactions protect multi-table writes (e.g. recording a performance review
  and updating the employee's latest score atomically).
- Straightforward compliance: GDPR deletion requests map to DELETE cascades.

**Weaknesses:**
- Schema rigidity. Adding a new signal source (e.g. a new collaboration tool)
  requires a migration.
- Wide joins for analytics queries. A flight-risk feature vector may require joining
  8-10 tables, which needs careful indexing.
- Storing semi-structured data (e.g. survey open-text themes, SHAP explanations) is
  awkward without JSONB escape hatches.

## Schema Definition

```sql
-- ============================================================
-- CORE ENTITIES
-- ============================================================

CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    industry        TEXT,
    employee_count  INT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE departments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    parent_dept_id  UUID REFERENCES departments(id),
    cost_centre     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, name)
);

CREATE TABLE locations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    country_code    CHAR(2) NOT NULL,
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE positions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    title           TEXT NOT NULL,
    level           TEXT,               -- e.g. IC3, M1, VP
    band            TEXT,               -- compensation band
    is_critical     BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE employees (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    external_hris_id TEXT,              -- ID from source HRIS (Workday, BambooHR, etc.)
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    email           TEXT,
    hire_date       DATE NOT NULL,
    termination_date DATE,
    termination_reason TEXT,
    department_id   UUID REFERENCES departments(id),
    position_id     UUID REFERENCES positions(id),
    location_id     UUID REFERENCES locations(id),
    manager_id      UUID REFERENCES employees(id),
    employment_type TEXT NOT NULL DEFAULT 'full_time'
                    CHECK (employment_type IN ('full_time','part_time','contractor','intern')),
    status          TEXT NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active','on_leave','terminated','retired')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_employees_org ON employees(organisation_id);
CREATE INDEX idx_employees_dept ON employees(department_id);
CREATE INDEX idx_employees_manager ON employees(manager_id);
CREATE INDEX idx_employees_status ON employees(organisation_id, status);
CREATE INDEX idx_employees_hire_date ON employees(organisation_id, hire_date);

-- ============================================================
-- COMPETENCY & SKILL FRAMEWORK
-- ============================================================

CREATE TABLE competency_frameworks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    version         INT NOT NULL DEFAULT 1,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE competencies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    framework_id    UUID NOT NULL REFERENCES competency_frameworks(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    category        TEXT,               -- e.g. 'technical', 'leadership', 'domain'
    description     TEXT
);

CREATE TABLE position_competency_requirements (
    position_id     UUID NOT NULL REFERENCES positions(id) ON DELETE CASCADE,
    competency_id   UUID NOT NULL REFERENCES competencies(id) ON DELETE CASCADE,
    required_level  SMALLINT NOT NULL CHECK (required_level BETWEEN 1 AND 5),
    PRIMARY KEY (position_id, competency_id)
);

CREATE TABLE employee_competency_assessments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id     UUID NOT NULL REFERENCES employees(id) ON DELETE CASCADE,
    competency_id   UUID NOT NULL REFERENCES competencies(id) ON DELETE CASCADE,
    assessed_level  SMALLINT NOT NULL CHECK (assessed_level BETWEEN 1 AND 5),
    assessed_by     UUID REFERENCES employees(id),
    assessed_at     DATE NOT NULL,
    source          TEXT CHECK (source IN ('self','manager','peer','360','system'))
);

CREATE INDEX idx_emp_comp_assess ON employee_competency_assessments(employee_id, competency_id, assessed_at DESC);

-- ============================================================
-- COMPENSATION
-- ============================================================

CREATE TABLE compensation_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id     UUID NOT NULL REFERENCES employees(id) ON DELETE CASCADE,
    effective_date  DATE NOT NULL,
    base_salary     NUMERIC(12,2) NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    bonus_target_pct NUMERIC(5,2),
    equity_grant    NUMERIC(12,2),
    total_comp      NUMERIC(12,2) GENERATED ALWAYS AS
                    (base_salary + COALESCE(base_salary * bonus_target_pct / 100, 0) + COALESCE(equity_grant, 0)) STORED,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_comp_employee ON compensation_records(employee_id, effective_date DESC);

CREATE TABLE market_benchmarks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    position_id     UUID NOT NULL REFERENCES positions(id) ON DELETE CASCADE,
    location_id     UUID REFERENCES locations(id),
    source          TEXT NOT NULL,       -- 'levels_fyi', 'radford', 'mercer', 'manual'
    percentile_25   NUMERIC(12,2),
    percentile_50   NUMERIC(12,2),
    percentile_75   NUMERIC(12,2),
    effective_date  DATE NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- PERFORMANCE REVIEWS
-- ============================================================

CREATE TABLE review_cycles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    status          TEXT NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft','open','closed','archived'))
);

CREATE TABLE performance_reviews (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_cycle_id UUID NOT NULL REFERENCES review_cycles(id) ON DELETE CASCADE,
    employee_id     UUID NOT NULL REFERENCES employees(id) ON DELETE CASCADE,
    reviewer_id     UUID REFERENCES employees(id),
    overall_rating  SMALLINT CHECK (overall_rating BETWEEN 1 AND 5),
    rating_label    TEXT,                -- 'exceeds', 'meets', 'below', etc.
    manager_comments TEXT,
    submitted_at    TIMESTAMPTZ,
    UNIQUE (review_cycle_id, employee_id, reviewer_id)
);

CREATE INDEX idx_perf_reviews_emp ON performance_reviews(employee_id, submitted_at DESC);

-- ============================================================
-- ENGAGEMENT SURVEYS
-- ============================================================

CREATE TABLE survey_campaigns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    survey_type     TEXT NOT NULL CHECK (survey_type IN ('pulse','annual','onboarding','exit')),
    launched_at     TIMESTAMPTZ,
    closed_at       TIMESTAMPTZ
);

CREATE TABLE survey_questions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id     UUID NOT NULL REFERENCES survey_campaigns(id) ON DELETE CASCADE,
    question_text   TEXT NOT NULL,
    category        TEXT,                -- 'engagement', 'manager', 'culture', 'growth'
    question_type   TEXT NOT NULL CHECK (question_type IN ('likert','open_text','nps','multiple_choice')),
    sort_order      INT NOT NULL DEFAULT 0
);

CREATE TABLE survey_responses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    question_id     UUID NOT NULL REFERENCES survey_questions(id) ON DELETE CASCADE,
    employee_id     UUID NOT NULL REFERENCES employees(id) ON DELETE CASCADE,
    numeric_value   SMALLINT,
    text_value      TEXT,
    responded_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_survey_resp_emp ON survey_responses(employee_id, responded_at DESC);
CREATE INDEX idx_survey_resp_question ON survey_responses(question_id);

-- ============================================================
-- COLLABORATION METADATA
-- ============================================================

CREATE TABLE collaboration_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id     UUID NOT NULL REFERENCES employees(id) ON DELETE CASCADE,
    snapshot_date   DATE NOT NULL,
    meeting_count_week      INT,
    avg_response_time_hrs   NUMERIC(6,2),
    cross_team_interactions  INT,
    unique_collaborators    INT,
    communication_volume    INT,         -- messages sent (metadata only, no content)
    source                  TEXT,        -- 'slack_metadata', 'teams_metadata', 'calendar'
    UNIQUE (employee_id, snapshot_date, source)
);

CREATE INDEX idx_collab_snap ON collaboration_snapshots(employee_id, snapshot_date DESC);

-- ============================================================
-- PREDICTION MODELS & SCORES
-- ============================================================

CREATE TABLE prediction_models (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    model_type      TEXT NOT NULL CHECK (model_type IN ('flight_risk','promotion_readiness','team_conflict','workforce_forecast')),
    version         INT NOT NULL DEFAULT 1,
    algorithm       TEXT,                -- 'xgboost', 'logistic_regression', 'transformer'
    feature_list    TEXT[],
    training_date   TIMESTAMPTZ,
    auc_score       NUMERIC(5,4),
    precision_score NUMERIC(5,4),
    recall_score    NUMERIC(5,4),
    is_active       BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, model_type, version)
);

CREATE TABLE prediction_scores (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_id        UUID NOT NULL REFERENCES prediction_models(id) ON DELETE CASCADE,
    employee_id     UUID REFERENCES employees(id) ON DELETE CASCADE,
    department_id   UUID REFERENCES departments(id),
    score_type      TEXT NOT NULL CHECK (score_type IN ('individual','cohort','role')),
    score           NUMERIC(5,4) NOT NULL CHECK (score BETWEEN 0 AND 1),
    risk_band       TEXT CHECK (risk_band IN ('low','moderate','high','critical')),
    scored_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pred_scores_emp ON prediction_scores(employee_id, scored_at DESC);
CREATE INDEX idx_pred_scores_dept ON prediction_scores(department_id, scored_at DESC);
CREATE INDEX idx_pred_scores_band ON prediction_scores(model_id, risk_band);

CREATE TABLE prediction_explanations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    prediction_score_id UUID NOT NULL REFERENCES prediction_scores(id) ON DELETE CASCADE,
    feature_name    TEXT NOT NULL,
    feature_value   TEXT,
    shap_value      NUMERIC(8,6),
    contribution_rank SMALLINT NOT NULL,
    plain_language  TEXT                 -- "Compensation is 15% below market median"
);

CREATE INDEX idx_pred_expl ON prediction_explanations(prediction_score_id, contribution_rank);

-- ============================================================
-- FAIRNESS & COMPLIANCE
-- ============================================================

CREATE TABLE fairness_audits (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_id        UUID NOT NULL REFERENCES prediction_models(id) ON DELETE CASCADE,
    audit_date      DATE NOT NULL,
    protected_attribute TEXT NOT NULL,   -- 'gender', 'ethnicity', 'age_band', 'disability'
    demographic_parity_diff NUMERIC(5,4),
    equalized_odds_diff     NUMERIC(5,4),
    pass            BOOLEAN NOT NULL,
    notes           TEXT
);

CREATE TABLE explanation_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id     UUID NOT NULL REFERENCES employees(id) ON DELETE CASCADE,
    prediction_score_id UUID NOT NULL REFERENCES prediction_scores(id),
    requested_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    fulfilled_at    TIMESTAMPTZ,
    explanation_text TEXT,
    regulation      TEXT                 -- 'gdpr_art22', 'eu_ai_act', 'other'
);

-- ============================================================
-- PRESCRIPTIVE ACTIONS
-- ============================================================

CREATE TABLE recommended_actions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    prediction_score_id UUID NOT NULL REFERENCES prediction_scores(id) ON DELETE CASCADE,
    action_type     TEXT NOT NULL,        -- 'stay_interview', 'comp_adjustment', 'dev_plan', 'team_restructure'
    description     TEXT NOT NULL,
    priority        SMALLINT NOT NULL CHECK (priority BETWEEN 1 AND 5),
    assigned_to     UUID REFERENCES employees(id),
    deadline        DATE,
    status          TEXT NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending','approved','in_progress','completed','declined')),
    approved_by     UUID REFERENCES employees(id),
    approved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rec_actions_status ON recommended_actions(status, deadline);
CREATE INDEX idx_rec_actions_assignee ON recommended_actions(assigned_to, status);

CREATE TABLE action_outcomes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    action_id       UUID NOT NULL REFERENCES recommended_actions(id) ON DELETE CASCADE,
    outcome_date    DATE NOT NULL,
    outcome_type    TEXT NOT NULL CHECK (outcome_type IN ('positive','neutral','negative')),
    metric_name     TEXT,                -- 'engagement_delta', 'retention_90d', 'performance_delta'
    metric_value    NUMERIC(8,4),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- SUCCESSION PLANNING
-- ============================================================

CREATE TABLE succession_plans (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    position_id     UUID NOT NULL REFERENCES positions(id) ON DELETE CASCADE,
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active','archived')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE succession_candidates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    plan_id         UUID NOT NULL REFERENCES succession_plans(id) ON DELETE CASCADE,
    employee_id     UUID NOT NULL REFERENCES employees(id) ON DELETE CASCADE,
    readiness       TEXT NOT NULL CHECK (readiness IN ('ready_now','ready_1yr','ready_2yr','developmental')),
    readiness_score NUMERIC(5,4),
    competency_gap_count INT DEFAULT 0,
    bench_rank      SMALLINT,
    notes           TEXT,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (plan_id, employee_id)
);

-- ============================================================
-- WORKFORCE FORECASTING
-- ============================================================

CREATE TABLE workforce_forecasts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    department_id   UUID REFERENCES departments(id),
    location_id     UUID REFERENCES locations(id),
    forecast_date   DATE NOT NULL,       -- the future date being forecasted
    horizon_months  SMALLINT NOT NULL,
    headcount_current INT NOT NULL,
    headcount_predicted INT NOT NULL,
    attrition_predicted NUMERIC(5,4),
    hiring_need     INT,
    skill_gap_areas TEXT[],
    confidence      NUMERIC(5,4),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_wf_forecast ON workforce_forecasts(organisation_id, department_id, forecast_date);

-- ============================================================
-- DATA INGESTION & HRIS SYNC
-- ============================================================

CREATE TABLE hris_connections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    provider        TEXT NOT NULL,        -- 'workday', 'bamboohr', 'successfactors', 'hibob', 'csv'
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active','paused','error')),
    last_sync_at    TIMESTAMPTZ,
    config          TEXT,                -- connection config (encrypted at app layer)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE sync_logs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    connection_id   UUID NOT NULL REFERENCES hris_connections(id) ON DELETE CASCADE,
    started_at      TIMESTAMPTZ NOT NULL,
    completed_at    TIMESTAMPTZ,
    records_synced  INT DEFAULT 0,
    records_failed  INT DEFAULT 0,
    status          TEXT NOT NULL CHECK (status IN ('running','success','partial','failed')),
    error_message   TEXT
);
```

## Scalability Considerations

- **Partitioning**: `prediction_scores`, `survey_responses`, and `collaboration_snapshots`
  should be range-partitioned by date once volumes exceed ~50M rows. PostgreSQL native
  partitioning supports this without application changes.
- **Read replicas**: Analytical queries (dashboard aggregations, feature extraction) should
  target a read replica to avoid contention with transactional writes from HRIS syncs.
- **Materialized views**: Pre-compute common aggregations (rolling 90-day engagement
  average, compensation-to-market ratio) as materialized views refreshed on a schedule.

## Migration Path

This schema can evolve toward the Hybrid JSONB model (Suggestion 3) by adding JSONB
columns to specific tables (e.g. `prediction_explanations.metadata`) without breaking
existing queries. If event-sourcing becomes necessary for audit trails, an event log
table can be layered alongside these relational tables without a full rewrite.
