# Data Model Suggestion 4: Time-Series + Graph Hybrid (TimescaleDB + Apache AGE)

## Overview

A domain-specific architecture combining two paradigms that map naturally to predictive
HR analytics: **time-series storage** for the continuous workforce signals that feed ML
models (engagement scores, collaboration metrics, compensation changes, prediction
scores over time) and a **property graph** for the organisational relationships that
drive team health analysis and influence propagation (reporting chains, collaboration
networks, succession paths, cross-functional interactions).

Both run inside PostgreSQL: TimescaleDB as an extension for hypertable-based time-series
and Apache AGE (A Graph Extension) for Cypher-queryable property graphs. This avoids
polyglot persistence while leveraging purpose-built data structures for each concern.

## Why This Approach Suits Predictive HR Analytics

### Time-Series Fit

Predictive HR analytics is fundamentally about signals over time. Every core prediction
-- flight risk, promotion readiness, team conflict -- depends on temporal patterns:

- **Engagement trend**: Is the score rising, stable, or declining over the last 4 quarters?
- **Compensation drift**: How has the employee's pay-to-market ratio changed over 24 months?
- **Collaboration patterns**: Are cross-team interactions increasing or decreasing weekly?
- **Performance trajectory**: Three "meets expectations" followed by a "below" is a different
  signal than a single "below."

Time-series databases excel at: continuous aggregation (rolling averages, rate of change),
efficient storage via compression (10-20x for numeric signals), retention policies (auto-
archive data older than N years), and downsampling (convert weekly snapshots to monthly
for long-term storage).

### Graph Fit

Several core features require relationship traversal:

- **Manager chains**: Flight risk propagates -- when a popular manager departs, their
  direct reports' flight risk scores should spike. Graph traversal makes this query trivial.
- **Collaboration networks**: Identifying isolated employees or detecting rising conflict
  requires analyzing the shape of communication graphs (centrality, clustering coefficient).
- **Succession planning**: Finding the best internal candidate for a critical role requires
  traversing competency graphs, reporting hierarchies, and cross-functional experience paths.
- **Influence mapping**: Understanding which employees are organizational hubs (high
  betweenness centrality) helps prioritize retention efforts.

## Trade-offs

**Strengths:**
- Time-series queries (rolling averages, trend detection, rate of change) run orders of
  magnitude faster than equivalent SQL window functions on regular tables.
- Automatic compression: TimescaleDB achieves 10-20x compression on numeric time-series
  data, dramatically reducing storage costs for multi-year longitudinal datasets.
- Graph queries express relationship patterns (shortest path, centrality, community
  detection) naturally in Cypher rather than recursive CTEs.
- Continuous aggregation materializes real-time rollups without manual materialized views.
- Both extensions run inside PostgreSQL, so relational tables coexist seamlessly.

**Weaknesses:**
- Operational complexity: two PostgreSQL extensions to install, upgrade, and monitor.
- Apache AGE is less mature than dedicated graph databases (Neo4j, Neptune). Complex
  graph analytics (PageRank, Louvain community detection) may need external computation.
- TimescaleDB hypertables cannot have foreign keys pointing TO them, which changes
  referential integrity patterns for time-series tables.
- Team skill requirements: developers need Cypher literacy alongside SQL.
- Testing and local development require PostgreSQL with both extensions installed.

## Schema Definition

### Relational Core (Standard PostgreSQL Tables)

```sql
-- ============================================================
-- REFERENCE ENTITIES (standard relational tables)
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
    level           TEXT,
    band            TEXT,
    is_critical     BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE employees (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    external_hris_id TEXT,
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
    employment_type TEXT NOT NULL DEFAULT 'full_time',
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_employees_org ON employees(organisation_id);
CREATE INDEX idx_employees_dept ON employees(department_id);
CREATE INDEX idx_employees_manager ON employees(manager_id);
CREATE INDEX idx_employees_status ON employees(organisation_id, status);

-- Prediction models (metadata, not time-series)
CREATE TABLE prediction_models (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    model_type      TEXT NOT NULL,
    version         INT NOT NULL DEFAULT 1,
    algorithm       TEXT,
    feature_list    TEXT[],
    hyperparameters JSONB,
    auc_score       NUMERIC(5,4),
    precision_score NUMERIC(5,4),
    recall_score    NUMERIC(5,4),
    is_active       BOOLEAN NOT NULL DEFAULT false,
    trained_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, model_type, version)
);

-- HRIS connections (metadata)
CREATE TABLE hris_connections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    provider        TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'active',
    last_sync_at    TIMESTAMPTZ,
    config          JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Time-Series Tables (TimescaleDB Hypertables)

```sql
-- ============================================================
-- TIME-SERIES: Engagement Signals
-- ============================================================

CREATE TABLE ts_engagement_scores (
    time            TIMESTAMPTZ NOT NULL,
    employee_id     UUID NOT NULL,
    campaign_id     UUID,
    overall_score   NUMERIC(5,2),
    engagement      NUMERIC(5,2),
    manager_score   NUMERIC(5,2),
    culture_score   NUMERIC(5,2),
    growth_score    NUMERIC(5,2),
    nps             SMALLINT,
    response_count  INT DEFAULT 1
);

SELECT create_hypertable('ts_engagement_scores', 'time');
CREATE INDEX idx_ts_engage_emp ON ts_engagement_scores(employee_id, time DESC);

-- Continuous aggregate: rolling quarterly engagement by employee
CREATE MATERIALIZED VIEW cagg_engagement_quarterly
WITH (timescaledb.continuous) AS
SELECT employee_id,
       time_bucket('90 days', time) AS quarter,
       avg(overall_score) AS avg_score,
       avg(engagement) AS avg_engagement,
       avg(manager_score) AS avg_manager,
       count(*) AS data_points
FROM ts_engagement_scores
GROUP BY employee_id, time_bucket('90 days', time);

-- ============================================================
-- TIME-SERIES: Compensation History
-- ============================================================

CREATE TABLE ts_compensation (
    time            TIMESTAMPTZ NOT NULL,
    employee_id     UUID NOT NULL,
    base_salary     NUMERIC(12,2) NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    bonus_target_pct NUMERIC(5,2),
    equity_grant    NUMERIC(12,2),
    total_comp      NUMERIC(12,2),
    market_p50      NUMERIC(12,2),
    comp_to_market_ratio NUMERIC(5,4),
    change_reason   TEXT
);

SELECT create_hypertable('ts_compensation', 'time');
CREATE INDEX idx_ts_comp_emp ON ts_compensation(employee_id, time DESC);

-- ============================================================
-- TIME-SERIES: Collaboration Metrics
-- ============================================================

CREATE TABLE ts_collaboration (
    time                    TIMESTAMPTZ NOT NULL,
    employee_id             UUID NOT NULL,
    source                  TEXT NOT NULL,
    meeting_count           INT,
    avg_response_time_hrs   NUMERIC(6,2),
    cross_team_interactions  INT,
    unique_collaborators    INT,
    communication_volume    INT,
    after_hours_activity    INT,
    focus_time_hrs          NUMERIC(6,2),
    network_centrality      NUMERIC(5,4)
);

SELECT create_hypertable('ts_collaboration', 'time');
CREATE INDEX idx_ts_collab_emp ON ts_collaboration(employee_id, time DESC);

-- Continuous aggregate: weekly collaboration averages
CREATE MATERIALIZED VIEW cagg_collaboration_weekly
WITH (timescaledb.continuous) AS
SELECT employee_id,
       time_bucket('7 days', time) AS week,
       avg(meeting_count) AS avg_meetings,
       avg(avg_response_time_hrs) AS avg_response_time,
       avg(cross_team_interactions) AS avg_cross_team,
       avg(unique_collaborators) AS avg_collaborators,
       avg(network_centrality) AS avg_centrality
FROM ts_collaboration
GROUP BY employee_id, time_bucket('7 days', time);

-- ============================================================
-- TIME-SERIES: Performance Ratings
-- ============================================================

CREATE TABLE ts_performance (
    time            TIMESTAMPTZ NOT NULL,
    employee_id     UUID NOT NULL,
    review_cycle_id UUID,
    reviewer_id     UUID,
    overall_rating  SMALLINT,
    rating_label    TEXT,
    review_type     TEXT        -- 'manager', 'peer', 'self', '360'
);

SELECT create_hypertable('ts_performance', 'time');
CREATE INDEX idx_ts_perf_emp ON ts_performance(employee_id, time DESC);

-- ============================================================
-- TIME-SERIES: Prediction Scores (the output of ML models)
-- ============================================================

CREATE TABLE ts_prediction_scores (
    time            TIMESTAMPTZ NOT NULL,
    employee_id     UUID,
    department_id   UUID,
    model_id        UUID NOT NULL,
    model_type      TEXT NOT NULL,
    score_type      TEXT NOT NULL,       -- 'individual', 'cohort', 'role'
    score           NUMERIC(5,4) NOT NULL,
    risk_band       TEXT,
    top_factors     JSONB,              -- top 5 SHAP explanations
    feature_vector  JSONB               -- complete input for reproducibility
);

SELECT create_hypertable('ts_prediction_scores', 'time');
CREATE INDEX idx_ts_pred_emp ON ts_prediction_scores(employee_id, time DESC);
CREATE INDEX idx_ts_pred_band ON ts_prediction_scores(model_type, risk_band, time DESC);

-- Continuous aggregate: monthly risk distribution by department
CREATE MATERIALIZED VIEW cagg_risk_by_department
WITH (timescaledb.continuous) AS
SELECT department_id,
       model_type,
       time_bucket('30 days', time) AS month,
       count(*) FILTER (WHERE risk_band = 'critical') AS critical_count,
       count(*) FILTER (WHERE risk_band = 'high') AS high_count,
       count(*) FILTER (WHERE risk_band = 'moderate') AS moderate_count,
       count(*) FILTER (WHERE risk_band = 'low') AS low_count,
       avg(score) AS avg_score
FROM ts_prediction_scores
WHERE score_type = 'individual'
GROUP BY department_id, model_type, time_bucket('30 days', time);

-- ============================================================
-- TIME-SERIES: Action Tracking (for prescriptive feedback loop)
-- ============================================================

CREATE TABLE ts_action_events (
    time            TIMESTAMPTZ NOT NULL,
    action_id       UUID NOT NULL,
    employee_id     UUID NOT NULL,
    event_type      TEXT NOT NULL,       -- 'recommended', 'approved', 'started', 'completed', 'declined'
    action_type     TEXT NOT NULL,       -- 'stay_interview', 'comp_adjustment', etc.
    assigned_to     UUID,
    prediction_score_at_event NUMERIC(5,4),
    outcome_type    TEXT,                -- null until completed
    outcome_metrics JSONB,
    notes           TEXT
);

SELECT create_hypertable('ts_action_events', 'time');
CREATE INDEX idx_ts_actions_emp ON ts_action_events(employee_id, time DESC);
CREATE INDEX idx_ts_actions_id ON ts_action_events(action_id, time DESC);

-- ============================================================
-- COMPRESSION & RETENTION POLICIES
-- ============================================================

-- Compress data older than 90 days (10-20x storage reduction)
SELECT add_compression_policy('ts_engagement_scores', INTERVAL '90 days');
SELECT add_compression_policy('ts_collaboration', INTERVAL '90 days');
SELECT add_compression_policy('ts_compensation', INTERVAL '90 days');
SELECT add_compression_policy('ts_performance', INTERVAL '90 days');
SELECT add_compression_policy('ts_prediction_scores', INTERVAL '90 days');
SELECT add_compression_policy('ts_action_events', INTERVAL '90 days');

-- Retain detailed data for 5 years, then drop (configurable per org)
SELECT add_retention_policy('ts_collaboration', INTERVAL '5 years');
SELECT add_retention_policy('ts_engagement_scores', INTERVAL '5 years');
```

### Property Graph (Apache AGE)

```sql
-- ============================================================
-- GRAPH: Organisational & Collaboration Network
-- ============================================================

-- Create the graph
SELECT * FROM ag_catalog.create_graph('hr_graph');

-- ── Node types ─────────────────────────────────────────────

-- Employee nodes (synced from relational employees table)
-- Properties: employee_id, name, department, position, location, hire_date, status

-- Department nodes
-- Properties: department_id, name, cost_centre

-- Position nodes
-- Properties: position_id, title, level, band, is_critical

-- Competency nodes
-- Properties: competency_id, name, category

-- ── Edge types ─────────────────────────────────────────────

-- REPORTS_TO: Employee -> Employee (manager)
--   Properties: since, is_current

-- BELONGS_TO: Employee -> Department
--   Properties: since, is_current

-- HOLDS: Employee -> Position
--   Properties: since, is_current

-- COLLABORATES_WITH: Employee -> Employee
--   Properties: interaction_count, avg_response_time, last_interaction, source, weight

-- HAS_COMPETENCY: Employee -> Competency
--   Properties: assessed_level, assessed_at, source

-- REQUIRES_COMPETENCY: Position -> Competency
--   Properties: required_level

-- SUCCESSOR_FOR: Employee -> Position
--   Properties: readiness, readiness_score, bench_rank

-- MENTORS: Employee -> Employee
--   Properties: since, focus_area
```

```cypher
-- ── Example: Create employee node and reporting relationship ──

SELECT * FROM cypher('hr_graph', $$
  CREATE (e:Employee {
    employee_id: 'abc-123',
    name: 'Jane Smith',
    department: 'Engineering',
    position: 'Senior Engineer',
    hire_date: '2022-03-15',
    status: 'active'
  })
  RETURN e
$$) AS (v agtype);

SELECT * FROM cypher('hr_graph', $$
  MATCH (e:Employee {employee_id: 'abc-123'}),
        (m:Employee {employee_id: 'mgr-456'})
  CREATE (e)-[:REPORTS_TO {since: '2022-03-15', is_current: true}]->(m)
$$) AS (e agtype);

-- ── Example: Find all reports N levels deep (flight risk propagation) ──

SELECT * FROM cypher('hr_graph', $$
  MATCH (m:Employee {employee_id: 'mgr-456'})<-[:REPORTS_TO*1..5]-(report)
  WHERE report.status = 'active'
  RETURN report.employee_id, report.name, length(path) AS depth
$$) AS (employee_id agtype, name agtype, depth agtype);

-- ── Example: Collaboration network centrality ──

SELECT * FROM cypher('hr_graph', $$
  MATCH (e:Employee)-[c:COLLABORATES_WITH]-(other:Employee)
  WHERE e.department = 'Engineering'
  RETURN e.employee_id, e.name,
         count(DISTINCT other) AS unique_collaborators,
         sum(c.interaction_count) AS total_interactions
  ORDER BY unique_collaborators DESC
$$) AS (id agtype, name agtype, collaborators agtype, interactions agtype);

-- ── Example: Succession path -- find candidates with smallest competency gap ──

SELECT * FROM cypher('hr_graph', $$
  MATCH (p:Position {is_critical: true})-[:REQUIRES_COMPETENCY]->(c:Competency)
  WITH p, collect({competency: c.name, required: c.required_level}) AS requirements
  MATCH (e:Employee)-[h:HAS_COMPETENCY]->(c2:Competency)
  WHERE c2.name IN [r IN requirements | r.competency]
  WITH p, e, requirements,
       collect({competency: c2.name, level: h.assessed_level}) AS current_levels
  RETURN p.title, e.name,
         size([r IN requirements WHERE NOT r.competency IN
              [cl IN current_levels | cl.competency]]) AS missing_competencies
  ORDER BY missing_competencies ASC
  LIMIT 10
$$) AS (position agtype, candidate agtype, gaps agtype);
```

## ML Feature Extraction Queries (Time-Series)

```sql
-- Build a flight-risk feature vector using time-series functions
SELECT
    e.id AS employee_id,
    -- Tenure
    EXTRACT(EPOCH FROM (now() - e.hire_date)) / (86400 * 30) AS tenure_months,
    -- Compensation trend
    last(tc.comp_to_market_ratio, tc.time) AS current_comp_ratio,
    last(tc.comp_to_market_ratio, tc.time) - first(tc.comp_to_market_ratio, tc.time)
        AS comp_ratio_change_2yr,
    -- Engagement trend
    last(te.overall_score, te.time) AS current_engagement,
    last(te.overall_score, te.time) - first(te.overall_score, te.time)
        AS engagement_change_2yr,
    -- Collaboration signals
    avg(tcl.meeting_count) AS avg_meetings_12m,
    avg(tcl.cross_team_interactions) AS avg_cross_team_12m,
    avg(tcl.network_centrality) AS avg_centrality_12m,
    -- Performance
    last(tp.overall_rating, tp.time) AS latest_rating,
    avg(tp.overall_rating) AS avg_rating_3yr
FROM employees e
LEFT JOIN ts_compensation tc ON tc.employee_id = e.id
    AND tc.time > now() - INTERVAL '2 years'
LEFT JOIN ts_engagement_scores te ON te.employee_id = e.id
    AND te.time > now() - INTERVAL '2 years'
LEFT JOIN ts_collaboration tcl ON tcl.employee_id = e.id
    AND tcl.time > now() - INTERVAL '1 year'
LEFT JOIN ts_performance tp ON tp.employee_id = e.id
    AND tp.time > now() - INTERVAL '3 years'
WHERE e.status = 'active'
GROUP BY e.id, e.hire_date;
```

## Scalability Considerations

- **TimescaleDB compression**: Achieves 10-20x compression on numeric time-series data.
  A 1,000-employee organisation generating weekly collaboration snapshots produces ~52K
  rows/year; compressed, this requires minimal storage even over 5+ years.
- **Continuous aggregates**: Pre-computed rollups (weekly, monthly, quarterly) eliminate
  expensive real-time aggregation queries on dashboards.
- **Graph partitioning**: For large enterprises (50,000+ employees), the collaboration
  graph can be partitioned by department or region. Apache AGE stores graph data in
  PostgreSQL tables, so standard partitioning strategies apply.
- **Horizontal scaling**: TimescaleDB supports multi-node distributed hypertables for
  very large deployments. The graph layer can be moved to a dedicated Neo4j or Neptune
  instance if AGE performance becomes a bottleneck.

## Migration Path

This architecture can be adopted incrementally from the normalized model (Suggestion 1):

1. **Phase 1**: Install TimescaleDB, convert time-series-heavy tables (collaboration
   snapshots, prediction scores) to hypertables. No application changes needed -- 
   hypertables are query-compatible with regular PostgreSQL tables.
2. **Phase 2**: Add compression and continuous aggregation policies. Update dashboards
   to query continuous aggregates for performance.
3. **Phase 3**: Install Apache AGE, build the organisational graph from existing relational
   data (employees, departments, manager relationships). Run graph queries for succession
   planning and collaboration analysis alongside existing SQL.
4. **Phase 4**: Add collaboration edges from metadata ingestion. Implement graph-based
   features (centrality, clustering) in the ML pipeline.

Fallback: if Apache AGE proves insufficient for complex graph analytics, export the
graph to Neo4j for batch computation while keeping AGE for real-time traversals, or
replace AGE entirely with a dedicated graph database connected via foreign data wrappers.
