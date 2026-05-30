# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: CMMS (Maintenance Management) · Created: 2026-05-22

## Philosophy

This model uses Event Sourcing as the foundational persistence strategy. Instead of storing the current state of a work order, asset, or inventory record in mutable rows, every state change is captured as an immutable event in an append-only event store. The current state of any entity is derived by replaying its event stream from the beginning, or (for performance) by reading a materialised projection that is rebuilt from the event stream.

The Command Query Responsibility Segregation (CQRS) pattern separates write operations (commands that produce events) from read operations (queries against materialised projections). Commands validate business rules and emit domain events like `WorkOrderCreated`, `WorkOrderAssigned`, `FailureModeRecorded`, `MeterReadingTaken`, `PartConsumed`. Read-side projections are purpose-built tables optimised for specific query patterns -- dashboard KPIs, work order lists, asset timelines, compliance reports.

This architecture is a natural fit for maintenance management because the domain inherently demands a complete, tamper-proof history of everything that happened to every asset. OSHA PSM, FDA 21 CFR Part 11, and ISO 55001 all require auditable records of who did what, when, and why. With event sourcing, the audit trail is not a secondary concern bolted onto a CRUD system -- it IS the system. Temporal queries ("what was the status of this asset on March 15th?", "show me all changes to this work order") are trivially answered by replaying events up to a point in time.

**Best for:** Organisations where full audit trail, temporal queries, and regulatory compliance are paramount -- pharmaceutical manufacturing, nuclear facilities, Oil & Gas process safety, and any environment subject to FDA 21 CFR Part 11 or OSHA PSM.

**Trade-offs:**
- (+) Complete, immutable, tamper-proof audit trail by design -- not an afterthought
- (+) Temporal queries are trivial: reconstruct state at any point in time
- (+) AI/ML can analyse event streams for pattern detection (failure prediction, anomaly detection)
- (+) Read models can be rebuilt or new projections created without data loss
- (+) Natural alignment with webhook/event-driven integrations (events are first-class)
- (-) Higher implementation complexity; developers must understand event sourcing patterns
- (-) Event schema evolution requires careful versioning (upcasters)
- (-) Read-side projections must be maintained and kept in sync
- (-) Eventual consistency between write and read sides; not instant
- (-) Higher storage requirements (every change is stored, not just current state)
- (-) Debugging can be harder when current state must be derived from event replay

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 55000/55001 | Asset lifecycle events (commissioned, decommissioned, disposed) stored as immutable events, providing verifiable lifecycle documentation |
| ISO 13306 | Maintenance terminology embedded in event type names (e.g., `CorrectiveMaintenanceStarted`, `PreventiveMaintenanceCompleted`) |
| ISO 14224 | Failure data captured as structured event payloads with ISO 14224 failure mode, cause, and mechanism codes |
| OSHA PSM (29 CFR 1910.119) | Mechanical integrity events form a tamper-proof compliance record; event immutability satisfies PSM documentation requirements |
| FDA 21 CFR Part 11 | Event immutability provides inherent Part 11 compliance for electronic records; electronic signatures are events themselves |
| MIMOSA CRIS | Event categories map to MIMOSA CRIS data exchange categories (asset register, work management, condition monitoring) |
| RFC 7519 (JWT) | Command authentication tokens included in event metadata for non-repudiation |
| OPC UA / MQTT | Sensor data arrives as events, naturally fitting the event-sourced architecture |

---

## Event Store -- Core Schema

```sql
-- ============================================================
-- EVENT STORE: The single source of truth
-- ============================================================

CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type     VARCHAR(50) NOT NULL,
        -- work_order, asset, pm_schedule, part_inventory, purchase_order, sensor_device
    stream_id       UUID NOT NULL,           -- the aggregate/entity ID
    event_type      VARCHAR(100) NOT NULL,   -- e.g., WorkOrderCreated, AssetMeterUpdated
    event_version   INTEGER NOT NULL,        -- sequential version within the stream
    tenant_id       UUID NOT NULL,
    site_id         UUID,

    -- Event payload (the facts that changed)
    payload         JSONB NOT NULL,

    -- Metadata (who, when, why, from where)
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata contains: {
    --   "user_id": "uuid",
    --   "user_email": "...",
    --   "correlation_id": "uuid",   -- ties related events together
    --   "causation_id": "uuid",     -- the event/command that caused this
    --   "ip_address": "...",
    --   "user_agent": "...",
    --   "source": "web|mobile|api|iot|system"
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- Optimistic concurrency: unique version per stream
    UNIQUE(stream_type, stream_id, event_version)
);

-- Primary query: get all events for a stream (replay)
CREATE INDEX idx_event_stream ON event_store(stream_type, stream_id, event_version);

-- Global ordering for projections
CREATE INDEX idx_event_created ON event_store(created_at);

-- Tenant-scoped queries
CREATE INDEX idx_event_tenant ON event_store(tenant_id, created_at);

-- Event type queries (for building specific projections)
CREATE INDEX idx_event_type ON event_store(event_type, created_at);

-- Partition by month for storage management
-- In production, this table would be range-partitioned on created_at
```

## Event Taxonomy

```sql
-- ============================================================
-- EVENT TYPE REGISTRY (documentation and schema validation)
-- ============================================================

CREATE TABLE event_type_registry (
    event_type      VARCHAR(100) PRIMARY KEY,
    stream_type     VARCHAR(50) NOT NULL,
    description     TEXT NOT NULL,
    payload_schema  JSONB NOT NULL,   -- JSON Schema for validation
    version         INTEGER NOT NULL DEFAULT 1,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Example event types and their payloads:
--
-- WorkOrderCreated:
-- {
--   "wo_number": "WO-2026-00142",
--   "title": "Replace pump seal - P-101A",
--   "wo_type": "corrective",           -- ISO 13306
--   "priority": "high",
--   "asset_id": "uuid",
--   "location_id": "uuid",
--   "description": "Seal leak detected during routine inspection",
--   "requested_by": "uuid"
-- }
--
-- WorkOrderAssigned:
-- {
--   "assigned_to": "uuid",
--   "assigned_team": "uuid",
--   "scheduled_start": "2026-05-25T08:00:00Z",
--   "scheduled_end": "2026-05-25T12:00:00Z",
--   "estimated_hours": 4
-- }
--
-- WorkOrderStatusChanged:
-- {
--   "old_status": "scheduled",
--   "new_status": "in_progress",
--   "reason": "Technician arrived on site"
-- }
--
-- FailureModeRecorded:
-- {
--   "failure_mode": "ELU",              -- ISO 14224: External Leakage - Utility
--   "failure_cause": "WEA",             -- Wear
--   "failure_mechanism": "ERO",         -- Erosion
--   "detection_method": "INS",          -- Inspection
--   "failure_description": "Mechanical seal wear causing utility fluid leakage"
-- }
--
-- PartConsumed:
-- {
--   "part_id": "uuid",
--   "part_number": "SEAL-P101-MECH",
--   "quantity": 1,
--   "storeroom_id": "uuid",
--   "unit_cost": 485.00
-- }
--
-- MeterReadingTaken:
-- {
--   "asset_id": "uuid",
--   "reading_value": 12450.5,
--   "meter_unit": "hours",
--   "previous_reading": 12200.0
-- }
--
-- AssetCommissioned:
-- {
--   "asset_tag": "P-101A",
--   "name": "Centrifugal Pump P-101A",
--   "equipment_class": "PUMP",
--   "serial_number": "SN-20250142",
--   "manufacturer": "Flowserve",
--   "install_date": "2026-03-15",
--   "criticality": "high",
--   "location_id": "uuid"
-- }
--
-- SensorThresholdBreached:
-- {
--   "sensor_device_id": "uuid",
--   "asset_id": "uuid",
--   "reading_value": 12.8,
--   "threshold_value": 11.0,
--   "threshold_type": "alarm_high",
--   "unit": "mm/s",
--   "auto_wo_created": true,
--   "wo_stream_id": "uuid"
-- }
--
-- ElectronicSignatureApplied:
-- {
--   "entity_type": "work_order",
--   "entity_id": "uuid",
--   "signer_id": "uuid",
--   "signature_meaning": "completed",   -- FDA 21 CFR Part 11
--   "signer_name": "Jane Smith",
--   "signer_title": "Maintenance Engineer"
-- }
--
-- PMScheduleTriggered:
-- {
--   "pm_schedule_id": "uuid",
--   "trigger_type": "time_based",
--   "trigger_reason": "30-day interval elapsed",
--   "generated_wo_id": "uuid",
--   "asset_id": "uuid"
-- }
```

## Snapshot Store (Performance Optimisation)

```sql
-- ============================================================
-- SNAPSHOT STORE (avoid replaying full event streams)
-- ============================================================

CREATE TABLE event_snapshot (
    stream_type     VARCHAR(50) NOT NULL,
    stream_id       UUID NOT NULL,
    snapshot_version INTEGER NOT NULL,       -- event_version at snapshot time
    state           JSONB NOT NULL,          -- full aggregate state at this version
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_type, stream_id, snapshot_version)
);

-- Take snapshots every N events (e.g., every 100 events per stream)
-- To rebuild current state: load latest snapshot + replay events since snapshot_version
```

## Read-Side Projections (CQRS Query Models)

```sql
-- ============================================================
-- PROJECTION: Work Order List (optimised for dashboard queries)
-- ============================================================

CREATE TABLE proj_work_order (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    site_id         UUID NOT NULL,
    wo_number       VARCHAR(50) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    wo_type         VARCHAR(30) NOT NULL,
    priority        VARCHAR(20),
    status          VARCHAR(30) NOT NULL,
    asset_id        UUID,
    asset_name      VARCHAR(255),       -- denormalised for read performance
    asset_tag       VARCHAR(100),
    location_name   VARCHAR(255),       -- denormalised
    assigned_to_id  UUID,
    assigned_to_name VARCHAR(255),      -- denormalised
    team_name       VARCHAR(255),
    failure_mode    VARCHAR(50),
    failure_cause   VARCHAR(50),
    requested_date  TIMESTAMPTZ,
    scheduled_start TIMESTAMPTZ,
    scheduled_end   TIMESTAMPTZ,
    actual_start    TIMESTAMPTZ,
    actual_end      TIMESTAMPTZ,
    due_date        TIMESTAMPTZ,
    estimated_hours NUMERIC(8, 2),
    actual_hours    NUMERIC(8, 2),
    total_part_cost NUMERIC(14, 2),
    total_labour_cost NUMERIC(14, 2),
    downtime_minutes INTEGER,
    event_count     INTEGER NOT NULL DEFAULT 0,
    last_event_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_wo_tenant_status ON proj_work_order(tenant_id, status);
CREATE INDEX idx_proj_wo_tenant_site ON proj_work_order(tenant_id, site_id);
CREATE INDEX idx_proj_wo_asset ON proj_work_order(asset_id);
CREATE INDEX idx_proj_wo_assigned ON proj_work_order(assigned_to_id);
CREATE INDEX idx_proj_wo_due ON proj_work_order(due_date);
CREATE INDEX idx_proj_wo_type ON proj_work_order(wo_type);

-- ============================================================
-- PROJECTION: Asset Register
-- ============================================================

CREATE TABLE proj_asset (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    site_id         UUID NOT NULL,
    asset_tag       VARCHAR(100) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    equipment_class VARCHAR(50),
    manufacturer    VARCHAR(255),
    model           VARCHAR(255),
    serial_number   VARCHAR(100),
    status          VARCHAR(30) NOT NULL,
    criticality     VARCHAR(20),
    location_id     UUID,
    location_name   VARCHAR(255),       -- denormalised
    parent_asset_id UUID,
    install_date    DATE,
    warranty_expiry DATE,
    current_meter   NUMERIC(14, 2),
    meter_unit      VARCHAR(50),
    last_meter_date TIMESTAMPTZ,
    purchase_cost   NUMERIC(14, 2),
    total_maintenance_cost NUMERIC(14, 2) DEFAULT 0,
    failure_count   INTEGER DEFAULT 0,
    last_failure_date TIMESTAMPTZ,
    open_wo_count   INTEGER DEFAULT 0,
    next_pm_date    DATE,
    event_count     INTEGER NOT NULL DEFAULT 0,
    last_event_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_asset_tenant ON proj_asset(tenant_id);
CREATE INDEX idx_proj_asset_site ON proj_asset(tenant_id, site_id);
CREATE INDEX idx_proj_asset_status ON proj_asset(status);
CREATE INDEX idx_proj_asset_class ON proj_asset(equipment_class);
CREATE INDEX idx_proj_asset_criticality ON proj_asset(criticality);

-- ============================================================
-- PROJECTION: Part Inventory Levels
-- ============================================================

CREATE TABLE proj_part_inventory (
    part_id         UUID NOT NULL,
    storeroom_id    UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    part_number     VARCHAR(100) NOT NULL,
    part_name       VARCHAR(255) NOT NULL,
    storeroom_name  VARCHAR(255),
    site_id         UUID NOT NULL,
    quantity_on_hand NUMERIC(14, 2) NOT NULL DEFAULT 0,
    min_quantity    NUMERIC(14, 2),
    max_quantity    NUMERIC(14, 2),
    reorder_point   NUMERIC(14, 2),
    unit_cost       NUMERIC(14, 2),
    is_below_reorder BOOLEAN NOT NULL DEFAULT false,
    last_consumed_at TIMESTAMPTZ,
    last_restocked_at TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (part_id, storeroom_id)
);

CREATE INDEX idx_proj_inv_tenant ON proj_part_inventory(tenant_id);
CREATE INDEX idx_proj_inv_reorder ON proj_part_inventory(is_below_reorder) WHERE is_below_reorder = true;

-- ============================================================
-- PROJECTION: Asset Timeline (full history view)
-- ============================================================

CREATE TABLE proj_asset_timeline (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    asset_id        UUID NOT NULL,
    event_time      TIMESTAMPTZ NOT NULL,
    event_type      VARCHAR(100) NOT NULL,
    summary         TEXT NOT NULL,            -- human-readable summary
    details         JSONB,                    -- event payload for drill-down
    user_name       VARCHAR(255),
    work_order_id   UUID,
    wo_number       VARCHAR(50),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_timeline_asset ON proj_asset_timeline(asset_id, event_time DESC);
CREATE INDEX idx_timeline_tenant ON proj_asset_timeline(tenant_id, event_time DESC);

-- ============================================================
-- PROJECTION: KPI Dashboard
-- ============================================================

CREATE TABLE proj_site_kpi (
    tenant_id       UUID NOT NULL,
    site_id         UUID NOT NULL,
    period_date     DATE NOT NULL,           -- daily snapshot
    open_wo_count   INTEGER DEFAULT 0,
    overdue_wo_count INTEGER DEFAULT 0,
    completed_wo_count INTEGER DEFAULT 0,
    avg_mttr_hours  NUMERIC(10, 2),
    avg_mtbf_hours  NUMERIC(10, 2),
    pm_compliance_pct NUMERIC(5, 2),
    total_downtime_hours NUMERIC(10, 2),
    total_maintenance_cost NUMERIC(14, 2),
    parts_below_reorder INTEGER DEFAULT 0,
    sensor_alarms   INTEGER DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, site_id, period_date)
);

-- ============================================================
-- PROJECTION: Compliance Audit Report
-- ============================================================

CREATE TABLE proj_compliance_event (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    site_id         UUID,
    event_time      TIMESTAMPTZ NOT NULL,
    compliance_type VARCHAR(50) NOT NULL,     -- osha_psm, fda_21cfr, iso_55001
    event_type      VARCHAR(100) NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    description     TEXT NOT NULL,
    user_id         UUID,
    user_name       VARCHAR(255),
    signature_id    UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_compliance_tenant ON proj_compliance_event(tenant_id, compliance_type, event_time DESC);
```

## Reference Data Tables

```sql
-- ============================================================
-- REFERENCE DATA (shared, rarely changing, read-heavy)
-- These are standard relational tables, not event-sourced.
-- ============================================================

CREATE TABLE ref_tenant (
    id              UUID PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    subscription    VARCHAR(50) NOT NULL DEFAULT 'standard',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ref_site (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL REFERENCES ref_tenant(id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50),
    country_code    CHAR(2),
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ref_app_user (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL REFERENCES ref_tenant(id),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ref_equipment_class (
    id              UUID PRIMARY KEY,
    code            VARCHAR(50) NOT NULL UNIQUE,
    name            VARCHAR(255) NOT NULL,
    iso_14224_level VARCHAR(20),
    parent_class_id UUID REFERENCES ref_equipment_class(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ref_failure_mode (
    code            VARCHAR(50) PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    equipment_class_code VARCHAR(50),
    iso_14224_code  VARCHAR(20)
);

CREATE TABLE ref_failure_cause (
    code            VARCHAR(50) PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    cause_category  VARCHAR(50),
    iso_14224_code  VARCHAR(20)
);

CREATE TABLE ref_maintenance_action (
    code            VARCHAR(50) PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    category        VARCHAR(50) NOT NULL  -- corrective, preventive, improvement
);
```

## Projection Tracking

```sql
-- ============================================================
-- PROJECTION CHECKPOINT (tracks what events each projection has processed)
-- ============================================================

CREATE TABLE projection_checkpoint (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id   UUID,
    last_event_time TIMESTAMPTZ,
    events_processed BIGINT NOT NULL DEFAULT 0,
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now(),
    status          VARCHAR(20) NOT NULL DEFAULT 'running'  -- running, paused, rebuilding, error
);

-- Example rows:
-- ('proj_work_order', 'uuid', '2026-05-22T10:30:00Z', 1542890, now(), 'running')
-- ('proj_asset', 'uuid', '2026-05-22T10:30:00Z', 1542890, now(), 'running')
-- ('proj_compliance_event', 'uuid', '2026-05-22T10:29:55Z', 1542885, now(), 'running')
```

## Example: Temporal Query

```sql
-- "What was the status of work order WO-2026-00142 on May 10th?"
-- Replay events up to the target timestamp:

SELECT event_type, payload, created_at
FROM event_store
WHERE stream_type = 'work_order'
  AND stream_id = '...'
  AND created_at <= '2026-05-10T23:59:59Z'
ORDER BY event_version ASC;

-- The application replays these events to reconstruct state at that point in time.
-- No separate "history" table needed -- the event store IS the history.
```

## Example: Failure Pattern Analysis for AI

```sql
-- Extract failure events for ML model training:
-- "All failure recordings for centrifugal pumps in the last 2 years"

SELECT
    es.stream_id AS work_order_id,
    es.payload->>'failure_mode' AS failure_mode,
    es.payload->>'failure_cause' AS failure_cause,
    es.payload->>'failure_mechanism' AS failure_mechanism,
    es.payload->>'detection_method' AS detection_method,
    es.created_at AS failure_recorded_at,
    pa.asset_tag,
    pa.equipment_class,
    pa.current_meter
FROM event_store es
JOIN proj_asset pa ON pa.id = (
    SELECT pw.asset_id FROM proj_work_order pw WHERE pw.id = es.stream_id
)
WHERE es.event_type = 'FailureModeRecorded'
  AND pa.equipment_class = 'PUMP'
  AND es.created_at >= now() - INTERVAL '2 years'
ORDER BY es.created_at;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 3 | event_store, event_type_registry, event_snapshot |
| Read Projections | 6 | proj_work_order, proj_asset, proj_part_inventory, proj_asset_timeline, proj_site_kpi, proj_compliance_event |
| Reference Data | 7 | ref_tenant, ref_site, ref_app_user, ref_equipment_class, ref_failure_mode, ref_failure_cause, ref_maintenance_action |
| Infrastructure | 1 | projection_checkpoint |
| **Total** | **17** | Significantly fewer tables than the normalized model; complexity shifts to event handlers and projectors |

---

## Key Design Decisions

1. **Single event store table for all aggregates.** Rather than separate event tables per entity, a single `event_store` table with `stream_type` discrimination simplifies infrastructure, backup, and replication. The `(stream_type, stream_id, event_version)` unique constraint provides optimistic concurrency control.

2. **JSONB payloads with registered schemas.** Event payloads are stored as JSONB for flexibility, but the `event_type_registry` table documents the expected JSON Schema for each event type. Application-level validation ensures payload conformance before events are persisted.

3. **Snapshots every N events.** For streams with many events (e.g., a high-activity asset with thousands of work orders), the snapshot store caches the aggregate state periodically. Replay starts from the latest snapshot rather than from event zero, keeping read latency bounded.

4. **Denormalised read projections.** Each projection table is purpose-built for a specific query pattern and includes denormalised data (e.g., `asset_name` on the work order projection). This eliminates joins on the read side and delivers sub-millisecond dashboard queries. The trade-off is that projections must be updated when events arrive.

5. **Projection checkpoints for reliable catch-up.** The `projection_checkpoint` table tracks which events each projector has processed, enabling reliable catch-up after downtime or failures without reprocessing the entire event stream.

6. **Reference data in standard relational tables.** Slowly-changing reference data (tenants, sites, users, equipment classes, failure taxonomies) is stored in conventional tables. Event-sourcing is reserved for domain entities with rich state transitions (work orders, assets, inventory). This is a pragmatic hybrid rather than pure event sourcing.

7. **Compliance audit trail is inherent.** There is no separate audit_log table. The event store itself IS the audit trail. Every change to every entity is an immutable event with metadata recording who, when, from where, and why. This inherently satisfies FDA 21 CFR Part 11 and OSHA PSM documentation requirements.

8. **Event stream as webhook payload.** Domain events map directly to webhook notifications. When a `WorkOrderStatusChanged` event is stored, the same event payload can be published to webhook subscribers and MQTT topics, eliminating the need for a separate notification layer.

9. **AI-friendly event streams.** The append-only event store is ideal for training ML models. Failure patterns, maintenance intervals, sensor threshold breaches, and work order lifecycle events can be extracted as training datasets without impacting operational queries. The `proj_asset_timeline` projection provides a pre-built feature for AI root-cause analysis.

10. **Event schema versioning via `event_type_registry`.** When event schemas evolve (e.g., adding a field to `WorkOrderCreated`), the registry tracks the version. Upcasters transform old event formats to current format during replay, ensuring backward compatibility without migrating historical events.
