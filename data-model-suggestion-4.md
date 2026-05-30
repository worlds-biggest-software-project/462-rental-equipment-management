# Data Model Suggestion 4: PostgreSQL + TimescaleDB + Apache AGE (Time-Series + Graph Hybrid)

> Project: Rental Equipment Management (462)
> Model Type: Domain-specific specialty — PostgreSQL with TimescaleDB for telematics time-series and Apache AGE for asset relationship graphs
> Generated: 2026-05-25

---

## Overview

This model uses PostgreSQL as the foundation but extends it with two specialized extensions that directly address the rental equipment management domain's hardest data challenges:

1. **TimescaleDB** for GPS telematics ingestion, hour-meter tracking, fuel consumption monitoring, and utilization analytics. Equipment fleets generate continuous streams of time-series data from Geotab, Samsara, and OEM telematics gateways. A standard PostgreSQL table with manual partitioning (as in Suggestions 1 and 3) works for small fleets but becomes a management burden at scale. TimescaleDB's hypertables, continuous aggregates, and compression policies handle this natively.

2. **Apache AGE** (A Graph Extension) for modeling the complex, many-to-many relationships between assets, depots, customers, contracts, maintenance histories, damage chains, and inter-depot transfers. Graph queries answer questions that are expensive or awkward in relational SQL: "Which customer has rented this asset before and what was the damage history?", "What is the complete lifecycle chain of this asset across depots, contracts, and maintenance events?", "Which assets at this depot are interchangeable substitutes for the one the customer requested?"

The core transactional data (contracts, invoices, payments) remains in standard PostgreSQL tables. TimescaleDB and Apache AGE are extensions -- not separate databases -- so all data lives in one PostgreSQL instance with full ACID guarantees, standard SQL access, and no inter-database synchronization.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    PostgreSQL 16+                            │
│                                                             │
│  ┌─────────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │ Standard Tables  │  │ TimescaleDB  │  │  Apache AGE   │  │
│  │                  │  │              │  │               │  │
│  │ - contracts      │  │ Hypertables: │  │ Graph:        │  │
│  │ - invoices       │  │ - telemetry  │  │ - Asset nodes │  │
│  │ - payments       │  │ - events     │  │ - edges for   │  │
│  │ - customers      │  │ - meter_log  │  │   rented_by,  │  │
│  │ - assets         │  │              │  │   maintained, │  │
│  │ - work_orders    │  │ Continuous   │  │   transferred │  │
│  │ - rate_tables    │  │ Aggregates:  │  │   damaged_on  │  │
│  │                  │  │ - hourly     │  │               │  │
│  │                  │  │ - daily      │  │               │  │
│  │                  │  │ - monthly    │  │               │  │
│  └─────────────────┘  └──────────────┘  └───────────────┘  │
│                                                             │
│  ┌─────────────────┐                                        │
│  │    PostGIS       │ (spatial queries, geofencing)          │
│  └─────────────────┘                                        │
└─────────────────────────────────────────────────────────────┘
```

---

## Part 1: Standard Relational Tables

The core transactional tables remain standard PostgreSQL tables. These are abbreviated here; the full definitions follow the same patterns as Suggestions 1 and 3.

```sql
-- Enable extensions
CREATE EXTENSION IF NOT EXISTS timescaledb;
CREATE EXTENSION IF NOT EXISTS age;
CREATE EXTENSION IF NOT EXISTS postgis;

-- Load the AGE extension
LOAD 'age';
SET search_path = ag_catalog, "$user", public;

-- ============================================================
-- STANDARD RELATIONAL TABLES (abbreviated)
-- ============================================================

CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    legal_name      VARCHAR(255),
    tax_id          VARCHAR(50),
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    timezone        VARCHAR(50) NOT NULL DEFAULT 'America/New_York',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE depots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(20) NOT NULL,
    address_line1   VARCHAR(255),
    city            VARCHAR(100),
    state_province  VARCHAR(100),
    postal_code     VARCHAR(20),
    country_code    CHAR(2) NOT NULL DEFAULT 'US',
    location        GEOGRAPHY(POINT, 4326),  -- PostGIS point
    phone           VARCHAR(30),
    email           VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, code)
);

CREATE INDEX idx_depots_location ON depots USING GIST (location);

CREATE TABLE equipment_categories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    parent_id       UUID REFERENCES equipment_categories(id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50) NOT NULL,
    description     TEXT,
    attribute_schema JSONB NOT NULL DEFAULT '[]',
    sort_order      INT NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, code)
);

CREATE TABLE assets (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    category_id         UUID NOT NULL REFERENCES equipment_categories(id),
    asset_number        VARCHAR(50) NOT NULL,
    serial_number       VARCHAR(100),
    vin                 VARCHAR(50),
    make                VARCHAR(100),
    model               VARCHAR(100),
    year                INT,
    description         TEXT,
    purchase_date       DATE,
    purchase_cost       DECIMAL(12, 2),
    current_value       DECIMAL(12, 2),
    fuel_type           VARCHAR(30),
    hour_meter_reading  DECIMAL(12, 1),
    odometer_reading    DECIMAL(12, 1),
    home_depot_id       UUID NOT NULL REFERENCES depots(id),
    current_depot_id    UUID NOT NULL REFERENCES depots(id),
    status              VARCHAR(30) NOT NULL DEFAULT 'available',
    condition_grade     VARCHAR(20) DEFAULT 'good',
    current_location    GEOGRAPHY(POINT, 4326),  -- PostGIS point
    last_location_at    TIMESTAMPTZ,
    specifications      JSONB NOT NULL DEFAULT '{}',
    compliance          JSONB NOT NULL DEFAULT '{}',
    is_active           BOOLEAN NOT NULL DEFAULT true,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, asset_number)
);

CREATE INDEX idx_assets_org_status ON assets(organization_id, status);
CREATE INDEX idx_assets_category ON assets(category_id);
CREATE INDEX idx_assets_location ON assets USING GIST (current_location);
CREATE INDEX idx_assets_specs ON assets USING GIN (specifications jsonb_path_ops);

CREATE TABLE customers (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    customer_number     VARCHAR(50) NOT NULL,
    customer_type       VARCHAR(20) NOT NULL DEFAULT 'business',
    company_name        VARCHAR(255),
    first_name          VARCHAR(100),
    last_name           VARCHAR(100),
    email               VARCHAR(255),
    phone               VARCHAR(30),
    credit_limit        DECIMAL(12, 2),
    payment_terms_days  INT NOT NULL DEFAULT 30,
    account_status      VARCHAR(20) NOT NULL DEFAULT 'active',
    billing_address     JSONB,
    contacts            JSONB NOT NULL DEFAULT '[]',
    sites               JSONB NOT NULL DEFAULT '[]',
    insurance_certificates JSONB NOT NULL DEFAULT '[]',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, customer_number)
);

CREATE TABLE rental_contracts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    contract_number VARCHAR(50) NOT NULL,
    customer_id     UUID NOT NULL REFERENCES customers(id),
    depot_id        UUID NOT NULL REFERENCES depots(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
    rental_start    TIMESTAMPTZ NOT NULL,
    rental_end_expected TIMESTAMPTZ,
    rental_end_actual   TIMESTAMPTZ,
    rate_table_id   UUID,
    deposit_amount  DECIMAL(12, 2) NOT NULL DEFAULT 0,
    subtotal        DECIMAL(12, 2) NOT NULL DEFAULT 0,
    tax_amount      DECIMAL(12, 2) NOT NULL DEFAULT 0,
    total           DECIMAL(12, 2) NOT NULL DEFAULT 0,
    delivery_details JSONB NOT NULL DEFAULT '{}',
    customer_snapshot JSONB NOT NULL DEFAULT '{}',
    signature_details JSONB,
    notes           TEXT,
    created_by      UUID NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, contract_number)
);

CREATE INDEX idx_contracts_org_status ON rental_contracts(organization_id, status);
CREATE INDEX idx_contracts_customer ON rental_contracts(customer_id);
CREATE INDEX idx_contracts_dates ON rental_contracts(rental_start, rental_end_expected);

CREATE TABLE contract_lines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id     UUID NOT NULL REFERENCES rental_contracts(id) ON DELETE CASCADE,
    asset_id        UUID NOT NULL REFERENCES assets(id),
    description     VARCHAR(500),
    period_type     VARCHAR(20) NOT NULL,
    rate_amount     DECIMAL(10, 2) NOT NULL,
    line_subtotal   DECIMAL(12, 2) NOT NULL DEFAULT 0,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    dispatched_at   TIMESTAMPTZ,
    returned_at     TIMESTAMPTZ,
    dispatch_readings JSONB,
    return_readings JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE inspections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_line_id UUID REFERENCES contract_lines(id),
    asset_id        UUID NOT NULL REFERENCES assets(id),
    inspection_type VARCHAR(20) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'in_progress',
    inspector_id    UUID NOT NULL,
    overall_condition VARCHAR(20),
    checklist       JSONB NOT NULL DEFAULT '[]',
    readings        JSONB NOT NULL DEFAULT '{}',
    photos          JSONB NOT NULL DEFAULT '[]',
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE damage_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inspection_id   UUID NOT NULL REFERENCES inspections(id),
    asset_id        UUID NOT NULL REFERENCES assets(id),
    contract_line_id UUID REFERENCES contract_lines(id),
    description     TEXT NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    estimated_repair_cost DECIMAL(10, 2),
    charge_to_customer    DECIMAL(10, 2),
    status          VARCHAR(20) NOT NULL DEFAULT 'reported',
    details         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE work_orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    work_order_number VARCHAR(50) NOT NULL,
    asset_id        UUID NOT NULL REFERENCES assets(id),
    depot_id        UUID NOT NULL REFERENCES depots(id),
    work_order_type VARCHAR(30) NOT NULL,
    priority        VARCHAR(20) NOT NULL DEFAULT 'normal',
    status          VARCHAR(20) NOT NULL DEFAULT 'open',
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    assigned_to     UUID,
    scheduled_start TIMESTAMPTZ,
    actual_start    TIMESTAMPTZ,
    actual_end      TIMESTAMPTZ,
    labor_cost      DECIMAL(10, 2) NOT NULL DEFAULT 0,
    parts_cost      DECIMAL(10, 2) NOT NULL DEFAULT 0,
    total_cost      DECIMAL(10, 2) NOT NULL DEFAULT 0,
    work_details    JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, work_order_number)
);

CREATE TABLE invoices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    invoice_number  VARCHAR(50) NOT NULL,
    contract_id     UUID NOT NULL REFERENCES rental_contracts(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    invoice_type    VARCHAR(20) NOT NULL DEFAULT 'rental',
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    invoice_date    DATE NOT NULL DEFAULT CURRENT_DATE,
    due_date        DATE NOT NULL,
    subtotal        DECIMAL(12, 2) NOT NULL DEFAULT 0,
    tax_amount      DECIMAL(12, 2) NOT NULL DEFAULT 0,
    total           DECIMAL(12, 2) NOT NULL DEFAULT 0,
    amount_paid     DECIMAL(12, 2) NOT NULL DEFAULT 0,
    balance_due     DECIMAL(12, 2) NOT NULL DEFAULT 0,
    accounting_sync JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, invoice_number)
);

CREATE TABLE payments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    invoice_id      UUID NOT NULL REFERENCES invoices(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    payment_method  VARCHAR(30) NOT NULL,
    amount          DECIMAL(12, 2) NOT NULL,
    payment_date    DATE NOT NULL DEFAULT CURRENT_DATE,
    status          VARCHAR(20) NOT NULL DEFAULT 'completed',
    gateway_details JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE depot_transfers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    asset_id        UUID NOT NULL REFERENCES assets(id),
    from_depot_id   UUID NOT NULL REFERENCES depots(id),
    to_depot_id     UUID NOT NULL REFERENCES depots(id),
    reason          VARCHAR(255),
    contract_id     UUID REFERENCES rental_contracts(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'requested',
    departed_at     TIMESTAMPTZ,
    arrived_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Part 2: TimescaleDB Hypertables for Telematics

TimescaleDB converts standard PostgreSQL tables into "hypertables" that automatically partition by time, compress old data, and provide continuous aggregates for real-time analytics.

### Telematics Data Ingestion

```sql
-- ============================================================
-- TIMESCALEDB HYPERTABLES
-- ============================================================

-- Primary telematics readings from GPS/IoT devices
CREATE TABLE telemetry (
    time            TIMESTAMPTZ NOT NULL,
    asset_id        UUID NOT NULL,
    provider_id     UUID NOT NULL,
    -- Position
    latitude        DOUBLE PRECISION,
    longitude       DOUBLE PRECISION,
    altitude_m      DOUBLE PRECISION,
    speed_kmh       DOUBLE PRECISION,
    heading         DOUBLE PRECISION,
    -- Engine and fuel
    engine_hours    DOUBLE PRECISION,
    odometer_km     DOUBLE PRECISION,
    fuel_level_pct  DOUBLE PRECISION,
    fuel_consumed_l DOUBLE PRECISION,
    engine_status   SMALLINT,  -- 0=off, 1=idle, 2=running
    -- Electrical
    battery_voltage DOUBLE PRECISION,
    -- Diagnostics
    dtc_codes       TEXT[],  -- diagnostic trouble codes
    -- Quality
    gps_accuracy_m  DOUBLE PRECISION,
    satellite_count SMALLINT
);

-- Convert to hypertable: auto-partitioned by time
SELECT create_hypertable('telemetry', by_range('time'));

-- Create indexes optimized for time-series queries
CREATE INDEX idx_telemetry_asset_time ON telemetry (asset_id, time DESC);

-- Enable columnar compression for data older than 7 days
-- Compression dramatically reduces storage (typically 10-20x)
ALTER TABLE telemetry SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'asset_id',
    timescaledb.compress_orderby = 'time DESC'
);

-- Auto-compress chunks older than 7 days
SELECT add_compression_policy('telemetry', INTERVAL '7 days');

-- Auto-drop raw data older than 13 months (keep aggregates longer)
SELECT add_retention_policy('telemetry', INTERVAL '13 months');


-- Equipment event log (state transitions, alerts, geofence events)
CREATE TABLE equipment_events (
    time            TIMESTAMPTZ NOT NULL,
    asset_id        UUID NOT NULL,
    event_type      VARCHAR(50) NOT NULL,
        -- engine_start, engine_stop, movement_start, movement_stop,
        -- geofence_enter, geofence_exit, idle_threshold,
        -- harsh_braking, excessive_speed, tow_detected,
        -- low_fuel, low_battery, dtc_triggered, dtc_cleared
    event_data      JSONB NOT NULL DEFAULT '{}',
    /*  Examples:
        engine_start:  {"hour_meter": 4589.7, "location": [33.49, -112.07]}
        geofence_exit: {"geofence_id": "uuid", "name": "Phoenix Site", "location": [33.51, -112.10]}
        dtc_triggered: {"code": "P0300", "description": "Random/Multiple Cylinder Misfire Detected"}
    */
    latitude        DOUBLE PRECISION,
    longitude       DOUBLE PRECISION
);

SELECT create_hypertable('equipment_events', by_range('time'));
CREATE INDEX idx_eq_events_asset_time ON equipment_events (asset_id, time DESC);
CREATE INDEX idx_eq_events_type ON equipment_events (event_type, time DESC);

ALTER TABLE equipment_events SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'asset_id',
    timescaledb.compress_orderby = 'time DESC'
);
SELECT add_compression_policy('equipment_events', INTERVAL '30 days');
SELECT add_retention_policy('equipment_events', INTERVAL '36 months');


-- Meter reading log: tracks cumulative hour meter and odometer
-- Used for maintenance scheduling thresholds
CREATE TABLE meter_readings (
    time            TIMESTAMPTZ NOT NULL,
    asset_id        UUID NOT NULL,
    reading_type    VARCHAR(20) NOT NULL,  -- engine_hours, odometer
    value           DOUBLE PRECISION NOT NULL,
    source          VARCHAR(20) NOT NULL,  -- telematics, manual
    delta           DOUBLE PRECISION  -- change since previous reading
);

SELECT create_hypertable('meter_readings', by_range('time'));
CREATE INDEX idx_meter_asset_type ON meter_readings (asset_id, reading_type, time DESC);

ALTER TABLE meter_readings SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'asset_id,reading_type',
    timescaledb.compress_orderby = 'time DESC'
);
SELECT add_compression_policy('meter_readings', INTERVAL '7 days');
```

### Continuous Aggregates for Analytics

```sql
-- ============================================================
-- CONTINUOUS AGGREGATES (auto-updating materialized views)
-- ============================================================

-- Hourly utilization per asset
CREATE MATERIALIZED VIEW telemetry_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS bucket,
    asset_id,
    -- Position: last known location in the hour
    last(latitude, time) AS last_latitude,
    last(longitude, time) AS last_longitude,
    -- Engine hours: delta within the hour
    max(engine_hours) - min(engine_hours) AS hours_operated,
    -- Fuel: consumption within the hour
    max(fuel_consumed_l) - min(fuel_consumed_l) AS fuel_consumed_l,
    last(fuel_level_pct, time) AS fuel_level_pct,
    -- Movement
    avg(speed_kmh) FILTER (WHERE speed_kmh > 0) AS avg_speed_kmh,
    max(speed_kmh) AS max_speed_kmh,
    -- Engine state distribution
    count(*) FILTER (WHERE engine_status = 2) AS running_readings,
    count(*) FILTER (WHERE engine_status = 1) AS idle_readings,
    count(*) FILTER (WHERE engine_status = 0) AS off_readings,
    count(*) AS total_readings,
    -- Odometer
    max(odometer_km) - min(odometer_km) AS distance_km
FROM telemetry
GROUP BY time_bucket('1 hour', time), asset_id
WITH NO DATA;

-- Refresh policy: update every hour, looking back 3 hours for late data
SELECT add_continuous_aggregate_policy('telemetry_hourly',
    start_offset => INTERVAL '3 hours',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour'
);

-- Daily utilization per asset
CREATE MATERIALIZED VIEW telemetry_daily
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', bucket) AS day,
    asset_id,
    last(last_latitude, bucket) AS last_latitude,
    last(last_longitude, bucket) AS last_longitude,
    sum(hours_operated) AS hours_operated,
    sum(fuel_consumed_l) AS fuel_consumed_l,
    last(fuel_level_pct, bucket) AS fuel_level_pct,
    avg(avg_speed_kmh) FILTER (WHERE avg_speed_kmh IS NOT NULL) AS avg_speed_kmh,
    max(max_speed_kmh) AS max_speed_kmh,
    sum(running_readings) AS running_readings,
    sum(idle_readings) AS idle_readings,
    sum(off_readings) AS off_readings,
    sum(total_readings) AS total_readings,
    sum(distance_km) AS distance_km,
    -- Utilization: hours operated / 24
    sum(hours_operated) / 24.0 AS utilization_ratio
FROM telemetry_hourly
GROUP BY time_bucket('1 day', bucket), asset_id
WITH NO DATA;

SELECT add_continuous_aggregate_policy('telemetry_daily',
    start_offset => INTERVAL '3 days',
    end_offset => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 day'
);

-- Monthly fleet summary (for fleet utilization dashboards)
CREATE MATERIALIZED VIEW fleet_monthly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 month', day) AS month,
    asset_id,
    sum(hours_operated) AS total_hours,
    sum(fuel_consumed_l) AS total_fuel_l,
    sum(distance_km) AS total_distance_km,
    avg(utilization_ratio) AS avg_utilization,
    count(*) FILTER (WHERE hours_operated > 0) AS days_active,
    count(*) AS days_in_period
FROM telemetry_daily
GROUP BY time_bucket('1 month', day), asset_id
WITH NO DATA;

SELECT add_continuous_aggregate_policy('fleet_monthly',
    start_offset => INTERVAL '2 months',
    end_offset => INTERVAL '1 month',
    schedule_interval => INTERVAL '1 day'
);
```

### Geofence Monitoring

```sql
-- Geofences: boundaries where assets should or should not be
CREATE TABLE geofences (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    fence_type      VARCHAR(20) NOT NULL,
        -- depot, customer_site, restricted_zone, custom
    boundary        GEOGRAPHY(POLYGON, 4326) NOT NULL,  -- PostGIS polygon
    reference_type  VARCHAR(50),  -- depot, customer_site
    reference_id    UUID,
    alert_on_exit   BOOLEAN NOT NULL DEFAULT true,
    alert_on_enter  BOOLEAN NOT NULL DEFAULT false,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_geofences_boundary ON geofences USING GIST (boundary);

-- Geofence check: compare latest telemetry position against fences
-- This query runs periodically or on each telemetry reading
-- PostGIS ST_Contains checks if a GPS point is within a polygon
--
-- SELECT g.id, g.name, g.fence_type
-- FROM geofences g
-- WHERE g.organization_id = 'org-uuid'
-- AND g.is_active = true
-- AND ST_Contains(
--     g.boundary::geometry,
--     ST_SetSRID(ST_MakePoint(-112.074, 33.494), 4326)
-- );
```

### Maintenance Threshold Monitoring

```sql
-- Maintenance thresholds: when to trigger work orders
CREATE TABLE maintenance_thresholds (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    category_id     UUID REFERENCES equipment_categories(id),
    asset_id        UUID REFERENCES assets(id),
    threshold_type  VARCHAR(20) NOT NULL,
        -- engine_hours, odometer, calendar, fuel_consumption
    interval_value  DOUBLE PRECISION NOT NULL,  -- e.g. 250 (hours)
    warning_at_pct  DOUBLE PRECISION DEFAULT 0.9,  -- alert at 90% of interval
    description     TEXT,
    tasks           JSONB NOT NULL DEFAULT '[]',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Query: find assets approaching maintenance threshold
-- Uses TimescaleDB last() function on meter_readings hypertable
--
-- SELECT a.asset_number, a.make, a.model,
--        last(mr.value, mr.time) AS current_hours,
--        mt.interval_value,
--        last(mr.value, mr.time)::numeric % mt.interval_value::numeric AS hours_since_last_service,
--        mt.interval_value - (last(mr.value, mr.time)::numeric % mt.interval_value::numeric) AS hours_until_service
-- FROM assets a
-- JOIN maintenance_thresholds mt ON (mt.category_id = a.category_id OR mt.asset_id = a.id)
-- JOIN meter_readings mr ON mr.asset_id = a.id AND mr.reading_type = 'engine_hours'
-- WHERE a.organization_id = 'org-uuid'
-- AND mt.is_active = true
-- GROUP BY a.id, a.asset_number, a.make, a.model, mt.interval_value
-- HAVING mt.interval_value - (last(mr.value, mr.time)::numeric % mt.interval_value::numeric) 
--        < mt.interval_value * (1 - mt.warning_at_pct);
```

---

## Part 3: Apache AGE Graph Model

Apache AGE adds graph database capabilities to PostgreSQL using openCypher query syntax. The graph model captures the web of relationships between assets, customers, depots, contracts, and maintenance events that are expensive to traverse with relational JOINs.

### Graph Schema

```sql
-- Create the equipment rental graph
SELECT create_graph('rental_graph');

-- ============================================================
-- VERTEX (NODE) TYPES
-- ============================================================

-- Asset vertices (synced from assets table)
-- Cypher:
-- CREATE (:Asset {
--     id: 'uuid',
--     asset_number: 'EX-042',
--     make: 'Caterpillar',
--     model: '320',
--     year: 2024,
--     category: 'Excavators',
--     status: 'available',
--     purchase_cost: 235000,
--     hour_meter: 4589.7
-- })

-- Customer vertices
-- CREATE (:Customer {
--     id: 'uuid',
--     customer_number: 'CUST-001',
--     name: 'Acme Construction LLC',
--     type: 'business',
--     account_status: 'active',
--     credit_limit: 50000
-- })

-- Depot vertices
-- CREATE (:Depot {
--     id: 'uuid',
--     name: 'Phoenix Main Depot',
--     code: 'PHX-01',
--     city: 'Phoenix',
--     state: 'AZ'
-- })

-- Contract vertices
-- CREATE (:Contract {
--     id: 'uuid',
--     contract_number: 'RC-2026-0142',
--     status: 'active',
--     rental_start: '2026-06-01',
--     rental_end: '2026-06-15',
--     total: 4250.00
-- })

-- WorkOrder vertices
-- CREATE (:WorkOrder {
--     id: 'uuid',
--     work_order_number: 'WO-2026-0089',
--     type: 'preventive',
--     status: 'completed',
--     total_cost: 450.00,
--     completed_at: '2026-05-20'
-- })

-- Inspection vertices
-- CREATE (:Inspection {
--     id: 'uuid',
--     type: 'post_rental',
--     condition: 'fair',
--     inspector: 'Dave Wilson',
--     date: '2026-05-28'
-- })

-- DamageRecord vertices
-- CREATE (:DamageRecord {
--     id: 'uuid',
--     description: 'Scratch on right panel',
--     severity: 'minor',
--     repair_cost: 380.00,
--     charge_to_customer: 450.00,
--     status: 'repaired'
-- })

-- Category vertices (for substitution queries)
-- CREATE (:Category {
--     id: 'uuid',
--     name: 'Mini Excavators',
--     code: 'mini_excavators',
--     parent_code: 'excavators'
-- })
```

### Edge (Relationship) Types

```sql
-- ============================================================
-- EDGE (RELATIONSHIP) TYPES
-- ============================================================

-- Asset located at Depot
-- (:Asset)-[:LOCATED_AT {since: '2026-05-01'}]->(:Depot)

-- Asset's home depot
-- (:Asset)-[:HOME_DEPOT]->(:Depot)

-- Asset belongs to Category
-- (:Asset)-[:BELONGS_TO]->(:Category)

-- Asset rented via Contract
-- (:Asset)-[:RENTED_ON {
--     dispatched_at: '2026-06-01',
--     returned_at: '2026-06-14',
--     hours_out: 4521.3,
--     hours_in: 4589.7,
--     hours_used: 68.4,
--     revenue: 2125.00
-- }]->(:Contract)

-- Customer placed Contract
-- (:Customer)-[:PLACED {
--     signed_at: '2026-05-28',
--     payment_status: 'paid'
-- }]->(:Contract)

-- Customer previously rented Asset (aggregated)
-- (:Customer)-[:HAS_RENTED {
--     times: 3,
--     total_revenue: 8750.00,
--     total_hours: 245.5,
--     first_rental: '2025-03-15',
--     last_rental: '2026-06-14',
--     damage_incidents: 1
-- }]->(:Asset)

-- Asset transferred between Depots
-- (:Asset)-[:TRANSFERRED {
--     transfer_id: 'uuid',
--     departed_at: '2026-04-15',
--     arrived_at: '2026-04-16',
--     reason: 'customer_demand'
-- }]->(:Depot)
-- (directional: from source depot to destination depot via asset)

-- Asset inspected in Inspection
-- (:Asset)-[:INSPECTED_IN {
--     condition_before: 'good',
--     condition_after: 'fair'
-- }]->(:Inspection)

-- Inspection found Damage
-- (:Inspection)-[:FOUND_DAMAGE]->(:DamageRecord)

-- Damage repaired by WorkOrder
-- (:DamageRecord)-[:REPAIRED_BY]->(:WorkOrder)

-- Asset maintained by WorkOrder
-- (:Asset)-[:MAINTAINED_BY {
--     type: 'preventive',
--     hour_meter_at: 4500.0,
--     cost: 450.00
-- }]->(:WorkOrder)

-- Contract raised from Depot
-- (:Contract)-[:ORIGINATED_FROM]->(:Depot)

-- Category parent hierarchy
-- (:Category)-[:SUBCATEGORY_OF]->(:Category)

-- Assets that can substitute for each other
-- (:Asset)-[:CAN_SUBSTITUTE {
--     compatibility: 0.95,
--     reason: 'same_category_similar_specs'
-- }]->(:Asset)
```

### Graph Sync Process

The graph is kept in sync with the relational tables using triggers or a background sync process.

```sql
-- Example: trigger to sync asset creation to graph
CREATE OR REPLACE FUNCTION sync_asset_to_graph()
RETURNS TRIGGER AS $$
DECLARE
    cat_name TEXT;
BEGIN
    SELECT name INTO cat_name FROM equipment_categories WHERE id = NEW.category_id;

    -- Create asset vertex
    EXECUTE format(
        'SELECT * FROM cypher(''rental_graph'', $$
            MERGE (a:Asset {id: %L})
            SET a.asset_number = %L,
                a.make = %L,
                a.model = %L,
                a.year = %s,
                a.category = %L,
                a.status = %L,
                a.purchase_cost = %s,
                a.hour_meter = %s
        $$) AS (result agtype)',
        NEW.id, NEW.asset_number, NEW.make, NEW.model,
        COALESCE(NEW.year::text, 'null'),
        cat_name, NEW.status,
        COALESCE(NEW.purchase_cost::text, '0'),
        COALESCE(NEW.hour_meter_reading::text, '0')
    );

    -- Create LOCATED_AT edge to current depot
    EXECUTE format(
        'SELECT * FROM cypher(''rental_graph'', $$
            MATCH (a:Asset {id: %L}), (d:Depot {id: %L})
            MERGE (a)-[:LOCATED_AT {since: %L}]->(d)
        $$) AS (result agtype)',
        NEW.id, NEW.current_depot_id, now()::text
    );

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_asset_graph_sync
    AFTER INSERT OR UPDATE ON assets
    FOR EACH ROW EXECUTE FUNCTION sync_asset_to_graph();

-- Similar triggers for customers, contracts, inspections, etc.
```

### Graph Query Examples

These queries demonstrate the power of graph traversal for rental-specific questions that would require complex multi-table JOINs in pure SQL.

```sql
-- ============================================================
-- GRAPH QUERIES (openCypher via Apache AGE)
-- ============================================================

-- 1. COMPLETE ASSET LIFECYCLE: Show all events for an asset
--    (rentals, maintenance, transfers, damage) in chronological order
SELECT * FROM cypher('rental_graph', $$
    MATCH (a:Asset {asset_number: 'EX-042'})-[r]->(target)
    RETURN type(r) AS relationship,
           properties(r) AS details,
           labels(target) AS target_type,
           properties(target) AS target_info
    ORDER BY COALESCE(
        r.dispatched_at, r.departed_at, r.completed_at,
        r.date, r.since
    )
$$) AS (relationship agtype, details agtype, target_type agtype, target_info agtype);

-- 2. CUSTOMER RENTAL HISTORY WITH DAMAGE: Which assets has this
--    customer rented, and what damage occurred?
SELECT * FROM cypher('rental_graph', $$
    MATCH (c:Customer {customer_number: 'CUST-001'})-[:PLACED]->(contract:Contract)
          <-[:RENTED_ON]-(asset:Asset)
    OPTIONAL MATCH (asset)-[:INSPECTED_IN]->(insp:Inspection)
                   -[:FOUND_DAMAGE]->(damage:DamageRecord)
    WHERE insp.type = 'post_rental'
    RETURN asset.asset_number, asset.make, asset.model,
           contract.contract_number, contract.rental_start,
           damage.description, damage.severity, damage.charge_to_customer
    ORDER BY contract.rental_start DESC
$$) AS (asset_num agtype, make agtype, model agtype,
        contract agtype, start_date agtype,
        damage_desc agtype, severity agtype, charge agtype);

-- 3. FIND SUBSTITUTE EQUIPMENT: Customer wants a CAT 320 but it's
--    unavailable. Find similar available assets at nearby depots.
SELECT * FROM cypher('rental_graph', $$
    MATCH (requested:Asset {asset_number: 'EX-042'})-[:BELONGS_TO]->(cat:Category)
          <-[:BELONGS_TO]-(substitute:Asset)-[:LOCATED_AT]->(depot:Depot)
    WHERE substitute.status = 'available'
    AND substitute.id <> requested.id
    RETURN substitute.asset_number, substitute.make, substitute.model,
           substitute.year, substitute.hour_meter,
           depot.name AS depot_name, depot.city
    ORDER BY substitute.year DESC
$$) AS (asset_num agtype, make agtype, model agtype,
        year agtype, hours agtype, depot agtype, city agtype);

-- 4. DAMAGE CHAIN ANALYSIS: Trace the full lifecycle of a damage
--    incident from detection through repair and billing
SELECT * FROM cypher('rental_graph', $$
    MATCH (asset:Asset)-[:INSPECTED_IN]->(insp:Inspection)
          -[:FOUND_DAMAGE]->(damage:DamageRecord)
          -[:REPAIRED_BY]->(wo:WorkOrder)
    MATCH (asset)-[:RENTED_ON]->(contract:Contract)
          <-[:PLACED]-(customer:Customer)
    WHERE damage.status = 'repaired'
    RETURN customer.name, contract.contract_number,
           asset.asset_number, damage.description,
           damage.severity, damage.repair_cost,
           damage.charge_to_customer,
           wo.work_order_number, wo.total_cost
$$) AS (customer agtype, contract agtype, asset agtype,
        damage agtype, severity agtype, repair_cost agtype,
        charge agtype, wo_number agtype, wo_cost agtype);

-- 5. FLEET MOVEMENT ANALYSIS: Show all inter-depot transfers
--    for a time period to identify demand patterns
SELECT * FROM cypher('rental_graph', $$
    MATCH (asset:Asset)-[t:TRANSFERRED]->(depot:Depot)
    WHERE t.departed_at >= '2026-01-01' AND t.departed_at < '2026-07-01'
    RETURN depot.name AS destination,
           count(t) AS transfer_count,
           collect(DISTINCT asset.category) AS equipment_types,
           collect(t.reason) AS reasons
    ORDER BY transfer_count DESC
$$) AS (destination agtype, count agtype, types agtype, reasons agtype);

-- 6. CUSTOMER RISK SCORING: Identify customers with repeated
--    damage across multiple rentals
SELECT * FROM cypher('rental_graph', $$
    MATCH (c:Customer)-[:PLACED]->(contract:Contract)
          <-[:RENTED_ON]-(asset:Asset)-[:INSPECTED_IN]->(insp:Inspection)
          -[:FOUND_DAMAGE]->(damage:DamageRecord)
    WITH c, count(DISTINCT damage) AS damage_count,
         sum(damage.charge_to_customer) AS total_charges,
         count(DISTINCT contract) AS contract_count
    WHERE damage_count > 1
    RETURN c.name, c.customer_number,
           contract_count, damage_count, total_charges,
           toFloat(damage_count) / contract_count AS damage_rate
    ORDER BY damage_rate DESC
$$) AS (name agtype, cust_num agtype, contracts agtype,
        damages agtype, charges agtype, rate agtype);

-- 7. MAINTENANCE PATTERN ANALYSIS: Find assets with escalating
--    maintenance costs (each work order more expensive than the last)
SELECT * FROM cypher('rental_graph', $$
    MATCH (a:Asset)-[m:MAINTAINED_BY]->(wo:WorkOrder)
    WITH a, wo ORDER BY wo.completed_at
    WITH a, collect(wo.total_cost) AS costs,
            collect(wo.work_order_number) AS wo_numbers
    WHERE size(costs) >= 3
    RETURN a.asset_number, a.make, a.model, a.hour_meter,
           wo_numbers, costs
$$) AS (asset agtype, make agtype, model agtype,
        hours agtype, wo_numbers agtype, costs agtype);
```

---

## Part 4: Combining All Three in Queries

The power of this architecture is that standard SQL, TimescaleDB functions, and Cypher queries can be combined in the same PostgreSQL session.

```sql
-- Combined query: Asset utilization from TimescaleDB + rental history from graph

-- Step 1: Get monthly utilization from TimescaleDB continuous aggregate
WITH monthly_usage AS (
    SELECT
        asset_id,
        month,
        total_hours,
        total_fuel_l,
        avg_utilization,
        days_active
    FROM fleet_monthly
    WHERE asset_id = 'asset-uuid'
    AND month >= '2026-01-01'
    ORDER BY month
),
-- Step 2: Get asset details from relational table
asset_info AS (
    SELECT id, asset_number, make, model, year,
           purchase_cost, current_value, hour_meter_reading
    FROM assets
    WHERE id = 'asset-uuid'
)
SELECT
    ai.asset_number, ai.make, ai.model,
    mu.month,
    mu.total_hours,
    mu.total_fuel_l,
    mu.avg_utilization,
    mu.days_active
FROM asset_info ai
CROSS JOIN monthly_usage mu;

-- Step 3 (separate query): Get rental and maintenance graph context
SELECT * FROM cypher('rental_graph', $$
    MATCH (a:Asset {id: 'asset-uuid'})
    OPTIONAL MATCH (a)-[r:RENTED_ON]->(c:Contract)<-[:PLACED]-(cust:Customer)
    OPTIONAL MATCH (a)-[m:MAINTAINED_BY]->(wo:WorkOrder)
    RETURN a.asset_number,
           count(DISTINCT c) AS total_rentals,
           sum(r.revenue) AS total_revenue,
           count(DISTINCT wo) AS total_work_orders,
           sum(wo.total_cost) AS total_maintenance_cost
$$) AS (asset agtype, rentals agtype, revenue agtype,
        work_orders agtype, maintenance agtype);


-- Predictive maintenance: combine meter readings trend with maintenance graph
-- Step 1: Meter reading velocity from TimescaleDB
WITH hourly_rate AS (
    SELECT
        asset_id,
        time_bucket('1 week', time) AS week,
        (max(value) - min(value)) / GREATEST(
            EXTRACT(EPOCH FROM (max(time) - min(time))) / 3600, 1
        ) AS hours_per_hour  -- how fast the meter is running
    FROM meter_readings
    WHERE reading_type = 'engine_hours'
    AND asset_id = 'asset-uuid'
    AND time > now() - INTERVAL '3 months'
    GROUP BY asset_id, time_bucket('1 week', time)
),
avg_rate AS (
    SELECT asset_id, avg(hours_per_hour) AS avg_rate
    FROM hourly_rate
    GROUP BY asset_id
)
SELECT
    a.asset_number,
    a.hour_meter_reading AS current_hours,
    ar.avg_rate AS avg_hourly_rate,
    mt.interval_value AS service_interval_hours,
    a.hour_meter_reading::numeric % mt.interval_value::numeric AS hours_since_service,
    (mt.interval_value - (a.hour_meter_reading::numeric % mt.interval_value::numeric))
        / GREATEST(ar.avg_rate * 24, 0.1) AS estimated_days_until_service
FROM assets a
JOIN maintenance_thresholds mt ON mt.category_id = a.category_id
    AND mt.threshold_type = 'engine_hours'
JOIN avg_rate ar ON ar.asset_id = a.id
WHERE a.id = 'asset-uuid';
```

---

## Pros and Cons

### Pros

1. **Time-series data is handled by purpose-built infrastructure.** TimescaleDB's hypertables, automatic chunking, columnar compression, and continuous aggregates are specifically designed for the telematics data that equipment rental platforms must ingest. A fleet of 500 assets reporting GPS/hours/fuel every 30 seconds generates ~525 million rows per year. TimescaleDB handles this with 10-20x compression and sub-second query performance on aggregates, without manual partition management.

2. **Graph queries answer critical business questions efficiently.** "What is this asset's complete lifecycle?", "Which customers have a pattern of damage?", "What equipment at nearby depots can substitute for this unavailable item?" -- these queries traverse 3-6 levels of relationships and would require 4-8 table JOINs in relational SQL. In openCypher, they are 3-5 lines of readable, maintainable code.

3. **Single database engine.** TimescaleDB and Apache AGE are PostgreSQL extensions, not separate databases. All data lives in one PostgreSQL instance with one connection string, one backup process, one monitoring setup, and full ACID transactions across all three storage paradigms. This eliminates the synchronization complexity of a polyglot persistence architecture.

4. **PostGIS integration enables spatial queries.** Nearest-depot lookups, delivery route optimization, geofence monitoring, and asset location heat maps all use PostGIS spatial functions on the same database. The `GEOGRAPHY(POINT, 4326)` type on assets and depots enables `ST_DWithin()`, `ST_Distance()`, and `ST_Contains()` queries natively.

5. **Continuous aggregates provide real-time dashboards without ETL.** Fleet utilization dashboards, fuel consumption reports, and maintenance forecasts draw from continuously updated materialized views. No separate data pipeline or warehouse is needed for operational analytics.

6. **Compression and retention policies automate data lifecycle.** Raw telematics data is compressed after 7 days and dropped after 13 months. Aggregates (hourly, daily, monthly) are retained for years. This is configured once and runs automatically, eliminating manual data lifecycle management.

7. **AI/ML training data is readily available.** Demand forecasting and predictive maintenance models need historical time-series data joined with business context (which customer, what contract, what condition). TimescaleDB continuous aggregates + relational tables + graph relationships provide all three in one queryable system.

### Cons

1. **Three query paradigms to learn and maintain.** Developers must be comfortable with standard SQL, TimescaleDB-specific functions (`time_bucket()`, `last()`, continuous aggregate DDL), and openCypher graph queries. This is a significant training investment and limits the hiring pool.

2. **Apache AGE is less mature than core PostgreSQL.** Apache AGE graduated from incubation in 2024, but its ecosystem (tooling, monitoring, query optimization) is less mature than PostgreSQL's relational engine. Complex graph queries may have unpredictable performance characteristics, and query plan optimization is less sophisticated than PostgreSQL's relational query planner.

3. **Graph sync adds operational complexity.** The graph must be kept in sync with relational tables. Triggers (shown above) add overhead to every write operation. A background sync process introduces eventual consistency. Either approach adds failure modes and monitoring requirements.

4. **TimescaleDB licensing considerations.** TimescaleDB Community Edition is open-source (Timescale License), but certain features (continuous aggregates on top of continuous aggregates, some compression features) require the paid Timescale platform. The community edition covers the use cases described here, but teams should verify feature availability for their specific needs.

5. **Hosting constraints.** Not all managed PostgreSQL services support TimescaleDB and Apache AGE extensions. AWS RDS supports neither natively (requires custom AMI or Timescale Cloud). Azure supports AGE but not TimescaleDB as a built-in extension. Self-hosted or Timescale Cloud may be required, limiting deployment flexibility.

6. **Debugging complexity when issues span paradigms.** A bug that manifests as incorrect fleet utilization data could originate in the telematics ingestion pipeline (TimescaleDB), the graph sync (Apache AGE), or the relational contract data. Tracing across three query paradigms requires broader expertise.

7. **Over-engineered for initial deployment.** A rental operator with 50 assets and no telematics integration does not need TimescaleDB hypertables or a graph database. These capabilities become valuable at 200+ telematics-tracked assets and when relationship-based queries (customer risk, asset lifecycle, substitution) are product priorities.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Database | PostgreSQL 16+ |
| Extensions | TimescaleDB 2.x, Apache AGE 1.5+, PostGIS 3.x, pgcrypto |
| Telematics ingestion | Kafka/NATS -> worker -> TimescaleDB INSERT |
| Graph sync | PostgreSQL triggers (simple) or Change Data Capture (scale) |
| Application ORM | Prisma or Drizzle for relational; raw SQL for TimescaleDB; AGE client library for Cypher |
| Hosting (recommended) | Timescale Cloud (TimescaleDB managed) + Apache AGE compiled from source |
| Hosting (alternative) | Self-hosted PostgreSQL on Linux with all extensions |
| Monitoring | pg_stat_statements, TimescaleDB chunk monitoring, AGE graph statistics |
| Backup | pg_dump (includes all extensions), WAL archiving |

---

## Migration and Scaling Considerations

### Phase 1: Relational + JSONB Only (MVP)
Start with the relational tables and JSONB columns from this schema. TimescaleDB and Apache AGE are not installed. Telematics data goes into a standard partitioned table. Graph queries are not available -- use standard SQL JOINs.

### Phase 2: Add TimescaleDB (When Telematics Is Integrated)
When GPS telematics integration is implemented (v1.1 per the features roadmap):
1. Install TimescaleDB extension.
2. Convert the existing telematics table to a hypertable: `SELECT create_hypertable('telemetry', by_range('time'), migrate_data => true);`
3. Create continuous aggregates for hourly, daily, monthly rollups.
4. Configure compression and retention policies.
5. This migration is non-destructive and can be done on a live database.

### Phase 3: Add Apache AGE (When Relationship Queries Are Needed)
When the product needs asset lifecycle views, customer risk scoring, or equipment substitution:
1. Install Apache AGE extension.
2. Create the graph schema.
3. Run a one-time backfill script to create vertices and edges from existing relational data.
4. Add triggers or CDC to keep the graph in sync going forward.
5. Build graph-powered API endpoints alongside existing relational queries.

### Phase 4: Scale
- TimescaleDB: enable multi-node for distributed hypertables if a single node cannot handle the ingestion rate.
- Apache AGE: monitor graph query performance; consider materializing frequently-traversed paths as relational summary tables if Cypher queries become slow.
- PostGIS: add spatial indexes as the number of geofences and location queries grows.

---

## Summary

This model is the most technically ambitious of the four suggestions, combining three PostgreSQL extensions (TimescaleDB, Apache AGE, PostGIS) with standard relational tables and JSONB columns. It is purpose-built for the rental equipment management domain: time-series hypertables for the high-volume telematics data that flows from GPS trackers and engine monitors, graph relationships for the complex web of asset-customer-contract-damage-maintenance interactions, and spatial indexes for location-based queries. The key advantage is that all of this runs inside a single PostgreSQL instance, avoiding the synchronization complexity of a polyglot persistence architecture. The key risk is operational complexity -- three query paradigms, extension compatibility constraints, and a more limited hosting landscape. This model is best adopted incrementally: start with relational + JSONB (Phase 1), add TimescaleDB when telematics data flows are live (Phase 2), and introduce the graph layer when relationship-based product features justify it (Phase 3).
