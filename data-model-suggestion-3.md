# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: CMMS (Maintenance Management) · Created: 2026-05-22

## Philosophy

This model keeps the relational backbone for core entities (assets, work orders, parts, users) where data integrity and cross-entity joins matter, but uses PostgreSQL JSONB columns extensively for variable, industry-specific, and tenant-customisable data. The insight is that a CMMS serving manufacturing plants, hospitals, commercial real estate, and utilities cannot have a single fixed schema that fits all of them. A pharmaceutical plant needs GMP batch tracking fields on work orders; an HVAC company needs refrigerant type and charge weight on assets; a data centre needs rack position and power draw. Rather than maintaining hundreds of nullable columns or building a full EAV (Entity-Attribute-Value) system, JSONB columns provide schema-flexible storage with full indexing and query capability.

The hybrid approach gives each tenant a configurable schema: a `field_definition` table defines what custom fields exist for each entity type per tenant, and JSONB columns on the core tables store the tenant-specific values. This is the pattern used by modern SaaS platforms (Jira's custom fields, HubSpot's custom properties, Salesforce's dynamic schema) and is well-proven at scale with PostgreSQL GIN indexes.

This model is explicitly designed for rapid MVP development and multi-industry deployment. The core schema covers 80% of all CMMS use cases with fixed relational columns. The remaining 20% -- the industry-specific, site-specific, and tenant-specific variations -- lives in JSONB. New fields can be added through the UI without database migrations.

**Best for:** SaaS platforms serving multiple industries (manufacturing, facilities, healthcare, hospitality) where tenants need custom fields, rapid feature iteration, and schema flexibility without sacrificing query performance on core data.

**Trade-offs:**
- (+) Fastest path to MVP -- core schema works immediately, custom fields added without migrations
- (+) Multi-industry flexibility -- each tenant configures their own custom attributes
- (+) No more ALTER TABLE for new fields -- custom data evolves through configuration
- (+) JSONB GIN indexes enable fast queries on custom fields
- (+) Simpler schema than fully normalized -- fewer tables, fewer joins
- (-) JSONB fields lack database-level constraint enforcement (validated at application layer)
- (-) Custom field queries are slightly slower than fixed-column queries
- (-) Schema documentation requires discipline -- JSONB contents must be documented externally
- (-) Data migration between JSONB and fixed columns adds complexity if a field "graduates"
- (-) Reporting tools may struggle with JSONB columns compared to flat relational columns

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 55000/55001 | Asset lifecycle status is a fixed relational column; industry-specific asset attributes stored in JSONB aligned to ISO 55001 requirements per deployment |
| ISO 13306 | Work order type classification uses fixed enum values; industry-specific maintenance action details in JSONB |
| ISO 14224 | Equipment class taxonomy is relational; class-specific attributes (e.g., pump design type, impeller material) stored in JSONB following ISO 14224 equipment-specific data requirements |
| ISO 31000 | Risk assessment fields (criticality matrix values) stored in asset JSONB `custom_fields`, with field definitions configured per tenant's risk framework |
| FDA 21 CFR Part 11 | Electronic signature and audit trail tables are fully relational (compliance-critical data never in JSONB) |
| OSHA PSM | PSM-specific inspection fields configured as custom field definitions for tenants in process industries |
| MIMOSA CRIS | Core entity structure aligns with MIMOSA CRIS categories; JSONB extensions accommodate CRIS-specific attributes per deployment |

---

## Tenant & Configuration

```sql
-- ============================================================
-- TENANT & CONFIGURATION
-- ============================================================

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    subscription    VARCHAR(50) NOT NULL DEFAULT 'standard',
    industry        VARCHAR(100),        -- manufacturing, healthcare, facilities, utilities, hospitality
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example: {
    --   "default_currency": "USD",
    --   "date_format": "YYYY-MM-DD",
    --   "require_esignatures": true,
    --   "enable_iot": true,
    --   "wo_number_prefix": "WO",
    --   "wo_number_sequence": 1
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE site (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50),
    country_code    CHAR(2),             -- ISO 3166-1 alpha-2
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    address         JSONB DEFAULT '{}',
    -- address example: {
    --   "line1": "123 Industrial Blvd",
    --   "city": "Houston",
    --   "state": "TX",
    --   "postal_code": "77001",
    --   "country": "US",
    --   "latitude": 29.7604,
    --   "longitude": -95.3698
    -- }
    settings        JSONB NOT NULL DEFAULT '{}',   -- site-level overrides
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_site_tenant ON site(tenant_id);

-- ============================================================
-- CUSTOM FIELD DEFINITIONS (per-tenant, per-entity configuration)
-- ============================================================

CREATE TABLE field_definition (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    entity_type     VARCHAR(50) NOT NULL,   -- asset, work_order, part, location
    field_key       VARCHAR(100) NOT NULL,   -- JSON key name (snake_case)
    field_label     VARCHAR(255) NOT NULL,   -- display label
    field_type      VARCHAR(30) NOT NULL,    -- text, number, date, boolean, select, multi_select, url
    is_required     BOOLEAN NOT NULL DEFAULT false,
    is_filterable   BOOLEAN NOT NULL DEFAULT false,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    options         JSONB,                    -- for select/multi_select: ["Option A", "Option B"]
    validation      JSONB,                    -- { "min": 0, "max": 100, "pattern": "..." }
    default_value   TEXT,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, entity_type, field_key)
);

CREATE INDEX idx_field_def_tenant_entity ON field_definition(tenant_id, entity_type);

-- Example field definitions:
--
-- Pharmaceutical tenant:
--   { entity_type: "work_order", field_key: "gmp_batch_number", field_label: "GMP Batch #",
--     field_type: "text", is_required: true }
--   { entity_type: "work_order", field_key: "cleanroom_class", field_label: "Cleanroom Class",
--     field_type: "select", options: ["ISO 5", "ISO 6", "ISO 7", "ISO 8"] }
--
-- HVAC tenant:
--   { entity_type: "asset", field_key: "refrigerant_type", field_label: "Refrigerant Type",
--     field_type: "select", options: ["R-410A", "R-32", "R-134a", "R-744"] }
--   { entity_type: "asset", field_key: "charge_weight_kg", field_label: "Charge Weight (kg)",
--     field_type: "number", validation: { "min": 0, "max": 500 } }
--
-- Data Centre tenant:
--   { entity_type: "asset", field_key: "rack_position", field_label: "Rack Position",
--     field_type: "text" }
--   { entity_type: "asset", field_key: "power_draw_kw", field_label: "Power Draw (kW)",
--     field_type: "number" }
```

## User & Access Control

```sql
-- ============================================================
-- USER & RBAC (fully relational -- no JSONB for security data)
-- ============================================================

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    phone           VARCHAR(50),
    job_title       VARCHAR(100),
    preferences     JSONB NOT NULL DEFAULT '{}',
    -- preferences: { "language": "en", "timezone": "America/Chicago", "notifications": {...} }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(100) NOT NULL,
    permissions     JSONB NOT NULL DEFAULT '[]',
    -- permissions: ["work_order:create", "work_order:assign", "asset:read", "asset:edit", ...]
    is_system       BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

CREATE TABLE user_role (
    user_id         UUID NOT NULL REFERENCES app_user(id),
    role_id         UUID NOT NULL REFERENCES role(id),
    site_id         UUID REFERENCES site(id),
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, role_id, site_id)
);

CREATE TABLE team (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    site_id         UUID REFERENCES site(id),
    name            VARCHAR(255) NOT NULL,
    members         UUID[] NOT NULL DEFAULT '{}',   -- array of user IDs for fast lookup
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_user_tenant ON app_user(tenant_id);
```

## Location & Asset (Hybrid)

```sql
-- ============================================================
-- LOCATION HIERARCHY
-- ============================================================

CREATE TABLE location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    site_id         UUID NOT NULL REFERENCES site(id),
    parent_id       UUID REFERENCES location(id),
    code            VARCHAR(100) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    location_type   VARCHAR(50) NOT NULL,
    path            TEXT,                  -- materialised path
    depth           INTEGER NOT NULL DEFAULT 0,
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    -- Example for a hospital: { "department": "Radiology", "floor": 3, "wing": "East" }
    -- Example for a factory: { "production_line": "Line 4", "zone": "Clean" }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, site_id, code)
);

CREATE INDEX idx_location_parent ON location(parent_id);
CREATE INDEX idx_location_site ON location(site_id);
CREATE INDEX idx_location_custom ON location USING GIN (custom_fields);

-- ============================================================
-- ASSET (core relational + JSONB custom attributes)
-- ============================================================

CREATE TABLE asset (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    site_id         UUID NOT NULL REFERENCES site(id),
    location_id     UUID REFERENCES location(id),
    parent_asset_id UUID REFERENCES asset(id),

    -- Fixed relational columns (universal to all CMMS deployments)
    asset_tag       VARCHAR(100) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    equipment_class VARCHAR(50),          -- ISO 14224 class code
    serial_number   VARCHAR(100),
    model           VARCHAR(255),
    manufacturer    VARCHAR(255),
    barcode         VARCHAR(100),

    -- Lifecycle
    status          VARCHAR(30) NOT NULL DEFAULT 'active',
    criticality     VARCHAR(20) NOT NULL DEFAULT 'medium',
    install_date    DATE,
    warranty_expiry DATE,
    purchase_cost   NUMERIC(14, 2),
    currency_code   CHAR(3) DEFAULT 'USD',

    -- Metering
    has_meter       BOOLEAN NOT NULL DEFAULT false,
    meter_unit      VARCHAR(50),
    current_meter   NUMERIC(14, 2),
    last_meter_date TIMESTAMPTZ,

    -- JSONB: Industry-specific and tenant-custom attributes
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    -- Pharma example:
    --   { "gmp_qualified": true, "qualification_date": "2025-06-15",
    --     "clean_utility": "WFI", "validation_protocol": "IQ/OQ/PQ" }
    -- HVAC example:
    --   { "refrigerant_type": "R-410A", "charge_weight_kg": 12.5,
    --     "cooling_capacity_kw": 35, "energy_rating": "A++" }
    -- Manufacturing example:
    --   { "plc_model": "Siemens S7-1500", "opc_ua_node_id": "ns=2;s=Pump.P101",
    --     "rated_flow_m3h": 150, "max_pressure_bar": 16 }

    -- JSONB: Technical specifications (structured per equipment class)
    specifications  JSONB NOT NULL DEFAULT '{}',
    -- Pump: { "design_type": "centrifugal", "impeller_type": "closed",
    --         "rated_speed_rpm": 3000, "motor_power_kw": 75 }
    -- Valve: { "valve_type": "gate", "bore_mm": 150, "pressure_class": "300",
    --          "body_material": "CS", "trim_material": "SS316" }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, asset_tag)
);

CREATE INDEX idx_asset_tenant_site ON asset(tenant_id, site_id);
CREATE INDEX idx_asset_location ON asset(location_id);
CREATE INDEX idx_asset_parent ON asset(parent_asset_id);
CREATE INDEX idx_asset_status ON asset(status);
CREATE INDEX idx_asset_class ON asset(equipment_class);
CREATE INDEX idx_asset_custom ON asset USING GIN (custom_fields);
CREATE INDEX idx_asset_specs ON asset USING GIN (specifications);
```

## Work Order (Hybrid)

```sql
-- ============================================================
-- WORK ORDER (core relational + JSONB for custom fields)
-- ============================================================

CREATE TABLE work_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    site_id         UUID NOT NULL REFERENCES site(id),

    -- Fixed relational columns
    wo_number       VARCHAR(50) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    wo_type         VARCHAR(30) NOT NULL,  -- ISO 13306 aligned
    priority        VARCHAR(20) NOT NULL DEFAULT 'medium',
    status          VARCHAR(30) NOT NULL DEFAULT 'requested',

    asset_id        UUID REFERENCES asset(id),
    location_id     UUID REFERENCES location(id),
    parent_wo_id    UUID REFERENCES work_order(id),
    pm_schedule_id  UUID,

    assigned_to     UUID REFERENCES app_user(id),
    assigned_team_id UUID REFERENCES team(id),
    requested_by    UUID REFERENCES app_user(id),

    -- Failure analysis (fixed columns for ISO 14224 codes)
    failure_mode    VARCHAR(50),
    failure_cause   VARCHAR(50),
    failure_mechanism VARCHAR(50),
    detection_method VARCHAR(50),

    -- Scheduling
    scheduled_start TIMESTAMPTZ,
    scheduled_end   TIMESTAMPTZ,
    actual_start    TIMESTAMPTZ,
    actual_end      TIMESTAMPTZ,
    due_date        TIMESTAMPTZ,

    -- Cost
    estimated_hours NUMERIC(8, 2),
    actual_hours    NUMERIC(8, 2),
    estimated_cost  NUMERIC(14, 2),
    actual_cost     NUMERIC(14, 2),
    currency_code   CHAR(3) DEFAULT 'USD',

    -- Downtime
    downtime_minutes INTEGER,

    -- JSONB: tenant-specific custom fields
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    -- Pharma: { "gmp_batch_number": "B-2026-0542", "cleanroom_class": "ISO 7",
    --           "deviation_number": "DEV-2026-018" }
    -- Oil & Gas: { "permit_to_work": "PTW-2026-0089", "loto_required": true,
    --              "h2s_present": false, "hot_work_permit": null }
    -- Facilities: { "tenant_notified": true, "access_instructions": "Use side entrance",
    --               "building_manager": "John Smith" }

    -- JSONB: checklist results (for simple inspections embedded in WO)
    checklist_results JSONB,
    -- [
    --   { "seq": 1, "task": "Check oil level", "result": "pass", "notes": "Level OK" },
    --   { "seq": 2, "task": "Measure vibration", "result": "fail",
    --     "measurement": 12.5, "unit": "mm/s", "threshold": 11.0 }
    -- ]

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    UNIQUE(tenant_id, wo_number)
);

CREATE INDEX idx_wo_tenant_site ON work_order(tenant_id, site_id);
CREATE INDEX idx_wo_status ON work_order(status);
CREATE INDEX idx_wo_asset ON work_order(asset_id);
CREATE INDEX idx_wo_assigned ON work_order(assigned_to);
CREATE INDEX idx_wo_due ON work_order(due_date);
CREATE INDEX idx_wo_type ON work_order(wo_type);
CREATE INDEX idx_wo_custom ON work_order USING GIN (custom_fields);

-- ============================================================
-- WORK ORDER TASKS (for complex multi-step procedures)
-- ============================================================

CREATE TABLE work_order_task (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_id   UUID NOT NULL REFERENCES work_order(id) ON DELETE CASCADE,
    sequence        INTEGER NOT NULL,
    title           VARCHAR(500) NOT NULL,
    task_type       VARCHAR(30) DEFAULT 'action',
    is_required     BOOLEAN NOT NULL DEFAULT true,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    completed_by    UUID REFERENCES app_user(id),
    completed_at    TIMESTAMPTZ,
    result_data     JSONB,
    -- { "measurement": 12.5, "unit": "mm/s", "pass": false,
    --   "photo_ids": ["uuid1", "uuid2"], "notes": "Excessive vibration on drive end" }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_wo_task_wo ON work_order_task(work_order_id);

-- ============================================================
-- WORK ORDER ACTIVITY LOG (comments, labour, parts -- unified)
-- ============================================================

CREATE TABLE work_order_activity (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_id   UUID NOT NULL REFERENCES work_order(id) ON DELETE CASCADE,
    activity_type   VARCHAR(30) NOT NULL,
    -- comment, labour, part_used, status_change, assignment, attachment, signature
    user_id         UUID REFERENCES app_user(id),
    data            JSONB NOT NULL,
    -- comment:       { "text": "Seal replaced successfully", "is_internal": false }
    -- labour:        { "start": "...", "end": "...", "hours": 2.5, "type": "regular" }
    -- part_used:     { "part_id": "uuid", "part_number": "SEAL-101", "qty": 1, "cost": 485 }
    -- status_change: { "from": "in_progress", "to": "completed" }
    -- attachment:    { "file_name": "photo.jpg", "file_type": "image/jpeg",
    --                  "storage_path": "s3://...", "size_bytes": 245000 }
    -- signature:     { "meaning": "completed", "signer_name": "Jane Smith" }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_wo_activity_wo ON work_order_activity(work_order_id);
CREATE INDEX idx_wo_activity_type ON work_order_activity(activity_type);
```

## Preventive Maintenance

```sql
-- ============================================================
-- PREVENTIVE MAINTENANCE
-- ============================================================

CREATE TABLE pm_schedule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    site_id         UUID NOT NULL REFERENCES site(id),
    asset_id        UUID NOT NULL REFERENCES asset(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,

    trigger_type    VARCHAR(30) NOT NULL,
    frequency_value INTEGER,
    frequency_unit  VARCHAR(20),
    meter_interval  NUMERIC(14, 2),

    start_date      DATE NOT NULL,
    end_date        DATE,
    next_due_date   DATE,
    last_completed  TIMESTAMPTZ,

    -- Work order template
    wo_type         VARCHAR(30) NOT NULL DEFAULT 'preventive',
    priority        VARCHAR(20) DEFAULT 'medium',
    assigned_to     UUID REFERENCES app_user(id),
    estimated_hours NUMERIC(8, 2),

    -- Task template stored as JSONB (avoids separate template table)
    task_template   JSONB,
    -- [
    --   { "seq": 1, "title": "Drain old oil", "type": "action", "required": true },
    --   { "seq": 2, "title": "Replace filter", "type": "action", "required": true },
    --   { "seq": 3, "title": "Refill with specified oil", "type": "action", "required": true },
    --   { "seq": 4, "title": "Record oil volume (litres)", "type": "measurement",
    --     "unit": "litres", "min": 3.5, "max": 4.5, "required": true },
    --   { "seq": 5, "title": "Check for leaks after 10 min run", "type": "inspection",
    --     "required": true }
    -- ]

    custom_fields   JSONB NOT NULL DEFAULT '{}',

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

ALTER TABLE work_order
    ADD CONSTRAINT fk_wo_pm FOREIGN KEY (pm_schedule_id) REFERENCES pm_schedule(id);

CREATE INDEX idx_pm_tenant ON pm_schedule(tenant_id, site_id);
CREATE INDEX idx_pm_asset ON pm_schedule(asset_id);
CREATE INDEX idx_pm_next_due ON pm_schedule(next_due_date);
```

## Spare Parts & Inventory

```sql
-- ============================================================
-- SPARE PARTS & INVENTORY
-- ============================================================

CREATE TABLE part (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    part_number     VARCHAR(100) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    category        VARCHAR(100),
    unit_of_measure VARCHAR(30) DEFAULT 'each',
    unit_cost       NUMERIC(14, 2),
    currency_code   CHAR(3) DEFAULT 'USD',
    barcode         VARCHAR(100),
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    -- { "hazmat_class": "3", "shelf_life_months": 24, "storage_temp_max_c": 30 }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, part_number)
);

CREATE TABLE part_inventory (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    part_id         UUID NOT NULL REFERENCES part(id),
    site_id         UUID NOT NULL REFERENCES site(id),
    storeroom       VARCHAR(100),
    bin_location    VARCHAR(100),
    quantity_on_hand NUMERIC(14, 2) NOT NULL DEFAULT 0,
    min_quantity    NUMERIC(14, 2),
    max_quantity    NUMERIC(14, 2),
    reorder_point   NUMERIC(14, 2),
    reorder_qty     NUMERIC(14, 2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(part_id, site_id, storeroom)
);

CREATE INDEX idx_part_tenant ON part(tenant_id);
CREATE INDEX idx_part_inv_part ON part_inventory(part_id);
CREATE INDEX idx_part_inv_site ON part_inventory(site_id);
CREATE INDEX idx_part_custom ON part USING GIN (custom_fields);

-- BOM: parts needed for an asset
CREATE TABLE asset_bom (
    asset_id        UUID NOT NULL REFERENCES asset(id),
    part_id         UUID NOT NULL REFERENCES part(id),
    quantity        NUMERIC(14, 2) NOT NULL DEFAULT 1,
    is_critical     BOOLEAN NOT NULL DEFAULT false,
    PRIMARY KEY (asset_id, part_id)
);
```

## Vendor & Purchase Orders

```sql
-- ============================================================
-- VENDOR & PURCHASE ORDERS
-- ============================================================

CREATE TABLE vendor (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50),
    contact_info    JSONB NOT NULL DEFAULT '{}',
    -- { "name": "...", "email": "...", "phone": "...", "address": { ... } }
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
    currency_code   CHAR(3) DEFAULT 'USD',
    lines           JSONB NOT NULL DEFAULT '[]',
    -- [
    --   { "part_id": "uuid", "part_number": "SEAL-101", "qty": 5,
    --     "unit_cost": 485.00, "qty_received": 0 }
    -- ]
    total_amount    NUMERIC(14, 2),
    ordered_date    DATE,
    expected_date   DATE,
    created_by      UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, po_number)
);
```

## IoT / Sensor

```sql
-- ============================================================
-- IoT / SENSOR
-- ============================================================

CREATE TABLE sensor_device (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    asset_id        UUID REFERENCES asset(id),
    device_name     VARCHAR(255) NOT NULL,
    device_type     VARCHAR(100),
    protocol        VARCHAR(50),
    connection_config JSONB NOT NULL DEFAULT '{}',
    -- MQTT:   { "broker": "mqtt://...", "topic": "sensors/vibration/P101" }
    -- OPC UA: { "endpoint": "opc.tcp://...", "node_id": "ns=2;s=P101.Vibration" }
    thresholds      JSONB NOT NULL DEFAULT '[]',
    -- [
    --   { "type": "warning_high", "value": 7.0, "auto_wo": false },
    --   { "type": "alarm_high", "value": 11.0, "auto_wo": true, "priority": "high" }
    -- ]
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
    unit            VARCHAR(50),
    quality         VARCHAR(20) DEFAULT 'good',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_reading_device_time ON sensor_reading(sensor_device_id, reading_time DESC);
```

## Compliance & Audit

```sql
-- ============================================================
-- AUDIT TRAIL & COMPLIANCE (fully relational -- never JSONB for audit data)
-- ============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    action          VARCHAR(20) NOT NULL,
    changes         JSONB,               -- { "status": { "old": "in_progress", "new": "completed" } }
    user_id         UUID REFERENCES app_user(id),
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_tenant_time ON audit_log(tenant_id, created_at DESC);

CREATE TABLE electronic_signature (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    signer_id       UUID NOT NULL REFERENCES app_user(id),
    signature_meaning VARCHAR(100) NOT NULL,
    signed_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    ip_address      INET
);

CREATE INDEX idx_esig_entity ON electronic_signature(entity_type, entity_id);
```

## Example: Custom Field Query

```sql
-- "Find all assets with refrigerant type R-410A across all sites"
SELECT a.asset_tag, a.name, a.custom_fields->>'refrigerant_type' AS refrigerant,
       s.name AS site_name
FROM asset a
JOIN site s ON s.id = a.site_id
WHERE a.tenant_id = $1
  AND a.custom_fields @> '{"refrigerant_type": "R-410A"}'::jsonb;
-- The @> containment operator uses the GIN index on custom_fields

-- "Find all work orders with open PTW (Permit to Work) in Oil & Gas tenant"
SELECT wo.wo_number, wo.title, wo.custom_fields->>'permit_to_work' AS ptw
FROM work_order wo
WHERE wo.tenant_id = $1
  AND wo.status IN ('planned', 'scheduled', 'in_progress')
  AND wo.custom_fields ? 'permit_to_work';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant & Configuration | 3 | tenant, site, field_definition |
| User & RBAC | 4 | app_user, role, user_role, team |
| Location & Asset | 2 | location, asset |
| Work Order | 3 | work_order, work_order_task, work_order_activity |
| Preventive Maintenance | 1 | pm_schedule (task templates in JSONB) |
| Parts & Inventory | 3 | part, part_inventory, asset_bom |
| Vendor & Procurement | 2 | vendor, purchase_order (lines in JSONB) |
| IoT / Sensor | 2 | sensor_device, sensor_reading |
| Compliance & Audit | 2 | audit_log, electronic_signature |
| **Total** | **22** | ~40% fewer tables than normalized model; JSONB absorbs variability |

---

## Key Design Decisions

1. **JSONB `custom_fields` on every major entity.** Assets, work orders, locations, parts, and PM schedules all carry a `custom_fields` JSONB column. The `field_definition` table tells the UI what fields to render and how to validate input. This means a pharmaceutical tenant and an HVAC tenant use the same database schema but see completely different forms.

2. **GIN indexes on all JSONB columns.** PostgreSQL GIN indexes support the `@>` containment operator, `?` existence operator, and `->` path queries. This makes custom field filtering fast enough for production dashboards without needing to extract JSONB values into separate columns.

3. **Work order activity as a unified log.** Instead of separate tables for comments, labour, parts used, and attachments (as in the normalized model), a single `work_order_activity` table with an `activity_type` discriminator and JSONB `data` column handles all activity types. This simplifies the work order timeline view (one query, sorted by `created_at`) and reduces table count.

4. **PM task templates stored as JSONB arrays.** Rather than a separate `pm_task_template` table, the task checklist is stored directly on the PM schedule as a JSONB array. When a PM triggers a work order, the template is copied into `work_order_task` rows. This keeps the PM configuration self-contained and reduces joins.

5. **Purchase order lines stored as JSONB.** For a CMMS (not a full ERP), purchase order line items are a secondary concern. Storing them as a JSONB array on the PO record simplifies the schema. If procurement becomes a core feature, these can be "graduated" to a relational table.

6. **Sensor thresholds embedded in device record.** Rather than a separate `sensor_threshold` table, thresholds are stored as a JSONB array on the `sensor_device` record. A sensor typically has 2-4 thresholds, making a separate table unnecessary overhead.

7. **Compliance data stays fully relational.** Audit logs and electronic signatures are never stored in JSONB. These are the tables most likely to be scrutinised by regulators and auditors, and they must support precise SQL queries (e.g., "show me every change to this work order in the last 90 days"). Relational columns with proper indexes serve this purpose better than JSONB.

8. **Address and contact info in JSONB.** Site addresses and vendor contact information vary by country (different address formats, different fields). JSONB handles this naturally without needing a polymorphic address table.

9. **`specifications` separate from `custom_fields` on assets.** The asset table has two JSONB columns: `specifications` (technical design data from the manufacturer, rarely changes) and `custom_fields` (tenant-configured operational data). This separation makes it clear which data is "factory" and which is "field."

10. **Gradual migration path from JSONB to columns.** If a custom field becomes universal across all tenants (e.g., `energy_rating`), it can be "graduated" to a fixed relational column with a migration. The hybrid model explicitly supports this evolution path.
