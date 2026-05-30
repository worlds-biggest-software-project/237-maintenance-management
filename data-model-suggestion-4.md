# Data Model Suggestion 4: Relational + Time-Series Hybrid (IoT/Predictive Maintenance First)

> Project: CMMS (Maintenance Management) · Created: 2026-05-22

## Philosophy

This model is designed for organisations where IoT sensor data and predictive maintenance are the primary value proposition, not just an add-on. It uses a standard relational schema for CRUD-heavy operational data (work orders, assets, users, parts) and a dedicated time-series layer (TimescaleDB hypertables on PostgreSQL) for high-volume sensor telemetry, equipment metrics, and AI/ML prediction outputs. The two layers share a single PostgreSQL instance, eliminating the need for a separate time-series database like InfluxDB while retaining the performance characteristics of a purpose-built TSDB.

Modern industrial facilities generate 1-10 TB of sensor data per day. Storing vibration readings, temperature logs, pressure measurements, and power consumption data in a regular relational table would overwhelm conventional indexing and query patterns within weeks. TimescaleDB hypertables automatically partition data by time, enable compression of historical data (10-20x reduction), and provide continuous aggregates that pre-compute rollups (hourly averages, daily maxima) without impacting write throughput.

The architecture is explicitly designed to support the AI-native capabilities outlined in the project README: anomaly detection on sensor streams, AI-triggered work-order generation, condition-based PM scheduling, and failure prediction. The time-series layer provides the data foundation that ML models consume, while the relational layer manages the operational workflows those predictions trigger. This separation allows the sensor ingestion pipeline to scale independently of the CMMS application.

**Best for:** Organisations investing heavily in condition-based and predictive maintenance, IIoT sensor networks, SCADA/OPC UA integration, and AI-driven anomaly detection -- manufacturing plants, utilities, Oil & Gas, and data centres.

**Trade-offs:**
- (+) Native support for high-volume sensor data (millions of readings per day per device)
- (+) Automatic time-based partitioning, compression, and retention policies
- (+) Continuous aggregates provide instant dashboard KPIs without full-table scans
- (+) ML-ready data: sensor streams, feature stores, and prediction logs in query-optimised format
- (+) Single PostgreSQL instance (TimescaleDB is a PG extension) -- no separate TSDB to manage
- (-) TimescaleDB extension required; not available on all managed PostgreSQL providers
- (-) Time-series hypertables have restrictions (no unique constraints across chunks, limited UPDATE)
- (-) More complex operational model: retention policies, compression jobs, and continuous aggregates need configuration
- (-) Sensor data schema must be designed upfront; changing the measurement model is harder
- (-) Higher storage and compute costs for the time-series layer

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 55000/55001 | Asset register and lifecycle management in the relational layer; condition data driving asset health scoring |
| ISO 13306 | Work order types include `condition_based` and `predictive` as first-class categories alongside corrective and preventive |
| ISO 14224 | Equipment taxonomy structures the asset hierarchy; failure mode coding on work orders follows ISO 14224 |
| OPC UA (IEC 62541) | Sensor device registry includes OPC UA node IDs; the schema models OPC UA information model concepts (variable nodes, data items) |
| MQTT v5.0 / Sparkplug B | Sensor ingestion pipeline modeled on MQTT topic hierarchies; device-to-hypertable mapping follows Sparkplug B namespace conventions |
| MIMOSA CRIS | Condition monitoring data structure (measurement point, measurement location, measurement value) aligns to MIMOSA CRIS condition monitoring entities |
| ISO 17359 | Condition monitoring and diagnostics of machines -- guidelines for measurement point selection inform the sensor point configuration |
| NIST AI RMF | AI model governance tables (model registry, prediction log, model performance metrics) follow NIST AI Risk Management Framework principles |

---

## Relational Layer -- Core CMMS Tables

```sql
-- ============================================================
-- RELATIONAL LAYER: Standard CMMS operational tables
-- (Abbreviated -- see Data Model Suggestion 1 for full detail
--  on user, role, vendor, purchase order tables)
-- ============================================================

CREATE EXTENSION IF NOT EXISTS timescaledb;

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    subscription    VARCHAR(50) NOT NULL DEFAULT 'standard',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE site (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50),
    country_code    CHAR(2),
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

CREATE TABLE location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    site_id         UUID NOT NULL REFERENCES site(id),
    parent_id       UUID REFERENCES location(id),
    code            VARCHAR(100) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    location_type   VARCHAR(50) NOT NULL,
    path            TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, site_id, code)
);

CREATE TABLE asset (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    site_id         UUID NOT NULL REFERENCES site(id),
    location_id     UUID REFERENCES location(id),
    parent_asset_id UUID REFERENCES asset(id),
    asset_tag       VARCHAR(100) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    equipment_class VARCHAR(50),
    serial_number   VARCHAR(100),
    model           VARCHAR(255),
    manufacturer    VARCHAR(255),
    status          VARCHAR(30) NOT NULL DEFAULT 'active',
    criticality     VARCHAR(20) NOT NULL DEFAULT 'medium',
    install_date    DATE,
    warranty_expiry DATE,
    has_meter       BOOLEAN NOT NULL DEFAULT false,
    meter_unit      VARCHAR(50),
    current_meter   NUMERIC(14, 2),
    specifications  JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, asset_tag)
);

CREATE INDEX idx_asset_tenant_site ON asset(tenant_id, site_id);
CREATE INDEX idx_asset_status ON asset(status);

CREATE TABLE work_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    site_id         UUID NOT NULL REFERENCES site(id),
    wo_number       VARCHAR(50) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    wo_type         VARCHAR(30) NOT NULL,
    priority        VARCHAR(20) NOT NULL DEFAULT 'medium',
    status          VARCHAR(30) NOT NULL DEFAULT 'requested',
    asset_id        UUID REFERENCES asset(id),
    location_id     UUID REFERENCES location(id),
    assigned_to     UUID REFERENCES app_user(id),
    failure_mode    VARCHAR(50),
    failure_cause   VARCHAR(50),
    scheduled_start TIMESTAMPTZ,
    scheduled_end   TIMESTAMPTZ,
    actual_start    TIMESTAMPTZ,
    actual_end      TIMESTAMPTZ,
    due_date        TIMESTAMPTZ,
    estimated_hours NUMERIC(8, 2),
    actual_hours    NUMERIC(8, 2),
    downtime_minutes INTEGER,

    -- Predictive maintenance linkage
    triggered_by_alert_id UUID,  -- FK to sensor_alert, added below
    ai_confidence_score NUMERIC(5, 4),  -- ML model confidence that triggered this WO

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    UNIQUE(tenant_id, wo_number)
);

CREATE INDEX idx_wo_tenant_status ON work_order(tenant_id, status);
CREATE INDEX idx_wo_asset ON work_order(asset_id);
CREATE INDEX idx_wo_due ON work_order(due_date);

CREATE TABLE pm_schedule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    asset_id        UUID NOT NULL REFERENCES asset(id),
    name            VARCHAR(255) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    trigger_type    VARCHAR(30) NOT NULL,
    frequency_value INTEGER,
    frequency_unit  VARCHAR(20),
    meter_interval  NUMERIC(14, 2),
    next_due_date   DATE,
    last_completed  TIMESTAMPTZ,

    -- Condition-based trigger: reference a sensor measurement point
    condition_sensor_id UUID,      -- FK to measurement_point
    condition_threshold NUMERIC(18, 6),
    condition_operator VARCHAR(10),  -- gt, gte, lt, lte, eq

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Time-Series Layer -- Sensor & IoT Data

```sql
-- ============================================================
-- TIME-SERIES LAYER: IoT sensor data (TimescaleDB hypertables)
-- ============================================================

-- ============================================================
-- MEASUREMENT POINT REGISTRY
-- A measurement point is a specific sensor channel on a specific asset.
-- E.g., "Pump P-101A, Drive End, Vibration (Overall)"
-- Aligned to MIMOSA CRIS measurement location / ISO 17359 concepts.
-- ============================================================

CREATE TABLE measurement_point (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    asset_id        UUID NOT NULL REFERENCES asset(id),
    site_id         UUID NOT NULL REFERENCES site(id),

    -- Identity
    point_name      VARCHAR(255) NOT NULL,
    point_code      VARCHAR(100) NOT NULL,   -- machine-readable tag: P101A.DE.VIB.OA
    description     TEXT,

    -- Measurement type
    measurement_type VARCHAR(50) NOT NULL,
    -- vibration_velocity, vibration_acceleration, vibration_displacement,
    -- temperature, pressure, flow_rate, level, current, voltage, power,
    -- speed_rpm, humidity, ph, conductivity, oil_quality
    unit            VARCHAR(50) NOT NULL,    -- mm/s, g, um, celsius, psi, m3/h, bar, amps, kW, rpm

    -- Protocol configuration
    protocol        VARCHAR(50),             -- mqtt, opcua, modbus, http, manual
    source_config   JSONB NOT NULL DEFAULT '{}',
    -- MQTT:    { "broker": "mqtt://...", "topic": "site1/pump/P101A/vibration/de" }
    -- OPC UA:  { "endpoint": "opc.tcp://...", "node_id": "ns=2;s=P101A.DE.VIB" }
    -- Modbus:  { "host": "192.168.1.100", "register": 40001, "type": "float32" }

    -- Location on the machine (ISO 17359)
    measurement_location VARCHAR(100),       -- drive_end, non_drive_end, casing, inlet, outlet
    orientation     VARCHAR(20),             -- horizontal, vertical, axial

    -- Thresholds (ISO 10816 / machine-specific)
    warning_low     NUMERIC(18, 6),
    alarm_low       NUMERIC(18, 6),
    warning_high    NUMERIC(18, 6),
    alarm_high      NUMERIC(18, 6),

    -- Sampling
    sample_interval_seconds INTEGER,          -- expected interval between readings
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_reading_at TIMESTAMPTZ,
    last_reading_value NUMERIC(18, 6),

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, point_code)
);

CREATE INDEX idx_mpoint_asset ON measurement_point(asset_id);
CREATE INDEX idx_mpoint_tenant_site ON measurement_point(tenant_id, site_id);
CREATE INDEX idx_mpoint_type ON measurement_point(measurement_type);

-- ============================================================
-- SENSOR READINGS (TimescaleDB hypertable)
-- This is the high-volume table: millions of rows per day.
-- ============================================================

CREATE TABLE sensor_reading (
    time            TIMESTAMPTZ NOT NULL,
    point_id        UUID NOT NULL,           -- references measurement_point(id)
    value           DOUBLE PRECISION NOT NULL,
    quality         SMALLINT NOT NULL DEFAULT 0,  -- 0=good, 1=uncertain, 2=bad, 3=missing
    raw_value       DOUBLE PRECISION,        -- before scaling/conversion
    tenant_id       UUID NOT NULL             -- for RLS and partitioning
);

-- Convert to TimescaleDB hypertable (auto-partitions by time)
SELECT create_hypertable('sensor_reading', 'time',
    chunk_time_interval => INTERVAL '1 day',
    if_not_exists => TRUE
);

-- Composite index for point-level time-range queries
CREATE INDEX idx_reading_point_time ON sensor_reading(point_id, time DESC);

-- Tenant-scoped index
CREATE INDEX idx_reading_tenant_time ON sensor_reading(tenant_id, time DESC);

-- Enable compression on chunks older than 7 days (10-20x storage reduction)
ALTER TABLE sensor_reading SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'point_id, tenant_id',
    timescaledb.compress_orderby = 'time DESC'
);

SELECT add_compression_policy('sensor_reading', INTERVAL '7 days');

-- Retention policy: drop raw data older than 2 years (keep aggregates longer)
SELECT add_retention_policy('sensor_reading', INTERVAL '2 years');

-- ============================================================
-- CONTINUOUS AGGREGATES (pre-computed rollups)
-- ============================================================

-- Hourly aggregates for dashboards and trend analysis
CREATE MATERIALIZED VIEW sensor_reading_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS bucket,
    point_id,
    tenant_id,
    AVG(value)   AS avg_value,
    MIN(value)   AS min_value,
    MAX(value)   AS max_value,
    STDDEV(value) AS stddev_value,
    COUNT(*)     AS sample_count,
    -- Peak detection
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY value) AS p95_value,
    PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY value) AS p99_value
FROM sensor_reading
GROUP BY bucket, point_id, tenant_id
WITH NO DATA;

-- Refresh hourly aggregates every 30 minutes, covering the last 2 hours
SELECT add_continuous_aggregate_policy('sensor_reading_hourly',
    start_offset => INTERVAL '2 hours',
    end_offset => INTERVAL '30 minutes',
    schedule_interval => INTERVAL '30 minutes'
);

-- Daily aggregates for long-term trending and reporting
CREATE MATERIALIZED VIEW sensor_reading_daily
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time) AS bucket,
    point_id,
    tenant_id,
    AVG(value)   AS avg_value,
    MIN(value)   AS min_value,
    MAX(value)   AS max_value,
    STDDEV(value) AS stddev_value,
    COUNT(*)     AS sample_count,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY value) AS p95_value
FROM sensor_reading
GROUP BY bucket, point_id, tenant_id
WITH NO DATA;

SELECT add_continuous_aggregate_policy('sensor_reading_daily',
    start_offset => INTERVAL '3 days',
    end_offset => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 day'
);

-- Keep daily aggregates for 10 years (even after raw data is dropped)
-- No retention policy on daily aggregates -- they are small and valuable
```

## Sensor Alerts & AI-Triggered Work Orders

```sql
-- ============================================================
-- SENSOR ALERTS (threshold breaches and anomaly detections)
-- ============================================================

CREATE TABLE sensor_alert (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    point_id        UUID NOT NULL,           -- references measurement_point
    asset_id        UUID NOT NULL REFERENCES asset(id),
    site_id         UUID NOT NULL REFERENCES site(id),

    alert_time      TIMESTAMPTZ NOT NULL,
    alert_type      VARCHAR(30) NOT NULL,
    -- threshold_warning, threshold_alarm, anomaly_detected,
    -- trend_deviation, rate_of_change, sensor_offline
    severity        VARCHAR(20) NOT NULL,    -- warning, alarm, critical

    -- What triggered it
    trigger_value   DOUBLE PRECISION,
    trigger_threshold DOUBLE PRECISION,
    trigger_unit    VARCHAR(50),
    trigger_details JSONB,
    -- Anomaly: { "model_id": "uuid", "anomaly_score": 0.95,
    --            "expected_range": [2.1, 4.8], "actual": 12.3 }
    -- Trend:   { "trend_direction": "increasing", "rate_per_hour": 0.5,
    --            "projected_alarm_in_hours": 48 }

    -- Resolution
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    -- active, acknowledged, investigating, resolved, false_positive
    acknowledged_by UUID REFERENCES app_user(id),
    acknowledged_at TIMESTAMPTZ,
    resolved_at     TIMESTAMPTZ,

    -- Work order linkage
    auto_wo_created BOOLEAN NOT NULL DEFAULT false,
    work_order_id   UUID REFERENCES work_order(id),

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alert_tenant_time ON sensor_alert(tenant_id, alert_time DESC);
CREATE INDEX idx_alert_asset ON sensor_alert(asset_id, alert_time DESC);
CREATE INDEX idx_alert_status ON sensor_alert(status) WHERE status = 'active';
CREATE INDEX idx_alert_point ON sensor_alert(point_id, alert_time DESC);

-- Add FK from work_order to sensor_alert
ALTER TABLE work_order
    ADD CONSTRAINT fk_wo_alert FOREIGN KEY (triggered_by_alert_id) REFERENCES sensor_alert(id);
```

## AI/ML Model Registry & Predictions

```sql
-- ============================================================
-- AI/ML MODEL REGISTRY (NIST AI RMF aligned)
-- ============================================================

CREATE TABLE ml_model (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    model_name      VARCHAR(255) NOT NULL,
    model_type      VARCHAR(50) NOT NULL,
    -- anomaly_detection, failure_prediction, remaining_useful_life,
    -- parts_demand_forecast, pm_interval_optimisation, root_cause_analysis
    description     TEXT,
    version         VARCHAR(50) NOT NULL,
    framework       VARCHAR(50),             -- pytorch, sklearn, xgboost, prophet
    target_equipment_class VARCHAR(50),       -- which equipment class this model serves

    -- Model configuration
    input_features  JSONB NOT NULL,
    -- ["vibration_velocity_de_h", "vibration_velocity_de_v", "temperature_bearing_de",
    --  "current_motor", "speed_rpm", "runtime_hours"]
    output_schema   JSONB NOT NULL,
    -- { "prediction": "float", "confidence": "float", "risk_level": "string",
    --   "remaining_useful_life_hours": "float" }

    -- Performance metrics
    metrics         JSONB,
    -- { "accuracy": 0.94, "precision": 0.91, "recall": 0.96, "f1": 0.935,
    --   "false_positive_rate": 0.03, "test_set_size": 2500 }

    -- Lifecycle
    status          VARCHAR(20) NOT NULL DEFAULT 'staging',
    -- staging, active, deprecated, retired
    trained_at      TIMESTAMPTZ,
    deployed_at     TIMESTAMPTZ,
    model_artifact_path TEXT,                -- S3 path to model weights/artifacts

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ml_model_tenant ON ml_model(tenant_id);
CREATE INDEX idx_ml_model_type ON ml_model(model_type);
CREATE INDEX idx_ml_model_status ON ml_model(status);

-- ============================================================
-- AI PREDICTION LOG (time-series: what the model predicted, when)
-- ============================================================

CREATE TABLE ai_prediction (
    time            TIMESTAMPTZ NOT NULL,
    tenant_id       UUID NOT NULL,
    model_id        UUID NOT NULL,           -- references ml_model
    asset_id        UUID NOT NULL,
    point_id        UUID,                    -- primary measurement point used

    -- Prediction output
    prediction_type VARCHAR(50) NOT NULL,
    -- anomaly_score, failure_probability, remaining_useful_life, health_score
    prediction_value DOUBLE PRECISION NOT NULL,
    confidence      DOUBLE PRECISION,
    risk_level      VARCHAR(20),             -- low, medium, high, critical

    -- Context
    input_features  JSONB,                   -- snapshot of feature values used
    explanation     JSONB,                   -- SHAP values or feature importance
    -- { "top_features": [
    --     { "feature": "vibration_de_h", "importance": 0.42, "value": 11.2 },
    --     { "feature": "temperature_bearing", "importance": 0.28, "value": 85.3 }
    --   ], "anomaly_contributors": ["vibration_de_h"] }

    -- Action taken
    alert_generated BOOLEAN NOT NULL DEFAULT false,
    alert_id        UUID                     -- references sensor_alert if created
);

-- Convert to hypertable for time-series storage
SELECT create_hypertable('ai_prediction', 'time',
    chunk_time_interval => INTERVAL '1 day',
    if_not_exists => TRUE
);

CREATE INDEX idx_prediction_asset ON ai_prediction(asset_id, time DESC);
CREATE INDEX idx_prediction_model ON ai_prediction(model_id, time DESC);
CREATE INDEX idx_prediction_risk ON ai_prediction(risk_level, time DESC)
    WHERE risk_level IN ('high', 'critical');

-- Compress predictions older than 30 days
ALTER TABLE ai_prediction SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'asset_id, model_id, tenant_id',
    timescaledb.compress_orderby = 'time DESC'
);

SELECT add_compression_policy('ai_prediction', INTERVAL '30 days');

-- ============================================================
-- AI FEATURE STORE (pre-computed features for ML inference)
-- ============================================================

CREATE TABLE ai_feature_store (
    time            TIMESTAMPTZ NOT NULL,
    tenant_id       UUID NOT NULL,
    asset_id        UUID NOT NULL,
    feature_set     VARCHAR(100) NOT NULL,   -- e.g., 'pump_health_v2'

    features        JSONB NOT NULL,
    -- {
    --   "vibration_de_h_avg_1h": 4.2,
    --   "vibration_de_h_max_1h": 7.8,
    --   "vibration_de_h_stddev_1h": 1.1,
    --   "vibration_de_h_trend_24h": 0.15,
    --   "temperature_bearing_de_avg_1h": 72.5,
    --   "runtime_hours_since_last_pm": 1240,
    --   "days_since_last_failure": 365,
    --   "failure_count_12m": 2,
    --   "pm_compliance_rate_6m": 0.92
    -- }

    computed_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

SELECT create_hypertable('ai_feature_store', 'time',
    chunk_time_interval => INTERVAL '1 day',
    if_not_exists => TRUE
);

CREATE INDEX idx_feature_asset ON ai_feature_store(asset_id, feature_set, time DESC);

ALTER TABLE ai_feature_store SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'asset_id, feature_set, tenant_id',
    timescaledb.compress_orderby = 'time DESC'
);

SELECT add_compression_policy('ai_feature_store', INTERVAL '30 days');
```

## Equipment Health Score

```sql
-- ============================================================
-- ASSET HEALTH SCORE (aggregated from sensor data + maintenance history)
-- ============================================================

CREATE TABLE asset_health_score (
    time            TIMESTAMPTZ NOT NULL,
    tenant_id       UUID NOT NULL,
    asset_id        UUID NOT NULL REFERENCES asset(id),
    site_id         UUID NOT NULL,

    -- Overall health (0-100, where 100 = perfect condition)
    health_score    NUMERIC(5, 2) NOT NULL,
    health_status   VARCHAR(20) NOT NULL,    -- excellent, good, fair, poor, critical
    trend           VARCHAR(20),             -- improving, stable, degrading

    -- Component scores
    component_scores JSONB,
    -- {
    --   "vibration": { "score": 72, "status": "fair", "worst_point": "DE Horizontal" },
    --   "temperature": { "score": 95, "status": "excellent" },
    --   "electrical": { "score": 88, "status": "good" },
    --   "maintenance_compliance": { "score": 92, "status": "good" }
    -- }

    -- Predictions
    estimated_rul_hours NUMERIC(10, 2),      -- Remaining Useful Life
    failure_probability_30d NUMERIC(5, 4),   -- probability of failure in next 30 days
    recommended_action VARCHAR(100),
    -- "schedule_inspection", "order_replacement_parts", "immediate_attention", "no_action"

    model_id        UUID                     -- which ML model computed this
);

SELECT create_hypertable('asset_health_score', 'time',
    chunk_time_interval => INTERVAL '1 day',
    if_not_exists => TRUE
);

CREATE INDEX idx_health_asset ON asset_health_score(asset_id, time DESC);
CREATE INDEX idx_health_status ON asset_health_score(health_status, time DESC)
    WHERE health_status IN ('poor', 'critical');

-- Daily health score continuous aggregate for trending
CREATE MATERIALIZED VIEW asset_health_daily
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time) AS bucket,
    asset_id,
    tenant_id,
    site_id,
    AVG(health_score) AS avg_health_score,
    MIN(health_score) AS min_health_score,
    MAX(failure_probability_30d) AS max_failure_prob
FROM asset_health_score
GROUP BY bucket, asset_id, tenant_id, site_id
WITH NO DATA;

SELECT add_continuous_aggregate_policy('asset_health_daily',
    start_offset => INTERVAL '3 days',
    end_offset => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 day'
);
```

## Meter Readings as Time Series

```sql
-- ============================================================
-- METER READINGS (runtime hours, odometer, cycles -- also time-series)
-- ============================================================

CREATE TABLE meter_reading (
    time            TIMESTAMPTZ NOT NULL,
    tenant_id       UUID NOT NULL,
    asset_id        UUID NOT NULL,
    meter_type      VARCHAR(50) NOT NULL,    -- runtime_hours, odometer_km, cycles, litres
    reading_value   DOUBLE PRECISION NOT NULL,
    delta_value     DOUBLE PRECISION,        -- change since last reading
    source          VARCHAR(30) DEFAULT 'manual',  -- manual, plc, scada, iot
    recorded_by     UUID                     -- user ID if manual
);

SELECT create_hypertable('meter_reading', 'time',
    chunk_time_interval => INTERVAL '7 days',
    if_not_exists => TRUE
);

CREATE INDEX idx_meter_asset ON meter_reading(asset_id, meter_type, time DESC);
```

## Audit Trail

```sql
-- ============================================================
-- AUDIT & COMPLIANCE (standard relational)
-- ============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    action          VARCHAR(20) NOT NULL,
    changes         JSONB,
    user_id         UUID,
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_tenant_time ON audit_log(tenant_id, created_at DESC);

CREATE TABLE electronic_signature (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    signer_id       UUID NOT NULL REFERENCES app_user(id),
    signature_meaning VARCHAR(100) NOT NULL,
    signed_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    ip_address      INET
);

CREATE INDEX idx_esig_entity ON electronic_signature(entity_type, entity_id);
```

## Example: Predictive Maintenance Query

```sql
-- "Show me assets with degrading health scores that have a >30% failure
--  probability in the next 30 days, ranked by risk"

SELECT
    a.asset_tag,
    a.name,
    a.equipment_class,
    s.name AS site_name,
    hs.health_score,
    hs.health_status,
    hs.trend,
    hs.failure_probability_30d,
    hs.estimated_rul_hours,
    hs.recommended_action,
    hs.component_scores
FROM asset_health_score hs
JOIN asset a ON a.id = hs.asset_id
JOIN site s ON s.id = a.site_id
WHERE hs.tenant_id = $1
  AND hs.time = (SELECT MAX(time) FROM asset_health_score WHERE asset_id = hs.asset_id)
  AND hs.failure_probability_30d > 0.30
  AND hs.trend = 'degrading'
ORDER BY hs.failure_probability_30d DESC;
```

## Example: Sensor Trend with Aggregates

```sql
-- "Show me hourly vibration trends for pump P-101A drive end over the last 7 days"

SELECT
    bucket AS time,
    avg_value,
    min_value,
    max_value,
    p95_value,
    sample_count
FROM sensor_reading_hourly srh
JOIN measurement_point mp ON mp.id = srh.point_id
WHERE mp.point_code = 'P101A.DE.VIB.OA'
  AND srh.tenant_id = $1
  AND bucket >= now() - INTERVAL '7 days'
ORDER BY bucket;
```

## Example: Feature Engineering for ML

```sql
-- Build features for the pump health ML model from continuous aggregates

SELECT
    srh.bucket,
    mp.asset_id,
    mp.point_code,
    srh.avg_value,
    srh.max_value,
    srh.stddev_value,
    srh.p95_value,
    -- Compute rate of change (trend)
    srh.avg_value - LAG(srh.avg_value, 24) OVER (
        PARTITION BY mp.id ORDER BY srh.bucket
    ) AS change_24h,
    -- Rolling statistics
    AVG(srh.avg_value) OVER (
        PARTITION BY mp.id ORDER BY srh.bucket
        ROWS BETWEEN 167 PRECEDING AND CURRENT ROW  -- 7-day rolling avg
    ) AS rolling_7d_avg
FROM sensor_reading_hourly srh
JOIN measurement_point mp ON mp.id = srh.point_id
WHERE mp.asset_id = $1
  AND mp.measurement_type = 'vibration_velocity'
  AND srh.bucket >= now() - INTERVAL '30 days'
ORDER BY srh.bucket;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Relational: Core CMMS | 8 | tenant, site, app_user, location, asset, work_order, pm_schedule, + user/role tables |
| Time-Series: Sensor Data | 1 hypertable + 2 continuous aggregates | sensor_reading, sensor_reading_hourly, sensor_reading_daily |
| Time-Series: Meter Data | 1 hypertable | meter_reading |
| Sensor Registry | 1 | measurement_point |
| Alerts | 1 | sensor_alert |
| AI/ML | 4 (2 hypertables) | ml_model, ai_prediction, ai_feature_store, asset_health_score |
| Continuous Aggregates | 1 | asset_health_daily |
| Compliance | 2 | audit_log, electronic_signature |
| **Total** | **~19 tables + 3 continuous aggregates** | Plus parts, vendor, PO tables from Model 1 |

---

## Key Design Decisions

1. **TimescaleDB on PostgreSQL, not a separate TSDB.** Using TimescaleDB as a PostgreSQL extension means sensor data and relational CMMS data share a single database. JOINs between sensor readings and assets are native SQL -- no cross-database ETL. This eliminates the operational complexity of running InfluxDB alongside PostgreSQL.

2. **Measurement point as the core IoT entity.** Rather than a generic "sensor device" table, the `measurement_point` table models a specific measurement on a specific location on a specific asset (e.g., "P-101A, Drive End, Horizontal Vibration"). This aligns with ISO 17359 and MIMOSA CRIS condition monitoring concepts and is how reliability engineers actually think about sensor data.

3. **Continuous aggregates for dashboard performance.** Hourly and daily aggregates are computed incrementally by TimescaleDB in the background. Dashboard queries for "show me the trend" hit pre-computed materialized views, not raw data. Raw data scans are reserved for detailed investigation.

4. **Compression and retention policies.** Raw sensor data is compressed after 7 days (10-20x reduction) and dropped after 2 years. Hourly aggregates are retained longer. Daily aggregates are kept indefinitely. This tiered retention ensures storage costs are manageable while preserving long-term trend data.

5. **AI model registry with NIST AI RMF alignment.** The `ml_model` table tracks model lifecycle (staging, active, deprecated, retired), performance metrics, and input/output schemas. This supports model governance and auditability as recommended by the NIST AI Risk Management Framework.

6. **AI prediction log as a hypertable.** Prediction outputs are time-series data -- every inference run produces a timestamped prediction. Storing predictions as a hypertable enables temporal analysis of model performance (e.g., "how accurate were predictions for this asset class over the last 6 months?") and supports model drift detection.

7. **Feature store for ML inference.** The `ai_feature_store` table pre-computes and caches feature vectors that ML models consume. This decouples feature engineering from model inference, enables feature reuse across models, and provides a record of exactly what inputs produced each prediction.

8. **Sensor alerts bridge time-series and relational worlds.** The `sensor_alert` table lives in the relational layer (not a hypertable) because alerts have lifecycle state (acknowledged, resolved) and link to work orders. This is where the time-series detection layer connects to the operational CMMS workflow.

9. **Asset health score as an aggregated time series.** The `asset_health_score` hypertable combines sensor data, maintenance history, and ML predictions into a single health metric per asset. This is the primary surface that maintenance managers interact with -- a simple 0-100 score with trend direction and recommended action.

10. **Work order tracks AI provenance.** The `triggered_by_alert_id` and `ai_confidence_score` columns on `work_order` record when a work order was generated by AI and how confident the model was. This enables analysis of AI-triggered vs. manually-created work orders and tracking of false-positive rates.
