# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: CMMS (Maintenance Management) · Created: 2026-05-22

## Philosophy

This model follows classical third-normal-form (3NF) relational design. Every real-world concept -- asset, work order, failure mode, spare part, technician, location -- gets its own dedicated table with strict foreign-key relationships. Reference data (failure modes, maintenance action types, priority levels, equipment classes) is stored in separate lookup tables aligned to ISO 14224 and ISO 13306 taxonomies, ensuring that every coded value in the system maps to an industry-standard definition.

The approach prioritises data integrity above all else. No piece of information is stored in more than one place, so updates propagate automatically and there is a single source of truth for every entity. Cross-entity queries (e.g., "show me all high-criticality assets at Site X that have overdue PMs and low spare-part stock") are handled through well-indexed joins. Audit trails are maintained via separate audit log tables that record who changed what, when.

This is the architecture used by IBM Maximo, SAP Plant Maintenance, and Infor EAM -- the enterprise-grade CMMS platforms that dominate regulated industries. It is proven at scale in petrochemical plants, nuclear facilities, and pharmaceutical manufacturing where regulatory compliance (OSHA PSM, FDA 21 CFR Part 11, ISO 55001) demands verifiable data lineage.

**Best for:** Organisations in regulated industries (Oil & Gas, Pharma, Utilities) that need ISO 55001 certification readiness, full compliance audit trails, and complex cross-entity reporting.

**Trade-offs:**
- (+) Strongest data integrity; no redundancy, no anomalies
- (+) Natural alignment with ISO 14224 equipment taxonomy and ISO 13306 terminology
- (+) Complex cross-entity queries are straightforward with SQL joins
- (+) Mature tooling; every ORM and reporting tool understands normalized schemas
- (-) Higher table count (80-100+ tables) increases schema complexity
- (-) Schema changes require migrations; adding a new field means ALTER TABLE
- (-) Junction tables for many-to-many relationships add join overhead
- (-) Multi-industry flexibility requires extensive reference-data seeding per deployment

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 55000/55001 | Asset lifecycle tables (procurement, commissioning, operation, decommissioning, disposal) map directly to ISO 55001 asset management system requirements |
| ISO 13306 | Maintenance terminology drives work order type codes (`corrective`, `preventive`, `condition_based`, `predictive`) and maintenance action classification |
| ISO 14224 | Nine-level equipment taxonomy (Industry > Plant > Section > Unit > Equipment > Subunit > Component > Part > Maintainable Item) structures the `asset` and `asset_hierarchy` tables |
| ISO 31000 | Risk assessment fields on assets (criticality, consequence, likelihood) align to ISO 31000 risk management guidelines |
| OSHA PSM (29 CFR 1910.119) | Mechanical integrity fields, inspection schedules, and PSM audit trail tables |
| FDA 21 CFR Part 11 | Electronic signature table, signature meaning codes, and Part 11-compliant audit trail design |
| MIMOSA CRIS | Entity relationship patterns for asset register, work management, and condition monitoring align to MIMOSA Common Relational Information Schema |
| OPC UA / MQTT | Sensor data ingestion tables and device registry aligned to OPC UA information model |

---

## Core Tables -- Tenant & Organisation

```sql
-- ============================================================
-- TENANT & ORGANISATION
-- ============================================================

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    subscription    VARCHAR(50) NOT NULL DEFAULT 'standard',  -- free, standard, enterprise
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE site (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50),
    address_line1   VARCHAR(255),
    address_line2   VARCHAR(255),
    city            VARCHAR(100),
    state_province  VARCHAR(100),
    postal_code     VARCHAR(20),
    country_code    CHAR(2),          -- ISO 3166-1 alpha-2
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    latitude        NUMERIC(10, 7),
    longitude       NUMERIC(10, 7),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_site_tenant ON site(tenant_id);
CREATE INDEX idx_site_country ON site(country_code);
```

## Core Tables -- User & RBAC

```sql
-- ============================================================
-- USER & ROLE-BASED ACCESS CONTROL
-- ============================================================

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    phone           VARCHAR(50),
    job_title       VARCHAR(100),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,  -- admin, maintenance_manager, planner, technician, viewer
    description     TEXT,
    permissions     JSONB NOT NULL DEFAULT '[]',
    is_system       BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

CREATE TABLE user_role (
    user_id         UUID NOT NULL REFERENCES app_user(id),
    role_id         UUID NOT NULL REFERENCES role(id),
    site_id         UUID REFERENCES site(id),  -- NULL = all sites
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by      UUID REFERENCES app_user(id),
    PRIMARY KEY (user_id, role_id, site_id)
);

CREATE TABLE team (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    site_id         UUID REFERENCES site(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE team_member (
    team_id         UUID NOT NULL REFERENCES team(id),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    role_in_team    VARCHAR(50) DEFAULT 'member',  -- lead, member
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (team_id, user_id)
);

CREATE INDEX idx_user_tenant ON app_user(tenant_id);
CREATE INDEX idx_user_role_user ON user_role(user_id);
CREATE INDEX idx_user_role_site ON user_role(site_id);
```

## Core Tables -- Location & Asset Hierarchy

```sql
-- ============================================================
-- LOCATION HIERARCHY (ISO 14224 Levels 1-5)
-- ============================================================

CREATE TABLE location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    site_id         UUID NOT NULL REFERENCES site(id),
    parent_id       UUID REFERENCES location(id),
    code            VARCHAR(100) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    location_type   VARCHAR(50) NOT NULL,  -- building, floor, zone, area, functional_location
    description     TEXT,
    path            TEXT,                  -- materialised path: /site/building/floor/zone
    depth           INTEGER NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, site_id, code)
);

CREATE INDEX idx_location_parent ON location(parent_id);
CREATE INDEX idx_location_site ON location(site_id);
CREATE INDEX idx_location_path ON location(path text_pattern_ops);

-- ============================================================
-- EQUIPMENT CLASS (ISO 14224 taxonomy reference data)
-- ============================================================

CREATE TABLE equipment_class (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(50) NOT NULL UNIQUE,    -- e.g., 'PUMP', 'COMPRESSOR', 'VALVE'
    name            VARCHAR(255) NOT NULL,
    iso_14224_level VARCHAR(20),                    -- equipment_unit, subunit, component
    parent_class_id UUID REFERENCES equipment_class(id),
    description     TEXT,
    default_attributes JSONB NOT NULL DEFAULT '{}',  -- default attribute schema for this class
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- ASSET (ISO 14224 Levels 6-9)
-- ============================================================

CREATE TABLE asset (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    site_id         UUID NOT NULL REFERENCES site(id),
    location_id     UUID REFERENCES location(id),
    parent_asset_id UUID REFERENCES asset(id),
    equipment_class_id UUID REFERENCES equipment_class(id),

    -- Identity
    asset_tag       VARCHAR(100) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    serial_number   VARCHAR(100),
    model           VARCHAR(255),
    manufacturer    VARCHAR(255),
    barcode         VARCHAR(100),

    -- Lifecycle (ISO 55001)
    status          VARCHAR(30) NOT NULL DEFAULT 'active',
        -- planned, procured, commissioned, active, idle, decommissioned, disposed
    criticality     VARCHAR(20) NOT NULL DEFAULT 'medium',
        -- critical, high, medium, low  (ISO 31000 risk-informed)
    install_date    DATE,
    warranty_expiry DATE,
    expected_life_years INTEGER,
    purchase_cost   NUMERIC(14, 2),
    replacement_cost NUMERIC(14, 2),
    salvage_value   NUMERIC(14, 2),
    currency_code   CHAR(3) DEFAULT 'USD',  -- ISO 4217

    -- Operational
    operating_environment VARCHAR(100),  -- ISO 14224: indoor, outdoor, subsea, etc.
    operating_mode  VARCHAR(50),          -- continuous, standby, intermittent

    -- Metering
    has_meter       BOOLEAN NOT NULL DEFAULT false,
    meter_unit      VARCHAR(50),           -- hours, km, cycles, litres
    current_meter   NUMERIC(14, 2),
    last_meter_date TIMESTAMPTZ,

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, asset_tag)
);

CREATE INDEX idx_asset_tenant_site ON asset(tenant_id, site_id);
CREATE INDEX idx_asset_location ON asset(location_id);
CREATE INDEX idx_asset_parent ON asset(parent_asset_id);
CREATE INDEX idx_asset_class ON asset(equipment_class_id);
CREATE INDEX idx_asset_status ON asset(status);
CREATE INDEX idx_asset_criticality ON asset(criticality);
```

## Core Tables -- Failure Taxonomy (ISO 14224 / ISO 13306)

```sql
-- ============================================================
-- FAILURE TAXONOMY (ISO 14224 / ISO 13306)
-- ============================================================

CREATE TABLE failure_mode (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(50) NOT NULL UNIQUE,
    name            VARCHAR(255) NOT NULL,
    equipment_class_id UUID REFERENCES equipment_class(id),  -- NULL = applies to all
    description     TEXT,
    iso_14224_code  VARCHAR(20),     -- ISO 14224 failure mode code
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
-- Examples: FTS (Fail to Start), STP (Spurious Stop), ELU (External Leakage - Utility),
--           AIR (Abnormal Instrument Reading), BRD (Breakdown), UST (Unable to Stop)

CREATE TABLE failure_cause (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(50) NOT NULL UNIQUE,
    name            VARCHAR(255) NOT NULL,
    cause_category  VARCHAR(50),     -- design, fabrication, installation, operation, maintenance, management
    description     TEXT,
    iso_14224_code  VARCHAR(20),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE failure_mechanism (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(50) NOT NULL UNIQUE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    iso_14224_code  VARCHAR(20),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
-- Examples: COR (Corrosion), ERO (Erosion), WEA (Wear), FAT (Fatigue),
--           BLO (Blockage), OVH (Overheating)

CREATE TABLE detection_method (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(50) NOT NULL UNIQUE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    iso_14224_code  VARCHAR(20),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
-- Examples: PRD (Periodic Maintenance), INS (Inspection), DEM (Demand),
--           OBS (Obvious), MON (Continuous Monitoring)

CREATE TABLE maintenance_action_type (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(50) NOT NULL UNIQUE,
    name            VARCHAR(255) NOT NULL,   -- ISO 13306 aligned
    category        VARCHAR(50) NOT NULL,     -- corrective, preventive, improvement
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
-- Examples: REP (Repair), RPL (Replace), MOD (Modify), ADJ (Adjust),
--           INS (Inspect), TST (Functional Test), OVH (Overhaul)
```

## Core Tables -- Work Order Management

```sql
-- ============================================================
-- WORK ORDER MANAGEMENT
-- ============================================================

CREATE TABLE work_order_priority (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    code            VARCHAR(20) NOT NULL,
    name            VARCHAR(100) NOT NULL,
    sort_order      INTEGER NOT NULL,
    sla_hours       INTEGER,              -- target completion time
    color_hex       VARCHAR(7),
    UNIQUE(tenant_id, code)
);

CREATE TABLE work_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    site_id         UUID NOT NULL REFERENCES site(id),

    -- Identity
    wo_number       VARCHAR(50) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    description     TEXT,

    -- Classification (ISO 13306)
    wo_type         VARCHAR(30) NOT NULL,
        -- corrective, preventive, condition_based, predictive, inspection, emergency
    priority_id     UUID REFERENCES work_order_priority(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'requested',
        -- requested, approved, planned, scheduled, in_progress, on_hold, completed, closed, cancelled

    -- Relationships
    asset_id        UUID REFERENCES asset(id),
    location_id     UUID REFERENCES location(id),
    parent_wo_id    UUID REFERENCES work_order(id),  -- for nested work orders
    pm_schedule_id  UUID,  -- FK added after pm_schedule table

    -- Assignment
    assigned_to     UUID REFERENCES app_user(id),
    assigned_team   UUID REFERENCES team(id),
    requested_by    UUID REFERENCES app_user(id),
    approved_by     UUID REFERENCES app_user(id),

    -- Failure analysis (ISO 14224)
    failure_mode_id     UUID REFERENCES failure_mode(id),
    failure_cause_id    UUID REFERENCES failure_cause(id),
    failure_mechanism_id UUID REFERENCES failure_mechanism(id),
    detection_method_id UUID REFERENCES detection_method(id),
    maintenance_action_id UUID REFERENCES maintenance_action_type(id),
    failure_description TEXT,

    -- Scheduling
    requested_date  TIMESTAMPTZ,
    scheduled_start TIMESTAMPTZ,
    scheduled_end   TIMESTAMPTZ,
    actual_start    TIMESTAMPTZ,
    actual_end      TIMESTAMPTZ,
    due_date        TIMESTAMPTZ,

    -- Cost tracking
    estimated_hours NUMERIC(8, 2),
    actual_hours    NUMERIC(8, 2),
    estimated_cost  NUMERIC(14, 2),
    actual_cost     NUMERIC(14, 2),
    currency_code   CHAR(3) DEFAULT 'USD',

    -- Downtime
    downtime_start  TIMESTAMPTZ,
    downtime_end    TIMESTAMPTZ,
    downtime_minutes INTEGER,

    -- Meter reading at completion
    meter_reading   NUMERIC(14, 2),

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    closed_at       TIMESTAMPTZ,
    UNIQUE(tenant_id, wo_number)
);

CREATE INDEX idx_wo_tenant_site ON work_order(tenant_id, site_id);
CREATE INDEX idx_wo_status ON work_order(status);
CREATE INDEX idx_wo_asset ON work_order(asset_id);
CREATE INDEX idx_wo_assigned ON work_order(assigned_to);
CREATE INDEX idx_wo_type ON work_order(wo_type);
CREATE INDEX idx_wo_due_date ON work_order(due_date);
CREATE INDEX idx_wo_scheduled ON work_order(scheduled_start);

-- ============================================================
-- WORK ORDER TASKS (checklist items / procedures)
-- ============================================================

CREATE TABLE work_order_task (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_id   UUID NOT NULL REFERENCES work_order(id) ON DELETE CASCADE,
    sequence        INTEGER NOT NULL,
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    task_type       VARCHAR(30) DEFAULT 'action',  -- action, inspection, measurement, signature
    is_required     BOOLEAN NOT NULL DEFAULT true,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',  -- pending, in_progress, completed, skipped
    completed_by    UUID REFERENCES app_user(id),
    completed_at    TIMESTAMPTZ,
    measurement_value NUMERIC(14, 4),
    measurement_unit  VARCHAR(50),
    measurement_min   NUMERIC(14, 4),
    measurement_max   NUMERIC(14, 4),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_wo_task_wo ON work_order_task(work_order_id);

-- ============================================================
-- WORK ORDER LABOUR LOG
-- ============================================================

CREATE TABLE work_order_labour (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_id   UUID NOT NULL REFERENCES work_order(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES app_user(id),
    start_time      TIMESTAMPTZ NOT NULL,
    end_time        TIMESTAMPTZ,
    hours           NUMERIC(6, 2),
    labour_type     VARCHAR(50) DEFAULT 'regular',  -- regular, overtime, contractor
    hourly_rate     NUMERIC(10, 2),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_wo_labour_wo ON work_order_labour(work_order_id);
CREATE INDEX idx_wo_labour_user ON work_order_labour(user_id);

-- ============================================================
-- WORK ORDER COMMENTS & ATTACHMENTS
-- ============================================================

CREATE TABLE work_order_comment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_id   UUID NOT NULL REFERENCES work_order(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES app_user(id),
    comment_text    TEXT NOT NULL,
    is_internal     BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_wo_comment_wo ON work_order_comment(work_order_id);

CREATE TABLE attachment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    entity_type     VARCHAR(50) NOT NULL,   -- work_order, asset, pm_schedule, inspection
    entity_id       UUID NOT NULL,
    file_name       VARCHAR(500) NOT NULL,
    file_type       VARCHAR(100),           -- MIME type
    file_size_bytes BIGINT,
    storage_path    TEXT NOT NULL,           -- S3 key or file path
    uploaded_by     UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_attachment_entity ON attachment(entity_type, entity_id);
```

## Core Tables -- Preventive Maintenance

```sql
-- ============================================================
-- PREVENTIVE MAINTENANCE SCHEDULING
-- ============================================================

CREATE TABLE pm_schedule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    site_id         UUID NOT NULL REFERENCES site(id),
    asset_id        UUID NOT NULL REFERENCES asset(id),

    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,

    -- Trigger type
    trigger_type    VARCHAR(30) NOT NULL,   -- time_based, meter_based, condition_based, event_based
    frequency_value INTEGER,                -- e.g., 30
    frequency_unit  VARCHAR(20),            -- days, weeks, months, years
    meter_interval  NUMERIC(14, 2),         -- e.g., every 500 hours
    meter_unit      VARCHAR(50),

    -- Scheduling
    start_date      DATE NOT NULL,
    end_date        DATE,
    lead_time_days  INTEGER DEFAULT 0,
    next_due_date   DATE,
    last_completed  TIMESTAMPTZ,

    -- Work order template
    wo_type         VARCHAR(30) NOT NULL DEFAULT 'preventive',
    priority_id     UUID REFERENCES work_order_priority(id),
    assigned_to     UUID REFERENCES app_user(id),
    assigned_team   UUID REFERENCES team(id),
    estimated_hours NUMERIC(8, 2),

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pm_tenant_site ON pm_schedule(tenant_id, site_id);
CREATE INDEX idx_pm_asset ON pm_schedule(asset_id);
CREATE INDEX idx_pm_next_due ON pm_schedule(next_due_date);

-- Add FK from work_order to pm_schedule
ALTER TABLE work_order
    ADD CONSTRAINT fk_wo_pm FOREIGN KEY (pm_schedule_id) REFERENCES pm_schedule(id);

-- ============================================================
-- PM TASK TEMPLATE (reusable checklist for PM-generated WOs)
-- ============================================================

CREATE TABLE pm_task_template (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    pm_schedule_id  UUID NOT NULL REFERENCES pm_schedule(id) ON DELETE CASCADE,
    sequence        INTEGER NOT NULL,
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    task_type       VARCHAR(30) DEFAULT 'action',
    is_required     BOOLEAN NOT NULL DEFAULT true,
    measurement_unit  VARCHAR(50),
    measurement_min   NUMERIC(14, 4),
    measurement_max   NUMERIC(14, 4),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pm_task_pm ON pm_task_template(pm_schedule_id);
```

## Core Tables -- Spare Parts & Inventory

```sql
-- ============================================================
-- SPARE PARTS & INVENTORY (MRO)
-- ============================================================

CREATE TABLE part_category (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    parent_id       UUID REFERENCES part_category(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE part (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    category_id     UUID REFERENCES part_category(id),
    part_number     VARCHAR(100) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    unit_of_measure VARCHAR(30) DEFAULT 'each',
    unit_cost       NUMERIC(14, 2),
    currency_code   CHAR(3) DEFAULT 'USD',
    barcode         VARCHAR(100),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, part_number)
);

CREATE INDEX idx_part_tenant ON part(tenant_id);
CREATE INDEX idx_part_category ON part(category_id);

CREATE TABLE storeroom (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    site_id         UUID NOT NULL REFERENCES site(id),
    name            VARCHAR(255) NOT NULL,
    location_id     UUID REFERENCES location(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE part_inventory (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    part_id         UUID NOT NULL REFERENCES part(id),
    storeroom_id    UUID NOT NULL REFERENCES storeroom(id),
    quantity_on_hand NUMERIC(14, 2) NOT NULL DEFAULT 0,
    min_quantity    NUMERIC(14, 2),
    max_quantity    NUMERIC(14, 2),
    reorder_point   NUMERIC(14, 2),
    reorder_qty     NUMERIC(14, 2),
    bin_location    VARCHAR(100),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(part_id, storeroom_id)
);

CREATE INDEX idx_part_inv_part ON part_inventory(part_id);
CREATE INDEX idx_part_inv_store ON part_inventory(storeroom_id);

-- Parts used on work orders
CREATE TABLE work_order_part (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_id   UUID NOT NULL REFERENCES work_order(id) ON DELETE CASCADE,
    part_id         UUID NOT NULL REFERENCES part(id),
    storeroom_id    UUID REFERENCES storeroom(id),
    quantity_used   NUMERIC(14, 2) NOT NULL,
    unit_cost       NUMERIC(14, 2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_wo_part_wo ON work_order_part(work_order_id);

-- Parts typically needed for an asset (BOM)
CREATE TABLE asset_part_bom (
    asset_id        UUID NOT NULL REFERENCES asset(id),
    part_id         UUID NOT NULL REFERENCES part(id),
    quantity        NUMERIC(14, 2) NOT NULL DEFAULT 1,
    is_critical     BOOLEAN NOT NULL DEFAULT false,
    notes           TEXT,
    PRIMARY KEY (asset_id, part_id)
);
```

## Core Tables -- Vendor & Purchase Orders

```sql
-- ============================================================
-- VENDOR & PURCHASE ORDERS
-- ============================================================

CREATE TABLE vendor (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50),
    contact_name    VARCHAR(255),
    email           VARCHAR(255),
    phone           VARCHAR(50),
    address         TEXT,
    website         VARCHAR(500),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE purchase_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    po_number       VARCHAR(50) NOT NULL,
    vendor_id       UUID NOT NULL REFERENCES vendor(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
        -- draft, submitted, approved, ordered, partially_received, received, cancelled
    total_amount    NUMERIC(14, 2),
    currency_code   CHAR(3) DEFAULT 'USD',
    ordered_date    DATE,
    expected_date   DATE,
    received_date   DATE,
    created_by      UUID REFERENCES app_user(id),
    approved_by     UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, po_number)
);

CREATE TABLE purchase_order_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    purchase_order_id UUID NOT NULL REFERENCES purchase_order(id) ON DELETE CASCADE,
    part_id         UUID NOT NULL REFERENCES part(id),
    quantity        NUMERIC(14, 2) NOT NULL,
    unit_cost       NUMERIC(14, 2) NOT NULL,
    quantity_received NUMERIC(14, 2) DEFAULT 0,
    line_total      NUMERIC(14, 2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Core Tables -- IoT / Sensor

```sql
-- ============================================================
-- IoT DEVICE & SENSOR REGISTRY
-- ============================================================

CREATE TABLE sensor_device (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    asset_id        UUID REFERENCES asset(id),
    device_name     VARCHAR(255) NOT NULL,
    device_type     VARCHAR(100),           -- vibration, temperature, pressure, flow, level
    protocol        VARCHAR(50),            -- mqtt, opcua, modbus, http
    connection_uri  VARCHAR(500),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_seen       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sensor_asset ON sensor_device(asset_id);

CREATE TABLE sensor_reading (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sensor_device_id UUID NOT NULL REFERENCES sensor_device(id),
    reading_time    TIMESTAMPTZ NOT NULL,
    value           NUMERIC(18, 6) NOT NULL,
    unit            VARCHAR(50),            -- celsius, psi, mm/s, rpm
    quality         VARCHAR(20) DEFAULT 'good',  -- good, suspect, bad
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sensor_reading_device_time ON sensor_reading(sensor_device_id, reading_time DESC);

CREATE TABLE sensor_threshold (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sensor_device_id UUID NOT NULL REFERENCES sensor_device(id),
    threshold_type  VARCHAR(20) NOT NULL,   -- warning_high, alarm_high, warning_low, alarm_low
    value           NUMERIC(18, 6) NOT NULL,
    auto_create_wo  BOOLEAN NOT NULL DEFAULT false,
    wo_priority_id  UUID REFERENCES work_order_priority(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Core Tables -- Compliance & Audit

```sql
-- ============================================================
-- COMPLIANCE & AUDIT TRAIL
-- ============================================================

-- FDA 21 CFR Part 11 electronic signatures
CREATE TABLE electronic_signature (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    entity_type     VARCHAR(50) NOT NULL,   -- work_order, inspection, pm_schedule
    entity_id       UUID NOT NULL,
    signer_id       UUID NOT NULL REFERENCES app_user(id),
    signature_meaning VARCHAR(100) NOT NULL, -- approved, reviewed, completed, authored
    signed_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    ip_address      INET,
    user_agent      TEXT
);

CREATE INDEX idx_esig_entity ON electronic_signature(entity_type, entity_id);

-- General audit trail
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    action          VARCHAR(20) NOT NULL,   -- created, updated, deleted, status_changed
    field_name      VARCHAR(100),
    old_value       TEXT,
    new_value       TEXT,
    user_id         UUID REFERENCES app_user(id),
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_tenant_time ON audit_log(tenant_id, created_at DESC);
CREATE INDEX idx_audit_user ON audit_log(user_id);

-- OSHA PSM inspection records
CREATE TABLE inspection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    asset_id        UUID NOT NULL REFERENCES asset(id),
    inspection_type VARCHAR(50) NOT NULL,   -- mechanical_integrity, pressure_test, visual, ndt
    scheduled_date  DATE,
    completed_date  DATE,
    inspector_id    UUID REFERENCES app_user(id),
    result          VARCHAR(30),            -- pass, fail, conditional
    findings        TEXT,
    next_due_date   DATE,
    work_order_id   UUID REFERENCES work_order(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_inspection_asset ON inspection(asset_id);
CREATE INDEX idx_inspection_next_due ON inspection(next_due_date);
```

## Core Tables -- KPI & Reporting

```sql
-- ============================================================
-- KPI METRICS (materialised for dashboard performance)
-- ============================================================

CREATE TABLE asset_kpi_snapshot (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    asset_id        UUID NOT NULL REFERENCES asset(id),
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    mtbf_hours      NUMERIC(10, 2),    -- Mean Time Between Failures
    mttr_hours      NUMERIC(10, 2),    -- Mean Time To Repair
    availability_pct NUMERIC(5, 2),    -- uptime percentage
    pm_compliance_pct NUMERIC(5, 2),   -- % of PMs completed on time
    total_downtime_hours NUMERIC(10, 2),
    total_maintenance_cost NUMERIC(14, 2),
    failure_count   INTEGER DEFAULT 0,
    wo_count        INTEGER DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(asset_id, period_start, period_end)
);

CREATE INDEX idx_kpi_asset_period ON asset_kpi_snapshot(asset_id, period_start);
CREATE INDEX idx_kpi_tenant ON asset_kpi_snapshot(tenant_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant & Organisation | 2 | tenant, site |
| User & RBAC | 4 | app_user, role, user_role, team, team_member |
| Location & Asset | 3 | location, equipment_class, asset |
| Failure Taxonomy | 5 | failure_mode, failure_cause, failure_mechanism, detection_method, maintenance_action_type |
| Work Order Management | 5 | work_order, work_order_task, work_order_labour, work_order_comment, attachment |
| Preventive Maintenance | 2 | pm_schedule, pm_task_template |
| Spare Parts & Inventory | 6 | part_category, part, storeroom, part_inventory, work_order_part, asset_part_bom |
| Vendor & Procurement | 3 | vendor, purchase_order, purchase_order_line |
| IoT / Sensor | 3 | sensor_device, sensor_reading, sensor_threshold |
| Compliance & Audit | 3 | electronic_signature, audit_log, inspection |
| KPI & Reporting | 1 | asset_kpi_snapshot |
| **Total** | **37** | |

---

## Key Design Decisions

1. **UUID primary keys throughout.** UUIDs enable multi-site data federation, offline mobile sync, and distributed ID generation without coordination. Every entity can be created on a mobile device and merged without collision.

2. **ISO 14224 equipment taxonomy as a first-class entity.** The `equipment_class` table implements the ISO 14224 hierarchical equipment classification, allowing assets to inherit default attributes and failure-mode coding from their equipment class. This is critical for benchmarking reliability data across sites and industries.

3. **ISO 13306-aligned work order classification.** The `wo_type` field uses ISO 13306 maintenance terminology (corrective, preventive, condition_based, predictive, inspection, emergency) ensuring consistent reporting and compliance documentation.

4. **Materialised path for location hierarchy.** The `path` column on the `location` table enables fast subtree queries (`WHERE path LIKE '/site-a/building-1/%'`) without recursive CTEs for common queries, while `parent_id` preserves referential integrity for structural modifications.

5. **Separate failure taxonomy tables per ISO 14224.** Failure mode, cause, mechanism, and detection method each get their own lookup table rather than being free-text fields. This enforces consistent coding that can be benchmarked against industry databases (e.g., OREDA) and supports AI-driven root-cause analysis.

6. **FDA 21 CFR Part 11 electronic signature table.** The `electronic_signature` table records who signed what, when, with what meaning, and from what IP/device. This is a dedicated table rather than a column on work_order because signatures may be required on multiple entity types (inspections, PMs, approvals).

7. **Multi-tenant with `tenant_id` on every business table.** Row-level multi-tenancy with `tenant_id` foreign keys enables a single database deployment serving many organisations. PostgreSQL Row-Level Security (RLS) policies can enforce tenant isolation at the database level.

8. **Sensor readings in a standard relational table.** In this normalized model, sensor readings are stored in a regular table with indexes on `(device_id, reading_time)`. For high-volume IoT deployments, this table would need partitioning or migration to a time-series extension (see Data Model Suggestion 4 for the time-series approach).

9. **KPI snapshots as materialised summaries.** Rather than computing MTBF, MTTR, and availability on every dashboard load, pre-computed snapshots are stored periodically. This trades storage for query performance on the most frequently accessed metrics.

10. **Attachment polymorphism via `entity_type` + `entity_id`.** A single attachment table serves work orders, assets, inspections, and PM schedules using a discriminator column pattern. This avoids proliferating attachment join tables while keeping a single storage and retrieval path.
