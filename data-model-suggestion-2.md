# Data Model Suggestion 2: Event-Sourced / CQRS Model

## Overview

An event-sourced architecture with Command Query Responsibility Segregation (CQRS).
Every state change in the HR domain is captured as an immutable event in an append-only
event store. Commands validate business rules and emit events. Read-side projections
materialize denormalized views optimized for dashboards, ML feature extraction, and
compliance queries. The event store serves as the single source of truth; all read
models are derivable from replaying the event log.

## Why This Approach Suits Predictive HR Analytics

Predictive HR analytics is fundamentally about understanding how employee signals
change over time. Event sourcing captures every transition -- a compensation change,
a manager reassignment, a dip in engagement score -- as a first-class, timestamped,
immutable record. This provides three critical advantages:

1. **Complete temporal history**: ML models need longitudinal feature vectors (e.g.
   "engagement trend over last 4 quarters"). Event sourcing preserves every data point
   naturally, without CDC or slowly-changing-dimension hacks.
2. **Full audit trail**: GDPR Article 22 and the EU AI Act require organisations to
   explain how predictions were derived. Event sourcing lets you reconstruct the exact
   state of an employee's data at the moment a prediction was generated.
3. **Feedback loops**: The prescriptive action layer needs to track whether an
   intervention was taken and whether outcomes improved. Events like `ActionAssigned`,
   `ActionCompleted`, `EngagementScoreChanged` create a causal chain that the model
   can learn from.

## Trade-offs

**Strengths:**
- Immutable audit log satisfies regulatory requirements without separate audit tables.
- Temporal queries are trivial: replay events up to any point in time.
- Read models can be rebuilt from scratch if requirements change.
- Natural fit for real-time streaming (Kafka/EventStoreDB) to trigger alerts.
- Supports multiple optimized read models for different consumers (dashboards vs. ML
  pipelines vs. compliance reports).

**Weaknesses:**
- Higher implementation complexity than a simple relational schema.
- Event schema evolution requires careful versioning (upcasting).
- Eventual consistency between write and read sides may confuse users expecting
  immediate reflection of changes.
- GDPR right-to-erasure conflicts with immutability -- requires crypto-shredding or
  tombstone events with careful design.
- Debugging production issues requires tooling to inspect event streams.

## Event Store Schema

```sql
-- ============================================================
-- EVENT STORE (append-only, single source of truth)
-- ============================================================

CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type  TEXT NOT NULL,        -- 'Employee', 'Department', 'ReviewCycle', 'PredictionRun'
    aggregate_id    UUID NOT NULL,
    event_type      TEXT NOT NULL,        -- 'EmployeeHired', 'CompensationChanged', etc.
    event_version   INT NOT NULL,         -- schema version for upcasting
    payload         JSONB NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',  -- correlation_id, causation_id, user_id
    organisation_id UUID NOT NULL,
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    sequence_number BIGSERIAL NOT NULL
);

-- Ensure ordering within an aggregate
CREATE UNIQUE INDEX idx_event_aggregate_seq
    ON event_store(aggregate_id, sequence_number);

-- Fast lookup by type for projection rebuilds
CREATE INDEX idx_event_type ON event_store(event_type, occurred_at);

-- Organisation scoping
CREATE INDEX idx_event_org ON event_store(organisation_id, occurred_at);

-- Partition by month for large deployments
-- CREATE TABLE event_store PARTITION BY RANGE (occurred_at);
```

## Domain Events Catalogue

```yaml
# ── Employee Lifecycle ──────────────────────────────────────
EmployeeHired:
  aggregate: Employee
  payload:
    employee_id: uuid
    organisation_id: uuid
    first_name: string
    last_name: string
    email: string
    hire_date: date
    department_id: uuid
    position_id: uuid
    location_id: uuid
    manager_id: uuid
    employment_type: enum(full_time, part_time, contractor, intern)
    external_hris_id: string

EmployeeTransferred:
  aggregate: Employee
  payload:
    employee_id: uuid
    from_department_id: uuid
    to_department_id: uuid
    from_position_id: uuid
    to_position_id: uuid
    effective_date: date

ManagerChanged:
  aggregate: Employee
  payload:
    employee_id: uuid
    previous_manager_id: uuid
    new_manager_id: uuid
    effective_date: date

EmployeeTerminated:
  aggregate: Employee
  payload:
    employee_id: uuid
    termination_date: date
    reason: string
    voluntary: boolean

# ── Compensation ────────────────────────────────────────────
CompensationChanged:
  aggregate: Employee
  payload:
    employee_id: uuid
    previous_base_salary: decimal
    new_base_salary: decimal
    currency: string
    bonus_target_pct: decimal
    equity_grant: decimal
    effective_date: date
    change_reason: enum(merit, promotion, adjustment, market)

MarketBenchmarkUpdated:
  aggregate: Position
  payload:
    position_id: uuid
    location_id: uuid
    source: string
    percentile_25: decimal
    percentile_50: decimal
    percentile_75: decimal
    effective_date: date

# ── Performance ─────────────────────────────────────────────
PerformanceReviewSubmitted:
  aggregate: Employee
  payload:
    employee_id: uuid
    review_cycle_id: uuid
    reviewer_id: uuid
    overall_rating: int
    rating_label: string
    comments: string
    submitted_at: timestamp

CompetencyAssessed:
  aggregate: Employee
  payload:
    employee_id: uuid
    competency_id: uuid
    assessed_level: int
    assessed_by: uuid
    source: enum(self, manager, peer, 360, system)

# ── Engagement ──────────────────────────────────────────────
SurveyResponseRecorded:
  aggregate: Employee
  payload:
    employee_id: uuid
    campaign_id: uuid
    question_id: uuid
    numeric_value: int
    text_value: string

EngagementScoreComputed:
  aggregate: Employee
  payload:
    employee_id: uuid
    campaign_id: uuid
    overall_score: decimal
    category_scores:
      engagement: decimal
      manager: decimal
      culture: decimal
      growth: decimal

# ── Collaboration Metadata ──────────────────────────────────
CollaborationSnapshotRecorded:
  aggregate: Employee
  payload:
    employee_id: uuid
    snapshot_date: date
    meeting_count_week: int
    avg_response_time_hrs: decimal
    cross_team_interactions: int
    unique_collaborators: int
    source: string

# ── Predictions ─────────────────────────────────────────────
PredictionModelTrained:
  aggregate: PredictionRun
  payload:
    model_id: uuid
    organisation_id: uuid
    model_type: enum(flight_risk, promotion_readiness, team_conflict, workforce_forecast)
    algorithm: string
    feature_list: string[]
    auc_score: decimal
    precision_score: decimal
    recall_score: decimal

PredictionScoreGenerated:
  aggregate: PredictionRun
  payload:
    prediction_id: uuid
    model_id: uuid
    employee_id: uuid
    score: decimal
    risk_band: enum(low, moderate, high, critical)
    explanations:
      - feature_name: string
        shap_value: decimal
        plain_language: string

FairnessAuditCompleted:
  aggregate: PredictionRun
  payload:
    model_id: uuid
    protected_attribute: string
    demographic_parity_diff: decimal
    equalized_odds_diff: decimal
    pass: boolean

# ── Prescriptive Actions ───────────────────────────────────
ActionRecommended:
  aggregate: Employee
  payload:
    action_id: uuid
    prediction_id: uuid
    action_type: string
    description: string
    priority: int
    assigned_to: uuid
    deadline: date

ActionApproved:
  aggregate: Employee
  payload:
    action_id: uuid
    approved_by: uuid

ActionCompleted:
  aggregate: Employee
  payload:
    action_id: uuid
    outcome_type: enum(positive, neutral, negative)
    metric_name: string
    metric_value: decimal
    notes: string

ActionDeclined:
  aggregate: Employee
  payload:
    action_id: uuid
    declined_by: uuid
    reason: string

# ── Succession ──────────────────────────────────────────────
SuccessionCandidateNominated:
  aggregate: Position
  payload:
    plan_id: uuid
    position_id: uuid
    employee_id: uuid
    readiness: enum(ready_now, ready_1yr, ready_2yr, developmental)
    readiness_score: decimal

SuccessionReadinessUpdated:
  aggregate: Position
  payload:
    plan_id: uuid
    employee_id: uuid
    previous_readiness: string
    new_readiness: string
    readiness_score: decimal

# ── GDPR / Compliance ──────────────────────────────────────
ExplanationRequested:
  aggregate: Employee
  payload:
    employee_id: uuid
    prediction_id: uuid
    regulation: string

EmployeeDataErased:
  aggregate: Employee
  payload:
    employee_id: uuid
    erasure_method: enum(crypto_shred, tombstone)
    encryption_key_id: uuid   # key that was destroyed
```

## Command Handlers (Pseudocode)

```python
class ChangeCompensationHandler:
    def handle(self, cmd: ChangeCompensation):
        employee = self.repository.load(cmd.employee_id)

        # Business rules
        if employee.status != 'active':
            raise InvalidOperation("Cannot change compensation for inactive employee")
        if cmd.new_base_salary <= 0:
            raise ValidationError("Salary must be positive")

        employee.apply(CompensationChanged(
            employee_id=cmd.employee_id,
            previous_base_salary=employee.current_salary,
            new_base_salary=cmd.new_base_salary,
            currency=cmd.currency,
            bonus_target_pct=cmd.bonus_target_pct,
            equity_grant=cmd.equity_grant,
            effective_date=cmd.effective_date,
            change_reason=cmd.reason,
        ))
        self.repository.save(employee)


class GeneratePredictionsHandler:
    def handle(self, cmd: GeneratePredictions):
        model = self.model_registry.get_active(cmd.organisation_id, cmd.model_type)
        employees = self.employee_projection.get_active(cmd.organisation_id)
        features = self.feature_store.extract(employees, model.feature_list)

        for employee_id, feature_vector in features.items():
            score, explanations = model.predict(feature_vector)
            self.event_store.append(PredictionScoreGenerated(
                prediction_id=uuid4(),
                model_id=model.id,
                employee_id=employee_id,
                score=score,
                risk_band=classify_risk(score),
                explanations=explanations,
            ))
```

## Read-Side Projections

```sql
-- ============================================================
-- PROJECTION: Current Employee State (materialized from events)
-- ============================================================

CREATE TABLE proj_employee_current (
    employee_id     UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    first_name      TEXT,
    last_name       TEXT,
    email           TEXT,
    hire_date       DATE,
    department_id   UUID,
    position_id     UUID,
    location_id     UUID,
    manager_id      UUID,
    status          TEXT,
    current_base_salary  NUMERIC(12,2),
    current_total_comp   NUMERIC(12,2),
    latest_perf_rating   SMALLINT,
    latest_engagement_score NUMERIC(5,2),
    latest_flight_risk_score NUMERIC(5,4),
    latest_flight_risk_band  TEXT,
    tenure_months   INT,
    manager_tenure_months INT,
    last_event_at   TIMESTAMPTZ,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_emp_org ON proj_employee_current(organisation_id);
CREATE INDEX idx_proj_emp_risk ON proj_employee_current(organisation_id, latest_flight_risk_band);

-- ============================================================
-- PROJECTION: Flight Risk Dashboard
-- ============================================================

CREATE TABLE proj_flight_risk_dashboard (
    employee_id     UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    employee_name   TEXT,
    department_name TEXT,
    position_title  TEXT,
    manager_name    TEXT,
    flight_risk_score NUMERIC(5,4),
    risk_band       TEXT,
    top_factor_1    TEXT,
    top_factor_2    TEXT,
    top_factor_3    TEXT,
    comp_to_market_ratio NUMERIC(5,2),
    engagement_trend TEXT,               -- 'rising', 'stable', 'declining'
    tenure_months   INT,
    last_promotion_months_ago INT,
    scored_at       TIMESTAMPTZ
);

-- ============================================================
-- PROJECTION: ML Feature Store (denormalized for model training)
-- ============================================================

CREATE TABLE proj_ml_feature_vectors (
    employee_id         UUID NOT NULL,
    snapshot_date       DATE NOT NULL,
    tenure_months       INT,
    comp_to_market_ratio NUMERIC(5,4),
    months_since_last_raise INT,
    months_since_promotion INT,
    manager_tenure_months INT,
    manager_changes_2yr INT,
    perf_rating_current SMALLINT,
    perf_rating_previous SMALLINT,
    perf_trend          NUMERIC(5,2),
    engagement_score    NUMERIC(5,2),
    engagement_delta_90d NUMERIC(5,2),
    meeting_count_avg   NUMERIC(6,2),
    cross_team_interactions_avg NUMERIC(6,2),
    response_time_avg   NUMERIC(6,2),
    competency_gap_count INT,
    department_attrition_rate_12m NUMERIC(5,4),
    PRIMARY KEY (employee_id, snapshot_date)
);

-- ============================================================
-- PROJECTION: Action Effectiveness (for feedback loop)
-- ============================================================

CREATE TABLE proj_action_effectiveness (
    action_id       UUID PRIMARY KEY,
    action_type     TEXT,
    employee_id     UUID,
    department_id   UUID,
    prediction_score_at_action NUMERIC(5,4),
    prediction_score_after_90d NUMERIC(5,4),
    score_delta     NUMERIC(5,4),
    outcome_type    TEXT,
    completed       BOOLEAN,
    time_to_complete_days INT,
    created_at      TIMESTAMPTZ
);
```

## GDPR Compliance: Crypto-Shredding Pattern

Since the event store is append-only, employee data cannot be deleted. Instead, use
crypto-shredding: all PII fields in event payloads are encrypted with a per-employee
key. When a GDPR erasure request is received, destroy the encryption key. The events
remain in the store but the PII is irrecoverable.

```
EmployeeHired payload (stored):
{
  "employee_id": "abc-123",
  "first_name": "ENC:AES256:keyref=emp-abc-123:...",
  "last_name":  "ENC:AES256:keyref=emp-abc-123:...",
  "email":      "ENC:AES256:keyref=emp-abc-123:...",
  "hire_date":  "2024-03-15",           // non-PII, stored in plain text
  "department_id": "dept-456",          // non-PII
  ...
}

On GDPR erasure: DELETE FROM encryption_keys WHERE employee_id = 'abc-123';
```

## Scalability Considerations

- **Event Store partitioning**: Partition `event_store` by `occurred_at` (monthly) for
  time-range queries and efficient archival of old partitions to cold storage.
- **Projection rebuilds**: Keep projections small and rebuildable. A full rebuild from
  1M events should complete in under 10 minutes with batch processing.
- **Streaming**: For real-time alerting (e.g. a high-risk employee's engagement drops
  further), pipe events through Kafka or EventStoreDB subscriptions to trigger
  immediate notifications.
- **Snapshotting**: For aggregates with long event histories (an employee with 10 years
  of data), periodically snapshot aggregate state to avoid replaying thousands of events
  on every load.

## Migration Path

An event-sourced system can coexist with a relational database during migration. The
relational tables become read-side projections maintained by event handlers. New
features are built event-first while legacy queries continue to hit the existing
projections. Over time, the relational schema becomes fully derived from the event
store, and the event store becomes the authoritative source.
