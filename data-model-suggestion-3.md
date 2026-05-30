# Data Model Suggestion 3: Hybrid Relational + JSONB Model (PostgreSQL)

## Overview

A PostgreSQL schema that keeps core entities (employees, departments, positions) in
traditional relational tables with strict foreign keys, while using JSONB columns for
semi-structured, frequently changing, or deeply nested data: survey responses, prediction
explanations, collaboration metadata, competency assessments, and HRIS connector
configurations. GIN indexes on JSONB columns enable efficient querying without
sacrificing the flexibility to evolve schemas incrementally.

## Why This Approach Suits Predictive HR Analytics

Predictive HR analytics sits at the intersection of structured transactional data
(employee records, compensation, org hierarchy) and semi-structured analytical data
(SHAP explanations, survey open-text themes, collaboration signal bundles, model
feature vectors). The hybrid approach addresses both:

- **Stable entities stay relational**: Employee, department, and position tables change
  infrequently in structure and benefit from referential integrity, typed columns, and
  standard indexing.
- **Volatile/nested data uses JSONB**: Prediction explanations vary by model version.
  Collaboration metadata differs by source (Slack vs. Teams vs. Calendar). Survey
  question schemas change every campaign. JSONB absorbs this variability without
  migrations.
- **Single database**: Unlike polyglot persistence, everything runs in one PostgreSQL
  instance, simplifying operations, backups, transactions, and compliance (GDPR
  deletion is a single-database operation).

## Trade-offs

**Strengths:**
- Schema flexibility where it matters, rigidity where it protects data quality.
- No migration needed when a new HRIS connector delivers unexpected fields -- they
  land in a JSONB column and can be promoted to typed columns later if needed.
- GIN indexes on JSONB provide performant queries on semi-structured data.
- Single-database simplicity for deployment, backup, and ACID transactions.
- Easy to evolve from the normalized model (Suggestion 1) by adding JSONB columns.

**Weaknesses:**
- JSONB columns lack schema enforcement at the database level -- validation must
  happen in the application layer or via CHECK constraints with jsonb_typeof.
- GIN indexes are larger and slower to update than B-tree indexes on typed columns.
- Complex JSONB queries (deep nesting, array element filtering) can be harder to
  optimize and debug than standard SQL.
- Reporting tools and BI platforms may not handle JSONB columns natively.

## Schema Definition

```sql
-- ============================================================
-- CORE RELATIONAL ENTITIES (strict types, foreign keys)
-- ============================================================

CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    industry        TEXT,
    employee_count  INT,
    settings        JSONB NOT NULL DEFAULT '{}',   -- org-level config: timezone, locale, feature flags
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE departments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    parent_dept_id  UUID REFERENCES departments(id),
    cost_centre     TEXT,
    metadata        JSONB NOT NULL DEFAULT '{}',   -- custom fields from HRIS
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, name)
);

CREATE TABLE locations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    country_code    CHAR(2) NOT NULL,
    timezone        TEXT NOT NULL DEFAULT 'UTC',
    geo             JSONB,                          -- { "lat": 37.77, "lng": -122.41, "address": "..." }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE positions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    title           TEXT NOT NULL,
    level           TEXT,
    band            TEXT,
    is_critical     BOOLEAN NOT NULL DEFAULT false,
    competency_requirements JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"competency": "leadership", "required_level": 4}, ...]
    -- Allows flexible competency frameworks without a join table
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_positions_competencies ON positions USING GIN (competency_requirements);

CREATE TABLE employees (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    external_ids    JSONB NOT NULL DEFAULT '{}',
    -- { "workday": "WD-12345", "bamboohr": "BHR-789", "hibob": "HB-456" }
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
    demographics    JSONB NOT NULL DEFAULT '{}',
    -- { "gender": "female", "age_band": "30-39", "ethnicity": "...", ... }
    -- Stored as JSONB because: (a) protected characteristics vary by jurisdiction,
    -- (b) not all orgs collect the same fields, (c) access must be tightly controlled
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    -- Catch-all for HRIS-specific fields that don't map to core columns
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_employees_org ON employees(organisation_id);
CREATE INDEX idx_employees_dept ON employees(department_id);
CREATE INDEX idx_employees_manager ON employees(manager_id);
CREATE INDEX idx_employees_status ON employees(organisation_id, status);
CREATE INDEX idx_employees_external_ids ON employees USING GIN (external_ids);
CREATE INDEX idx_employees_demographics ON employees USING GIN (demographics);

-- ============================================================
-- COMPENSATION (relational core + JSONB for variable components)
-- ============================================================

CREATE TABLE compensation_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id     UUID NOT NULL REFERENCES employees(id) ON DELETE CASCADE,
    effective_date  DATE NOT NULL,
    base_salary     NUMERIC(12,2) NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    variable_comp   JSONB NOT NULL DEFAULT '{}',
    -- { "bonus_target_pct": 15, "equity_grant": 50000, "signing_bonus": 10000,
    --   "relocation": 5000, "stock_type": "RSU", "vesting_schedule": "4yr_1yr_cliff" }
    change_reason   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_comp_employee ON compensation_records(employee_id, effective_date DESC);

CREATE TABLE market_benchmarks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    position_id     UUID NOT NULL REFERENCES positions(id) ON DELETE CASCADE,
    location_id     UUID REFERENCES locations(id),
    source          TEXT NOT NULL,
    percentiles     JSONB NOT NULL,
    -- { "p10": 85000, "p25": 95000, "p50": 110000, "p75": 130000, "p90": 155000 }
    effective_date  DATE NOT NULL,
    raw_data        JSONB,               -- original source payload for traceability
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_benchmarks_position ON market_benchmarks(position_id, effective_date DESC);

-- ============================================================
-- PERFORMANCE REVIEWS
-- ============================================================

CREATE TABLE review_cycles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    config          JSONB NOT NULL DEFAULT '{}',
    -- { "rating_scale": 5, "includes_peer": true, "includes_self": true }
    status          TEXT NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft','open','closed','archived'))
);

CREATE TABLE performance_reviews (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_cycle_id UUID NOT NULL REFERENCES review_cycles(id) ON DELETE CASCADE,
    employee_id     UUID NOT NULL REFERENCES employees(id) ON DELETE CASCADE,
    reviewer_id     UUID REFERENCES employees(id),
    overall_rating  SMALLINT CHECK (overall_rating BETWEEN 1 AND 5),
    rating_label    TEXT,
    feedback        JSONB NOT NULL DEFAULT '{}',
    -- { "strengths": "...", "areas_for_growth": "...", "goals_met": ["g1","g2"],
    --   "competency_ratings": {"leadership": 4, "technical": 3, ...} }
    submitted_at    TIMESTAMPTZ,
    UNIQUE (review_cycle_id, employee_id, reviewer_id)
);

CREATE INDEX idx_perf_reviews_emp ON performance_reviews(employee_id, submitted_at DESC);
CREATE INDEX idx_perf_feedback ON performance_reviews USING GIN (feedback);

-- ============================================================
-- ENGAGEMENT SURVEYS (JSONB-heavy -- schema varies per campaign)
-- ============================================================

CREATE TABLE survey_campaigns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    survey_type     TEXT NOT NULL CHECK (survey_type IN ('pulse','annual','onboarding','exit')),
    question_schema JSONB NOT NULL,
    -- Array of question definitions:
    -- [{ "id": "q1", "text": "How engaged do you feel?", "type": "likert",
    --    "category": "engagement", "scale": 5 }, ...]
    launched_at     TIMESTAMPTZ,
    closed_at       TIMESTAMPTZ
);

CREATE TABLE survey_responses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id     UUID NOT NULL REFERENCES survey_campaigns(id) ON DELETE CASCADE,
    employee_id     UUID NOT NULL REFERENCES employees(id) ON DELETE CASCADE,
    responses       JSONB NOT NULL,
    -- { "q1": 4, "q2": 3, "q3_text": "I feel undervalued because...",
    --   "q4_choice": "somewhat_agree" }
    computed_scores JSONB,
    -- { "overall": 3.8, "engagement": 4.0, "manager": 3.5, "growth": 4.2 }
    sentiment_analysis JSONB,
    -- { "themes": ["compensation", "career_growth"], "sentiment": "mixed",
    --   "key_phrases": ["need more mentorship", "salary below market"] }
    responded_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (campaign_id, employee_id)
);

CREATE INDEX idx_survey_resp_emp ON survey_responses(employee_id, responded_at DESC);
CREATE INDEX idx_survey_resp_scores ON survey_responses USING GIN (computed_scores);
CREATE INDEX idx_survey_resp_sentiment ON survey_responses USING GIN (sentiment_analysis);

-- ============================================================
-- COLLABORATION METADATA
-- ============================================================

CREATE TABLE collaboration_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id     UUID NOT NULL REFERENCES employees(id) ON DELETE CASCADE,
    snapshot_date   DATE NOT NULL,
    source          TEXT NOT NULL,
    metrics         JSONB NOT NULL,
    -- { "meeting_count_week": 12, "avg_response_time_hrs": 1.5,
    --   "cross_team_interactions": 8, "unique_collaborators": 15,
    --   "after_hours_messages": 3, "focus_time_hrs": 22,
    --   "network_centrality": 0.72 }
    -- JSONB because different sources provide different metrics
    UNIQUE (employee_id, snapshot_date, source)
);

CREATE INDEX idx_collab_snap ON collaboration_snapshots(employee_id, snapshot_date DESC);
CREATE INDEX idx_collab_metrics ON collaboration_snapshots USING GIN (metrics);

-- ============================================================
-- PREDICTION MODELS & SCORES
-- ============================================================

CREATE TABLE prediction_models (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    model_type      TEXT NOT NULL CHECK (model_type IN (
                    'flight_risk','promotion_readiness','team_conflict','workforce_forecast')),
    version         INT NOT NULL DEFAULT 1,
    algorithm       TEXT,
    config          JSONB NOT NULL DEFAULT '{}',
    -- { "feature_list": [...], "hyperparameters": {...}, "training_window_months": 24 }
    metrics         JSONB NOT NULL DEFAULT '{}',
    -- { "auc": 0.87, "precision": 0.82, "recall": 0.79, "f1": 0.80,
    --   "feature_importance": {"tenure": 0.15, "comp_ratio": 0.12, ...} }
    is_active       BOOLEAN NOT NULL DEFAULT false,
    trained_at      TIMESTAMPTZ,
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
    explanations    JSONB NOT NULL DEFAULT '[]',
    -- [{ "feature": "comp_to_market_ratio", "value": 0.82, "shap": -0.15,
    --    "rank": 1, "plain_language": "Compensation is 18% below market median" },
    --  { "feature": "engagement_trend", "value": -0.3, "shap": -0.12,
    --    "rank": 2, "plain_language": "Engagement score declined 30% over 2 quarters" }]
    feature_vector  JSONB,
    -- The complete input feature vector used for this prediction, enabling
    -- point-in-time reproducibility without replaying event streams
    scored_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pred_scores_emp ON prediction_scores(employee_id, scored_at DESC);
CREATE INDEX idx_pred_scores_dept ON prediction_scores(department_id, scored_at DESC);
CREATE INDEX idx_pred_scores_band ON prediction_scores(model_id, risk_band);
CREATE INDEX idx_pred_explanations ON prediction_scores USING GIN (explanations);

-- ============================================================
-- FAIRNESS & COMPLIANCE
-- ============================================================

CREATE TABLE fairness_audits (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_id        UUID NOT NULL REFERENCES prediction_models(id) ON DELETE CASCADE,
    audit_date      DATE NOT NULL,
    results         JSONB NOT NULL,
    -- { "gender": { "demographic_parity_diff": 0.03, "equalized_odds_diff": 0.05, "pass": true },
    --   "ethnicity": { "demographic_parity_diff": 0.07, "equalized_odds_diff": 0.09, "pass": false },
    --   "age_band": { ... } }
    overall_pass    BOOLEAN NOT NULL,
    notes           TEXT
);

CREATE TABLE explanation_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id     UUID NOT NULL REFERENCES employees(id) ON DELETE CASCADE,
    prediction_score_id UUID NOT NULL REFERENCES prediction_scores(id),
    regulation      TEXT,
    requested_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    fulfilled_at    TIMESTAMPTZ,
    response        JSONB,
    -- { "explanation_text": "...", "contributing_factors": [...],
    --   "model_version": 3, "data_sources_used": [...] }
    status          TEXT NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending','fulfilled','escalated'))
);

-- ============================================================
-- PRESCRIPTIVE ACTIONS
-- ============================================================

CREATE TABLE recommended_actions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    prediction_score_id UUID NOT NULL REFERENCES prediction_scores(id) ON DELETE CASCADE,
    action_type     TEXT NOT NULL,
    description     TEXT NOT NULL,
    priority        SMALLINT NOT NULL CHECK (priority BETWEEN 1 AND 5),
    assigned_to     UUID REFERENCES employees(id),
    deadline        DATE,
    status          TEXT NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending','approved','in_progress','completed','declined')),
    approval        JSONB,
    -- { "approved_by": "uuid", "approved_at": "2025-03-15T10:00:00Z",
    --   "approval_notes": "Proceed with market adjustment" }
    outcome         JSONB,
    -- { "outcome_type": "positive", "metrics": { "engagement_delta": 0.4,
    --   "retention_90d": true }, "completed_at": "2025-06-15", "notes": "..." }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rec_actions_status ON recommended_actions(status, deadline);
CREATE INDEX idx_rec_actions_assignee ON recommended_actions(assigned_to, status);
CREATE INDEX idx_rec_actions_outcome ON recommended_actions USING GIN (outcome);

-- ============================================================
-- SUCCESSION PLANNING
-- ============================================================

CREATE TABLE succession_plans (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    position_id     UUID NOT NULL REFERENCES positions(id) ON DELETE CASCADE,
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active','archived')),
    candidates      JSONB NOT NULL DEFAULT '[]',
    -- [{ "employee_id": "uuid", "readiness": "ready_1yr", "readiness_score": 0.72,
    --    "competency_gaps": ["strategic_planning", "budget_management"],
    --    "bench_rank": 1, "development_plan": "..." }, ...]
    impact_assessment JSONB,
    -- { "revenue_at_risk": 2500000, "team_size_affected": 12,
    --   "key_relationships": ["client_acme", "partner_xyz"],
    --   "knowledge_concentration_risk": "high" }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- WORKFORCE FORECASTING
-- ============================================================

CREATE TABLE workforce_forecasts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    department_id   UUID REFERENCES departments(id),
    location_id     UUID REFERENCES locations(id),
    forecast_date   DATE NOT NULL,
    horizon_months  SMALLINT NOT NULL,
    forecast_data   JSONB NOT NULL,
    -- { "headcount_current": 145, "headcount_predicted": 132,
    --   "attrition_predicted": 0.09, "hiring_need": 18,
    --   "confidence": 0.82, "confidence_interval": [125, 139],
    --   "skill_gaps": [{"skill": "ML engineering", "gap_count": 3, "priority": "high"}],
    --   "scenario_optimistic": { "headcount": 140, ... },
    --   "scenario_pessimistic": { "headcount": 122, ... } }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_wf_forecast ON workforce_forecasts(organisation_id, department_id, forecast_date);
CREATE INDEX idx_wf_forecast_data ON workforce_forecasts USING GIN (forecast_data);

-- ============================================================
-- DATA INGESTION & HRIS SYNC
-- ============================================================

CREATE TABLE hris_connections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    provider        TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active','paused','error')),
    last_sync_at    TIMESTAMPTZ,
    config          JSONB NOT NULL DEFAULT '{}',
    -- { "api_endpoint": "https://...", "auth_method": "oauth2",
    --   "field_mapping": { "workday_worker_id": "external_ids.workday", ... },
    --   "sync_schedule": "0 2 * * *", "filters": {"active_only": true} }
    -- Encrypted at the application layer before storage
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE sync_logs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    connection_id   UUID NOT NULL REFERENCES hris_connections(id) ON DELETE CASCADE,
    started_at      TIMESTAMPTZ NOT NULL,
    completed_at    TIMESTAMPTZ,
    status          TEXT NOT NULL CHECK (status IN ('running','success','partial','failed')),
    summary         JSONB NOT NULL DEFAULT '{}',
    -- { "records_synced": 1250, "records_failed": 3, "records_skipped": 12,
    --   "errors": [{"employee_id": "...", "error": "missing required field hire_date"}],
    --   "field_mapping_warnings": ["unknown field 'custom_attr_7' ignored"] }
    error_message   TEXT
);

-- ============================================================
-- COMPETENCY ASSESSMENTS (JSONB approach)
-- ============================================================

CREATE TABLE employee_assessments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id     UUID NOT NULL REFERENCES employees(id) ON DELETE CASCADE,
    assessment_type TEXT NOT NULL CHECK (assessment_type IN ('competency','360','self','skill_test')),
    assessed_at     DATE NOT NULL,
    assessed_by     UUID REFERENCES employees(id),
    results         JSONB NOT NULL,
    -- Competency: { "leadership": 4, "technical": 3, "communication": 5, ... }
    -- 360 feedback: { "manager": {...}, "peers": [{...}], "direct_reports": [{...}] }
    -- Skill test: { "test_name": "SQL proficiency", "score": 87, "percentile": 72 }
    source          TEXT CHECK (source IN ('self','manager','peer','360','system'))
);

CREATE INDEX idx_emp_assess ON employee_assessments(employee_id, assessed_at DESC);
CREATE INDEX idx_emp_assess_results ON employee_assessments USING GIN (results);
```

## Example Queries

```sql
-- Find high flight-risk employees with compensation below market
SELECT e.first_name, e.last_name, d.name AS department,
       ps.score AS flight_risk, ps.risk_band,
       ps.explanations->0->>'plain_language' AS top_reason
FROM employees e
JOIN departments d ON e.department_id = d.id
JOIN prediction_scores ps ON ps.employee_id = e.id
WHERE ps.risk_band IN ('high', 'critical')
  AND ps.scored_at = (SELECT MAX(scored_at) FROM prediction_scores WHERE employee_id = e.id)
  AND ps.explanations @> '[{"feature": "comp_to_market_ratio"}]'
ORDER BY ps.score DESC;

-- Find employees whose engagement sentiment mentions "compensation"
SELECT e.id, e.first_name, e.last_name,
       sr.sentiment_analysis->'themes' AS themes,
       sr.computed_scores->>'overall' AS engagement_score
FROM employees e
JOIN survey_responses sr ON sr.employee_id = e.id
WHERE sr.sentiment_analysis->'themes' ? 'compensation'
ORDER BY (sr.computed_scores->>'overall')::numeric ASC;
```

## Scalability Considerations

- **JSONB column sizing**: Keep JSONB documents under 1MB. For prediction explanations
  and feature vectors, this is rarely a concern (typically 1-5KB).
- **Partial GIN indexes**: If only a subset of JSONB fields are queried, use expression
  indexes: `CREATE INDEX ON prediction_scores USING GIN ((explanations->0->'feature'))`.
- **TOAST compression**: PostgreSQL automatically compresses JSONB values larger than 2KB
  using TOAST, which is transparent to queries.
- **Partitioning**: `prediction_scores` and `survey_responses` should be partitioned by
  `scored_at`/`responded_at` for large deployments.
- **Schema validation**: Use application-layer JSON Schema validation or PostgreSQL
  CHECK constraints with `jsonb_typeof()` to enforce structure on critical JSONB columns.

## Migration Path

This model is the natural evolution of the normalized schema (Suggestion 1). Migration
involves: (1) adding JSONB columns to existing tables, (2) migrating data from
normalized child tables into JSONB, (3) adding GIN indexes, (4) dropping the
now-redundant child tables. This can be done incrementally, table by table, with no
downtime using PostgreSQL's transactional DDL.

Moving toward event sourcing (Suggestion 2) is also straightforward: add an event log
table alongside this schema, emit events from application code on every write, and
gradually shift authority to the event store while these tables become read projections.
