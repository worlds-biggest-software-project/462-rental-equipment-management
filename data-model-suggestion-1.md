# Data Model Suggestion 1: Normalized Relational Database (PostgreSQL)

> Project: Rental Equipment Management (462)
> Model Type: Fully normalized relational schema (3NF) on PostgreSQL 16+
> Generated: 2026-05-25

---

## Overview

This model uses a traditional normalized relational schema on PostgreSQL. Every domain concept maps to its own table with strict foreign key constraints, enforced types, and explicit join paths. The design follows Third Normal Form (3NF) throughout, with controlled denormalization only in reporting views.

PostgreSQL is chosen for its mature ecosystem, PostGIS extension for GPS coordinate handling, strong ACID guarantees critical for financial transactions (invoicing, payments, deposits), and broad hosting compatibility (self-hosted and cloud-managed).

---

## Schema Design

### 1. Organization and Location Management

```sql
-- Tenancy: supports multi-tenant SaaS or single-tenant self-hosted
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

-- Depots / branches / warehouses
CREATE TABLE depots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(20) NOT NULL,
    address_line1   VARCHAR(255),
    address_line2   VARCHAR(255),
    city            VARCHAR(100),
    state_province  VARCHAR(100),
    postal_code     VARCHAR(20),
    country_code    CHAR(2) NOT NULL DEFAULT 'US',
    latitude        DECIMAL(10, 7),
    longitude       DECIMAL(10, 7),
    phone           VARCHAR(30),
    email           VARCHAR(255),
    operating_hours JSONB,  -- e.g. {"mon": {"open": "07:00", "close": "17:00"}, ...}
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, code)
);

CREATE INDEX idx_depots_org ON depots(organization_id);
```

### 2. User and Access Management

```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    email           VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    phone           VARCHAR(30),
    role            VARCHAR(50) NOT NULL DEFAULT 'staff',
        -- roles: admin, manager, staff, driver, mechanic, readonly
    depot_id        UUID REFERENCES depots(id),  -- primary depot assignment
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, email)
);

CREATE TABLE user_depot_access (
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    depot_id        UUID NOT NULL REFERENCES depots(id),
    access_level    VARCHAR(20) NOT NULL DEFAULT 'full', -- full, readonly
    PRIMARY KEY (user_id, depot_id)
);
```

### 3. Equipment Categories and Asset Registry

```sql
-- Hierarchical equipment categories (e.g. Earthmoving > Excavators > Mini Excavators)
CREATE TABLE equipment_categories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    parent_id       UUID REFERENCES equipment_categories(id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50) NOT NULL,
    description     TEXT,
    sort_order      INT NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, code)
);

CREATE INDEX idx_eqcat_parent ON equipment_categories(parent_id);

-- Core asset table: one row per physical piece of equipment
CREATE TABLE assets (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    category_id         UUID NOT NULL REFERENCES equipment_categories(id),
    asset_number        VARCHAR(50) NOT NULL,  -- internal fleet number
    serial_number       VARCHAR(100),
    vin                 VARCHAR(50),           -- for vehicles
    make                VARCHAR(100),
    model               VARCHAR(100),
    year                INT,
    description         TEXT,
    purchase_date       DATE,
    purchase_cost       DECIMAL(12, 2),
    current_value       DECIMAL(12, 2),
    depreciation_method VARCHAR(30),  -- straight_line, declining_balance
    salvage_value       DECIMAL(12, 2),
    useful_life_months  INT,
    weight_kg           DECIMAL(10, 2),
    fuel_type           VARCHAR(30),  -- diesel, petrol, electric, none
    hour_meter_reading  DECIMAL(12, 1),
    odometer_reading    DECIMAL(12, 1),
    home_depot_id       UUID NOT NULL REFERENCES depots(id),
    current_depot_id    UUID NOT NULL REFERENCES depots(id),
    status              VARCHAR(30) NOT NULL DEFAULT 'available',
        -- available, on_rent, reserved, in_maintenance, in_transit,
        -- out_of_service, retired
    condition_grade     VARCHAR(20) DEFAULT 'good',
        -- excellent, good, fair, poor
    photo_url           VARCHAR(500),
    barcode             VARCHAR(100),
    qr_code             VARCHAR(100),
    insurance_policy_id VARCHAR(100),
    insurance_expiry    DATE,
    registration_number VARCHAR(50),
    registration_expiry DATE,
    last_inspection_at  TIMESTAMPTZ,
    next_service_due_at TIMESTAMPTZ,
    next_service_hours  DECIMAL(12, 1),
    is_active           BOOLEAN NOT NULL DEFAULT true,
    retired_at          TIMESTAMPTZ,
    retirement_reason   VARCHAR(255),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, asset_number)
);

CREATE INDEX idx_assets_org_status ON assets(organization_id, status);
CREATE INDEX idx_assets_category ON assets(category_id);
CREATE INDEX idx_assets_current_depot ON assets(current_depot_id);
CREATE INDEX idx_assets_home_depot ON assets(home_depot_id);
CREATE INDEX idx_assets_serial ON assets(serial_number) WHERE serial_number IS NOT NULL;

-- Asset photos (multiple per asset)
CREATE TABLE asset_photos (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID NOT NULL REFERENCES assets(id) ON DELETE CASCADE,
    photo_url       VARCHAR(500) NOT NULL,
    caption         VARCHAR(255),
    taken_at        TIMESTAMPTZ,
    uploaded_by     UUID REFERENCES users(id),
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_asset_photos_asset ON asset_photos(asset_id);

-- Asset documents (manuals, certificates, warranties)
CREATE TABLE asset_documents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID NOT NULL REFERENCES assets(id) ON DELETE CASCADE,
    document_type   VARCHAR(50) NOT NULL,
        -- manual, warranty, certificate, registration, insurance
    title           VARCHAR(255) NOT NULL,
    file_url        VARCHAR(500) NOT NULL,
    expiry_date     DATE,
    uploaded_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_asset_docs_asset ON asset_documents(asset_id);
CREATE INDEX idx_asset_docs_expiry ON asset_documents(expiry_date) WHERE expiry_date IS NOT NULL;
```

### 4. GPS Telematics Integration

```sql
-- Telematics providers configured per organization
CREATE TABLE telematics_providers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    provider_type   VARCHAR(50) NOT NULL,
        -- geotab, samsara, caterpillar_visionlink, deere_jdlink, generic_aemp
    name            VARCHAR(255) NOT NULL,
    api_endpoint    VARCHAR(500),
    credentials     JSONB,  -- encrypted at rest; OAuth tokens, API keys
    sync_interval   INT NOT NULL DEFAULT 300,  -- seconds
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Link assets to their telematics device identifiers
CREATE TABLE asset_telematics_links (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID NOT NULL REFERENCES assets(id),
    provider_id     UUID NOT NULL REFERENCES telematics_providers(id),
    device_id       VARCHAR(255) NOT NULL,  -- provider-specific device identifier
    is_active       BOOLEAN NOT NULL DEFAULT true,
    linked_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (provider_id, device_id)
);

-- Telematics data points (GPS position, hours, fuel)
-- NOTE: For high-volume deployments, consider partitioning by recorded_at
CREATE TABLE telematics_readings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID NOT NULL REFERENCES assets(id),
    provider_id     UUID NOT NULL REFERENCES telematics_providers(id),
    recorded_at     TIMESTAMPTZ NOT NULL,
    received_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    latitude        DECIMAL(10, 7),
    longitude       DECIMAL(10, 7),
    altitude_m      DECIMAL(8, 2),
    speed_kmh       DECIMAL(6, 1),
    heading         DECIMAL(5, 1),
    engine_hours    DECIMAL(12, 1),
    odometer_km     DECIMAL(12, 1),
    fuel_level_pct  DECIMAL(5, 2),
    fuel_consumed_l DECIMAL(12, 2),
    engine_status   VARCHAR(20),  -- running, idle, off
    battery_voltage DECIMAL(5, 2),
    raw_payload     JSONB  -- original provider payload for debugging
) PARTITION BY RANGE (recorded_at);

-- Create monthly partitions (example for initial setup)
CREATE TABLE telematics_readings_2026_01 PARTITION OF telematics_readings
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE telematics_readings_2026_02 PARTITION OF telematics_readings
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
-- ... create partitions as needed

CREATE INDEX idx_telem_asset_time ON telematics_readings(asset_id, recorded_at DESC);
CREATE INDEX idx_telem_recorded ON telematics_readings(recorded_at DESC);
```

### 5. Customer Management

```sql
CREATE TABLE customers (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    customer_number     VARCHAR(50) NOT NULL,
    customer_type       VARCHAR(20) NOT NULL DEFAULT 'business',
        -- business, individual
    company_name        VARCHAR(255),
    first_name          VARCHAR(100),
    last_name           VARCHAR(100),
    email               VARCHAR(255),
    phone               VARCHAR(30),
    mobile              VARCHAR(30),
    billing_address_line1 VARCHAR(255),
    billing_address_line2 VARCHAR(255),
    billing_city        VARCHAR(100),
    billing_state       VARCHAR(100),
    billing_postal_code VARCHAR(20),
    billing_country     CHAR(2) DEFAULT 'US',
    tax_exempt          BOOLEAN NOT NULL DEFAULT false,
    tax_id              VARCHAR(50),
    credit_limit        DECIMAL(12, 2),
    payment_terms_days  INT NOT NULL DEFAULT 30,
    account_status      VARCHAR(20) NOT NULL DEFAULT 'active',
        -- active, on_hold, suspended, closed
    notes               TEXT,
    portal_enabled      BOOLEAN NOT NULL DEFAULT false,
    portal_password_hash VARCHAR(255),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, customer_number)
);

CREATE INDEX idx_customers_org ON customers(organization_id);
CREATE INDEX idx_customers_email ON customers(email) WHERE email IS NOT NULL;
CREATE INDEX idx_customers_name ON customers(organization_id, company_name, last_name);

-- Customer contacts (multiple per company)
CREATE TABLE customer_contacts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id     UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    email           VARCHAR(255),
    phone           VARCHAR(30),
    mobile          VARCHAR(30),
    job_title       VARCHAR(100),
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    can_sign        BOOLEAN NOT NULL DEFAULT false,  -- authorized to sign contracts
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cust_contacts ON customer_contacts(customer_id);

-- Customer certificates of insurance
CREATE TABLE customer_insurance_certificates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id     UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    insurer_name    VARCHAR(255) NOT NULL,
    policy_number   VARCHAR(100) NOT NULL,
    coverage_type   VARCHAR(100),
    coverage_amount DECIMAL(14, 2),
    effective_date  DATE NOT NULL,
    expiry_date     DATE NOT NULL,
    document_url    VARCHAR(500),
    verified_by     UUID REFERENCES users(id),
    verified_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cust_insurance ON customer_insurance_certificates(customer_id);
CREATE INDEX idx_cust_insurance_expiry ON customer_insurance_certificates(expiry_date);

-- Delivery site addresses (where equipment is delivered)
CREATE TABLE customer_sites (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id     UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    site_name       VARCHAR(255) NOT NULL,
    address_line1   VARCHAR(255),
    address_line2   VARCHAR(255),
    city            VARCHAR(100),
    state_province  VARCHAR(100),
    postal_code     VARCHAR(20),
    country_code    CHAR(2) DEFAULT 'US',
    latitude        DECIMAL(10, 7),
    longitude       DECIMAL(10, 7),
    contact_name    VARCHAR(200),
    contact_phone   VARCHAR(30),
    access_notes    TEXT,  -- gate codes, delivery instructions
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cust_sites ON customer_sites(customer_id);
```

### 6. Rate Tables and Pricing

```sql
-- Rate structures: per-category or per-asset pricing
CREATE TABLE rate_tables (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    is_default      BOOLEAN NOT NULL DEFAULT false,
    effective_from  DATE NOT NULL,
    effective_to    DATE,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rate_tables_org ON rate_tables(organization_id);

-- Individual rate entries
CREATE TABLE rate_entries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rate_table_id   UUID NOT NULL REFERENCES rate_tables(id) ON DELETE CASCADE,
    category_id     UUID REFERENCES equipment_categories(id),
    asset_id        UUID REFERENCES assets(id),  -- NULL = category-level rate
    period_type     VARCHAR(20) NOT NULL,
        -- hourly, daily, weekly, monthly, shift
    rate_amount     DECIMAL(10, 2) NOT NULL,
    minimum_charge  DECIMAL(10, 2),
    minimum_period  INT,  -- minimum billable periods
    weekend_rate    DECIMAL(10, 2),  -- override for Sat/Sun
    holiday_rate    DECIMAL(10, 2),  -- override for holidays
    overtime_rate   DECIMAL(10, 2),  -- beyond standard hours
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT chk_rate_target CHECK (
        (category_id IS NOT NULL AND asset_id IS NULL)
        OR (category_id IS NULL AND asset_id IS NOT NULL)
    )
);

CREATE INDEX idx_rate_entries_table ON rate_entries(rate_table_id);
CREATE INDEX idx_rate_entries_category ON rate_entries(category_id);
CREATE INDEX idx_rate_entries_asset ON rate_entries(asset_id);

-- Customer-specific rate overrides
CREATE TABLE customer_rate_overrides (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    rate_table_id   UUID NOT NULL REFERENCES rate_tables(id),
    discount_pct    DECIMAL(5, 2),
    effective_from  DATE NOT NULL,
    effective_to    DATE,
    approved_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (customer_id, rate_table_id, effective_from)
);

-- Extra charges catalog (fuel, delivery, cleaning, operator, etc.)
CREATE TABLE extra_charge_types (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    code            VARCHAR(50) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    charge_method   VARCHAR(30) NOT NULL,
        -- fixed, per_day, per_hour, per_km, per_unit, percentage
    default_amount  DECIMAL(10, 2),
    tax_code        VARCHAR(20),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, code)
);
```

### 7. Quotes, Rental Contracts, and Reservations

```sql
-- Quotes (may convert to rental contracts)
CREATE TABLE quotes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    quote_number    VARCHAR(50) NOT NULL,
    customer_id     UUID NOT NULL REFERENCES customers(id),
    contact_id      UUID REFERENCES customer_contacts(id),
    depot_id        UUID NOT NULL REFERENCES depots(id),
    site_id         UUID REFERENCES customer_sites(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
        -- draft, sent, accepted, declined, expired, converted
    rental_start    TIMESTAMPTZ NOT NULL,
    rental_end      TIMESTAMPTZ,
    subtotal        DECIMAL(12, 2) NOT NULL DEFAULT 0,
    tax_amount      DECIMAL(12, 2) NOT NULL DEFAULT 0,
    total           DECIMAL(12, 2) NOT NULL DEFAULT 0,
    notes           TEXT,
    valid_until     DATE,
    created_by      UUID NOT NULL REFERENCES users(id),
    converted_to_contract_id UUID,  -- set when quote becomes a contract
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, quote_number)
);

CREATE TABLE quote_lines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    quote_id        UUID NOT NULL REFERENCES quotes(id) ON DELETE CASCADE,
    asset_id        UUID REFERENCES assets(id),
    category_id     UUID REFERENCES equipment_categories(id),
    description     VARCHAR(500) NOT NULL,
    quantity        INT NOT NULL DEFAULT 1,
    period_type     VARCHAR(20) NOT NULL,
    rate_amount     DECIMAL(10, 2) NOT NULL,
    estimated_periods INT NOT NULL DEFAULT 1,
    line_total      DECIMAL(12, 2) NOT NULL,
    sort_order      INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Rental contracts (the core transactional entity)
CREATE TABLE rental_contracts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    contract_number VARCHAR(50) NOT NULL,
    quote_id        UUID REFERENCES quotes(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    contact_id      UUID REFERENCES customer_contacts(id),
    depot_id        UUID NOT NULL REFERENCES depots(id),
    site_id         UUID REFERENCES customer_sites(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
        -- draft, pending_signature, active, extended, completed,
        -- cancelled, disputed
    rental_start    TIMESTAMPTZ NOT NULL,
    rental_end_expected TIMESTAMPTZ,
    rental_end_actual   TIMESTAMPTZ,
    delivery_method VARCHAR(20) NOT NULL DEFAULT 'delivery',
        -- delivery, pickup
    delivery_date   TIMESTAMPTZ,
    return_method   VARCHAR(20) DEFAULT 'pickup',
        -- pickup, dropoff
    return_date     TIMESTAMPTZ,
    rate_table_id   UUID REFERENCES rate_tables(id),
    deposit_amount  DECIMAL(12, 2) NOT NULL DEFAULT 0,
    deposit_paid    BOOLEAN NOT NULL DEFAULT false,
    subtotal        DECIMAL(12, 2) NOT NULL DEFAULT 0,
    tax_amount      DECIMAL(12, 2) NOT NULL DEFAULT 0,
    damage_charges  DECIMAL(12, 2) NOT NULL DEFAULT 0,
    extra_charges   DECIMAL(12, 2) NOT NULL DEFAULT 0,
    total           DECIMAL(12, 2) NOT NULL DEFAULT 0,
    terms_accepted  BOOLEAN NOT NULL DEFAULT false,
    signature_url   VARCHAR(500),  -- stored e-signature image
    signature_ip    INET,
    signed_at       TIMESTAMPTZ,
    signed_by_name  VARCHAR(200),
    notes           TEXT,
    internal_notes  TEXT,
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, contract_number)
);

CREATE INDEX idx_contracts_org_status ON rental_contracts(organization_id, status);
CREATE INDEX idx_contracts_customer ON rental_contracts(customer_id);
CREATE INDEX idx_contracts_depot ON rental_contracts(depot_id);
CREATE INDEX idx_contracts_dates ON rental_contracts(rental_start, rental_end_expected);

-- Contract line items (each asset rented)
CREATE TABLE contract_lines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id     UUID NOT NULL REFERENCES rental_contracts(id) ON DELETE CASCADE,
    asset_id        UUID NOT NULL REFERENCES assets(id),
    description     VARCHAR(500),
    period_type     VARCHAR(20) NOT NULL,
    rate_amount     DECIMAL(10, 2) NOT NULL,
    minimum_charge  DECIMAL(10, 2),
    estimated_periods INT,
    actual_periods  DECIMAL(10, 2),
    line_subtotal   DECIMAL(12, 2) NOT NULL DEFAULT 0,
    dispatched_at   TIMESTAMPTZ,
    returned_at     TIMESTAMPTZ,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
        -- pending, dispatched, on_rent, returned, cancelled
    hour_meter_out  DECIMAL(12, 1),
    hour_meter_in   DECIMAL(12, 1),
    odometer_out    DECIMAL(12, 1),
    odometer_in     DECIMAL(12, 1),
    fuel_level_out  DECIMAL(5, 2),
    fuel_level_in   DECIMAL(5, 2),
    sort_order      INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_contract_lines_contract ON contract_lines(contract_id);
CREATE INDEX idx_contract_lines_asset ON contract_lines(asset_id);

-- Extra charges applied to a contract
CREATE TABLE contract_extras (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id     UUID NOT NULL REFERENCES rental_contracts(id) ON DELETE CASCADE,
    contract_line_id UUID REFERENCES contract_lines(id),
    charge_type_id  UUID NOT NULL REFERENCES extra_charge_types(id),
    description     VARCHAR(500),
    quantity        DECIMAL(10, 2) NOT NULL DEFAULT 1,
    unit_amount     DECIMAL(10, 2) NOT NULL,
    total_amount    DECIMAL(12, 2) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Contract extensions (extending the return date)
CREATE TABLE contract_extensions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id     UUID NOT NULL REFERENCES rental_contracts(id),
    previous_end    TIMESTAMPTZ NOT NULL,
    new_end         TIMESTAMPTZ NOT NULL,
    reason          TEXT,
    approved_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 8. Inspections and Damage Management

```sql
-- Inspection templates (define what to check)
CREATE TABLE inspection_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    category_id     UUID REFERENCES equipment_categories(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE inspection_template_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    template_id     UUID NOT NULL REFERENCES inspection_templates(id) ON DELETE CASCADE,
    item_label      VARCHAR(255) NOT NULL,
    item_type       VARCHAR(30) NOT NULL,
        -- pass_fail, rating_1_5, text, photo_required
    sort_order      INT NOT NULL DEFAULT 0,
    is_required     BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Completed inspections
CREATE TABLE inspections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_line_id UUID NOT NULL REFERENCES contract_lines(id),
    asset_id        UUID NOT NULL REFERENCES assets(id),
    template_id     UUID REFERENCES inspection_templates(id),
    inspection_type VARCHAR(20) NOT NULL,
        -- pre_rental, post_rental, periodic, damage_report
    status          VARCHAR(20) NOT NULL DEFAULT 'in_progress',
        -- in_progress, completed, reviewed
    inspector_id    UUID NOT NULL REFERENCES users(id),
    inspected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    overall_condition VARCHAR(20),
        -- excellent, good, fair, poor, damaged
    notes           TEXT,
    hour_meter      DECIMAL(12, 1),
    odometer        DECIMAL(12, 1),
    latitude        DECIMAL(10, 7),
    longitude       DECIMAL(10, 7),
    completed_at    TIMESTAMPTZ,
    reviewed_by     UUID REFERENCES users(id),
    reviewed_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_inspections_contract_line ON inspections(contract_line_id);
CREATE INDEX idx_inspections_asset ON inspections(asset_id);

-- Individual inspection check items
CREATE TABLE inspection_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inspection_id   UUID NOT NULL REFERENCES inspections(id) ON DELETE CASCADE,
    template_item_id UUID REFERENCES inspection_template_items(id),
    item_label      VARCHAR(255) NOT NULL,
    result          VARCHAR(20),  -- pass, fail, na
    rating          INT,
    notes           TEXT,
    sort_order      INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Photos attached to inspections
CREATE TABLE inspection_photos (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inspection_id   UUID NOT NULL REFERENCES inspections(id) ON DELETE CASCADE,
    inspection_item_id UUID REFERENCES inspection_items(id),
    photo_url       VARCHAR(500) NOT NULL,
    caption         VARCHAR(255),
    taken_at        TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_inspection_photos ON inspection_photos(inspection_id);

-- Damage records linked to inspections
CREATE TABLE damage_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inspection_id   UUID NOT NULL REFERENCES inspections(id),
    asset_id        UUID NOT NULL REFERENCES assets(id),
    contract_line_id UUID NOT NULL REFERENCES contract_lines(id),
    description     TEXT NOT NULL,
    severity        VARCHAR(20) NOT NULL,
        -- minor, moderate, major, critical
    estimated_repair_cost DECIMAL(10, 2),
    charge_to_customer    DECIMAL(10, 2),
    charge_approved       BOOLEAN NOT NULL DEFAULT false,
    charge_approved_by    UUID REFERENCES users(id),
    insurance_claim_ref   VARCHAR(100),
    status          VARCHAR(20) NOT NULL DEFAULT 'reported',
        -- reported, assessed, charged, repaired, insurance_claimed, closed
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_damage_asset ON damage_records(asset_id);
CREATE INDEX idx_damage_contract_line ON damage_records(contract_line_id);

-- Photos specifically tied to damage records
CREATE TABLE damage_photos (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    damage_record_id UUID NOT NULL REFERENCES damage_records(id) ON DELETE CASCADE,
    photo_url       VARCHAR(500) NOT NULL,
    photo_type      VARCHAR(20) NOT NULL,
        -- pre_rental, post_rental, during_repair, after_repair
    caption         VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 9. Maintenance and Work Orders

```sql
-- Maintenance schedules (preventive maintenance rules)
CREATE TABLE maintenance_schedules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    category_id     UUID REFERENCES equipment_categories(id),
    asset_id        UUID REFERENCES assets(id),
    schedule_type   VARCHAR(20) NOT NULL,
        -- calendar, meter, both
    interval_days   INT,       -- for calendar-based
    interval_hours  DECIMAL(10, 1),  -- for meter-based
    interval_km     DECIMAL(10, 1),  -- for odometer-based
    description     TEXT,
    estimated_duration_hours DECIMAL(5, 1),
    estimated_cost  DECIMAL(10, 2),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Work orders
CREATE TABLE work_orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    work_order_number VARCHAR(50) NOT NULL,
    asset_id        UUID NOT NULL REFERENCES assets(id),
    depot_id        UUID NOT NULL REFERENCES depots(id),
    maintenance_schedule_id UUID REFERENCES maintenance_schedules(id),
    damage_record_id UUID REFERENCES damage_records(id),
    work_order_type VARCHAR(30) NOT NULL,
        -- preventive, corrective, inspection, damage_repair, modification
    priority        VARCHAR(20) NOT NULL DEFAULT 'normal',
        -- urgent, high, normal, low
    status          VARCHAR(20) NOT NULL DEFAULT 'open',
        -- open, assigned, in_progress, awaiting_parts, completed, cancelled
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    assigned_to     UUID REFERENCES users(id),
    scheduled_start TIMESTAMPTZ,
    scheduled_end   TIMESTAMPTZ,
    actual_start    TIMESTAMPTZ,
    actual_end      TIMESTAMPTZ,
    hour_meter_at   DECIMAL(12, 1),
    labor_hours     DECIMAL(8, 2),
    labor_cost      DECIMAL(10, 2),
    parts_cost      DECIMAL(10, 2),
    total_cost      DECIMAL(10, 2),
    completion_notes TEXT,
    completed_by    UUID REFERENCES users(id),
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, work_order_number)
);

CREATE INDEX idx_wo_asset ON work_orders(asset_id);
CREATE INDEX idx_wo_status ON work_orders(organization_id, status);
CREATE INDEX idx_wo_assigned ON work_orders(assigned_to) WHERE assigned_to IS NOT NULL;
CREATE INDEX idx_wo_scheduled ON work_orders(scheduled_start, scheduled_end);

-- Work order parts used
CREATE TABLE work_order_parts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_id   UUID NOT NULL REFERENCES work_orders(id) ON DELETE CASCADE,
    part_number     VARCHAR(100),
    part_name       VARCHAR(255) NOT NULL,
    quantity        DECIMAL(10, 2) NOT NULL DEFAULT 1,
    unit_cost       DECIMAL(10, 2) NOT NULL,
    total_cost      DECIMAL(10, 2) NOT NULL,
    supplier        VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Work order labor entries
CREATE TABLE work_order_labor (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_id   UUID NOT NULL REFERENCES work_orders(id) ON DELETE CASCADE,
    technician_id   UUID NOT NULL REFERENCES users(id),
    work_date       DATE NOT NULL,
    hours           DECIMAL(5, 2) NOT NULL,
    hourly_rate     DECIMAL(8, 2),
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 10. Invoicing and Payments

```sql
CREATE TABLE invoices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    invoice_number  VARCHAR(50) NOT NULL,
    contract_id     UUID NOT NULL REFERENCES rental_contracts(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    invoice_type    VARCHAR(20) NOT NULL DEFAULT 'rental',
        -- rental, damage, deposit, credit_note, progress
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
        -- draft, sent, viewed, partially_paid, paid, overdue,
        -- void, written_off
    invoice_date    DATE NOT NULL DEFAULT CURRENT_DATE,
    due_date        DATE NOT NULL,
    period_start    DATE,
    period_end      DATE,
    subtotal        DECIMAL(12, 2) NOT NULL DEFAULT 0,
    tax_amount      DECIMAL(12, 2) NOT NULL DEFAULT 0,
    total           DECIMAL(12, 2) NOT NULL DEFAULT 0,
    amount_paid     DECIMAL(12, 2) NOT NULL DEFAULT 0,
    balance_due     DECIMAL(12, 2) NOT NULL DEFAULT 0,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    notes           TEXT,
    -- Accounting integration
    quickbooks_id   VARCHAR(100),
    xero_id         VARCHAR(100),
    synced_at       TIMESTAMPTZ,
    created_by      UUID NOT NULL REFERENCES users(id),
    sent_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, invoice_number)
);

CREATE INDEX idx_invoices_contract ON invoices(contract_id);
CREATE INDEX idx_invoices_customer ON invoices(customer_id);
CREATE INDEX idx_invoices_status ON invoices(organization_id, status);
CREATE INDEX idx_invoices_due ON invoices(due_date) WHERE status NOT IN ('paid', 'void');

CREATE TABLE invoice_lines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id      UUID NOT NULL REFERENCES invoices(id) ON DELETE CASCADE,
    contract_line_id UUID REFERENCES contract_lines(id),
    description     VARCHAR(500) NOT NULL,
    quantity        DECIMAL(10, 2) NOT NULL DEFAULT 1,
    unit_amount     DECIMAL(10, 2) NOT NULL,
    tax_rate        DECIMAL(5, 4) NOT NULL DEFAULT 0,
    tax_amount      DECIMAL(10, 2) NOT NULL DEFAULT 0,
    line_total      DECIMAL(12, 2) NOT NULL,
    line_type       VARCHAR(30) NOT NULL DEFAULT 'rental',
        -- rental, extra, damage, fuel, delivery, deposit, discount
    sort_order      INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE payments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    invoice_id      UUID NOT NULL REFERENCES invoices(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    payment_method  VARCHAR(30) NOT NULL,
        -- credit_card, ach, check, cash, wire, stripe
    amount          DECIMAL(12, 2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    payment_date    DATE NOT NULL DEFAULT CURRENT_DATE,
    reference       VARCHAR(100),
    stripe_payment_id VARCHAR(100),
    stripe_charge_id  VARCHAR(100),
    status          VARCHAR(20) NOT NULL DEFAULT 'completed',
        -- pending, completed, failed, refunded, partial_refund
    notes           TEXT,
    recorded_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payments_invoice ON payments(invoice_id);
CREATE INDEX idx_payments_customer ON payments(customer_id);
CREATE INDEX idx_payments_stripe ON payments(stripe_payment_id) WHERE stripe_payment_id IS NOT NULL;

-- Deposit management
CREATE TABLE deposits (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id     UUID NOT NULL REFERENCES rental_contracts(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    deposit_type    VARCHAR(20) NOT NULL,
        -- security, damage, fuel
    amount          DECIMAL(12, 2) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'held',
        -- held, partially_applied, applied, refunded
    stripe_payment_intent_id VARCHAR(100),
    collected_at    TIMESTAMPTZ,
    applied_amount  DECIMAL(12, 2) NOT NULL DEFAULT 0,
    refund_amount   DECIMAL(12, 2) NOT NULL DEFAULT 0,
    refunded_at     TIMESTAMPTZ,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_deposits_contract ON deposits(contract_id);
```

### 11. Dispatch and Delivery

```sql
CREATE TABLE delivery_orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    contract_id     UUID NOT NULL REFERENCES rental_contracts(id),
    order_type      VARCHAR(20) NOT NULL,
        -- delivery, pickup, transfer
    status          VARCHAR(20) NOT NULL DEFAULT 'scheduled',
        -- scheduled, assigned, in_transit, completed, cancelled
    depot_id        UUID NOT NULL REFERENCES depots(id),
    driver_id       UUID REFERENCES users(id),
    vehicle_id      UUID REFERENCES assets(id),  -- delivery truck
    scheduled_date  TIMESTAMPTZ NOT NULL,
    scheduled_window_start TIMESTAMPTZ,
    scheduled_window_end   TIMESTAMPTZ,
    actual_departure TIMESTAMPTZ,
    actual_arrival  TIMESTAMPTZ,
    delivery_address_line1 VARCHAR(255),
    delivery_address_line2 VARCHAR(255),
    delivery_city   VARCHAR(100),
    delivery_state  VARCHAR(100),
    delivery_postal_code VARCHAR(20),
    delivery_latitude  DECIMAL(10, 7),
    delivery_longitude DECIMAL(10, 7),
    customer_signature_url VARCHAR(500),
    signed_by_name  VARCHAR(200),
    signed_at       TIMESTAMPTZ,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_delivery_contract ON delivery_orders(contract_id);
CREATE INDEX idx_delivery_driver ON delivery_orders(driver_id, scheduled_date);
CREATE INDEX idx_delivery_status ON delivery_orders(organization_id, status, scheduled_date);

-- Items on each delivery
CREATE TABLE delivery_order_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    delivery_order_id UUID NOT NULL REFERENCES delivery_orders(id) ON DELETE CASCADE,
    contract_line_id UUID NOT NULL REFERENCES contract_lines(id),
    asset_id        UUID NOT NULL REFERENCES assets(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
        -- pending, loaded, delivered, confirmed
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Inter-depot transfers
CREATE TABLE depot_transfers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    asset_id        UUID NOT NULL REFERENCES assets(id),
    from_depot_id   UUID NOT NULL REFERENCES depots(id),
    to_depot_id     UUID NOT NULL REFERENCES depots(id),
    reason          VARCHAR(255),
    contract_id     UUID REFERENCES rental_contracts(id),  -- if transfer is to fulfil a booking
    status          VARCHAR(20) NOT NULL DEFAULT 'requested',
        -- requested, approved, in_transit, completed, cancelled
    requested_by    UUID NOT NULL REFERENCES users(id),
    approved_by     UUID REFERENCES users(id),
    departed_at     TIMESTAMPTZ,
    arrived_at      TIMESTAMPTZ,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_transfers_asset ON depot_transfers(asset_id);
CREATE INDEX idx_transfers_status ON depot_transfers(organization_id, status);
```

### 12. Notifications and Audit Trail

```sql
CREATE TABLE notifications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    recipient_type  VARCHAR(20) NOT NULL, -- customer, user
    recipient_id    UUID NOT NULL,
    channel         VARCHAR(20) NOT NULL, -- email, sms, in_app
    notification_type VARCHAR(50) NOT NULL,
        -- booking_confirmation, dispatch_reminder, return_reminder,
        -- overdue_notice, invoice_sent, payment_received,
        -- insurance_expiry_warning, maintenance_due
    subject         VARCHAR(255),
    body            TEXT,
    reference_type  VARCHAR(50),  -- rental_contract, invoice, etc.
    reference_id    UUID,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
        -- pending, sent, delivered, failed, read
    sent_at         TIMESTAMPTZ,
    read_at         TIMESTAMPTZ,
    error_message   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_notifications_recipient ON notifications(recipient_type, recipient_id);
CREATE INDEX idx_notifications_status ON notifications(status) WHERE status = 'pending';

-- Audit log for compliance and dispute resolution
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    user_id         UUID REFERENCES users(id),
    action          VARCHAR(50) NOT NULL,
        -- create, update, delete, status_change, sign, approve
    entity_type     VARCHAR(50) NOT NULL,
        -- asset, contract, invoice, payment, inspection, work_order
    entity_id       UUID NOT NULL,
    changes         JSONB,  -- before/after values
    ip_address      INET,
    user_agent      VARCHAR(500),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_user ON audit_log(user_id);
CREATE INDEX idx_audit_created ON audit_log(created_at DESC);
```

### 13. Integration Sync Tracking

```sql
CREATE TABLE integration_connections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    provider        VARCHAR(50) NOT NULL,
        -- quickbooks, xero, sage, stripe, geotab, samsara, docusign
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
        -- pending, connected, disconnected, error
    access_token    TEXT,  -- encrypted at rest
    refresh_token   TEXT,  -- encrypted at rest
    token_expires_at TIMESTAMPTZ,
    realm_id        VARCHAR(100),  -- provider-specific tenant ID
    metadata        JSONB,
    connected_at    TIMESTAMPTZ,
    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, provider)
);

CREATE TABLE integration_sync_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    connection_id   UUID NOT NULL REFERENCES integration_connections(id),
    sync_type       VARCHAR(50) NOT NULL,
        -- invoice_push, payment_pull, customer_sync, telematics_pull
    direction       VARCHAR(10) NOT NULL, -- push, pull
    status          VARCHAR(20) NOT NULL,
        -- started, completed, partial, failed
    records_processed INT NOT NULL DEFAULT 0,
    records_failed  INT NOT NULL DEFAULT 0,
    error_details   JSONB,
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ
);
```

---

## Entity Relationship Summary

```
organizations ──1:N──> depots
organizations ──1:N──> users
organizations ──1:N──> equipment_categories
organizations ──1:N──> customers
organizations ──1:N──> rate_tables
organizations ──1:N──> assets

equipment_categories ──1:N──> assets (via category_id)
depots ──1:N──> assets (via home_depot_id, current_depot_id)

customers ──1:N──> customer_contacts
customers ──1:N──> customer_insurance_certificates
customers ──1:N──> customer_sites
customers ──1:N──> rental_contracts

rental_contracts ──1:N──> contract_lines
contract_lines ──N:1──> assets
rental_contracts ──1:N──> contract_extras
rental_contracts ──1:N──> contract_extensions
rental_contracts ──1:N──> invoices
rental_contracts ──1:N──> delivery_orders

contract_lines ──1:N──> inspections
inspections ──1:N──> inspection_items
inspections ──1:N──> inspection_photos
inspections ──1:N──> damage_records
damage_records ──1:N──> damage_photos

assets ──1:N──> work_orders
assets ──1:N──> telematics_readings
assets ──1:N──> depot_transfers

invoices ──1:N──> invoice_lines
invoices ──1:N──> payments
```

---

## Pros and Cons

### Pros

1. **Data integrity is maximized.** Foreign key constraints across all relationships prevent orphaned records. A contract line cannot reference a non-existent asset; a payment cannot reference a non-existent invoice. In a domain where billing disputes and equipment accountability are constant concerns, this integrity is invaluable.

2. **Mature query tooling.** Complex reporting queries -- fleet utilization rates, revenue per asset, maintenance cost vs. rental revenue -- are straightforward with standard SQL JOINs and aggregations. Every BI tool, reporting library, and analytics platform speaks SQL natively.

3. **Well-understood by hiring pool.** PostgreSQL relational modeling is the most widely known data architecture pattern. New developers can contribute immediately without learning a specialized paradigm.

4. **Transaction safety for financial operations.** ACID transactions ensure that contract creation, invoice generation, and payment recording are atomic operations. A partially-created invoice with missing line items cannot occur.

5. **Proven at the scale this project targets.** For operators with 50-500 assets, a well-indexed PostgreSQL database handles the full workload on a single server. The data volumes (thousands of contracts per year, millions of telematics readings per year) are well within PostgreSQL's single-node capacity.

6. **PostGIS integration.** GPS coordinates stored as DECIMAL columns can be upgraded to PostGIS GEOGRAPHY types for spatial queries (nearest depot, geofencing, distance calculations) without migrating to a different database.

### Cons

1. **Schema rigidity for equipment attributes.** Different equipment categories have vastly different specifications (a generator has kVA rating, fuel tank size; an excavator has bucket capacity, dig depth, boom reach). The normalized approach requires either a wide table with many nullable columns or an Entity-Attribute-Value pattern, both of which have usability drawbacks.

2. **Telematics data volume may outgrow single-table performance.** While PostgreSQL table partitioning handles moderate volumes, a fleet of 500 GPS-tracked assets reporting every 30 seconds generates ~525 million rows per year. Partition management becomes a maintenance burden.

3. **Rate table complexity creates many joins.** Resolving the effective rate for a specific asset-customer-date combination requires joining rate_tables, rate_entries, and customer_rate_overrides with date-range filtering. This query pattern is correct but verbose and can be slow without careful indexing.

4. **No built-in temporal queries.** Answering "what was this asset's status at 3pm yesterday?" requires querying the audit_log and reconstructing state from change records. The relational model stores current state, not history.

5. **Migration complexity grows with table count.** With 40+ tables, schema migrations during active production use require careful sequencing, especially for columns involved in billing calculations.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Database | PostgreSQL 16+ |
| Extensions | PostGIS (spatial), pg_cron (scheduled jobs), pgcrypto (token encryption) |
| Connection pooling | PgBouncer or built-in pool (Supabase, Neon) |
| Migration tool | Flyway, golang-migrate, or Prisma Migrate |
| ORM / query builder | Prisma (TypeScript), SQLAlchemy (Python), or Drizzle ORM |
| Hosting (cloud) | AWS RDS PostgreSQL, Google Cloud SQL, Azure Database for PostgreSQL, Neon, Supabase |
| Hosting (self-hosted) | PostgreSQL on Linux with streaming replication |
| Backup | pg_dump for logical backups; WAL archiving for point-in-time recovery |

---

## Migration and Scaling Considerations

### Initial Deployment (0-100 assets)
- Single PostgreSQL instance (4 vCPU, 16 GB RAM) handles all workloads.
- All tables in a single database, single schema.
- Monthly telematics partitions created manually or via pg_cron.

### Growth Phase (100-500 assets)
- Add read replica for reporting queries and analytics dashboards.
- Implement connection pooling (PgBouncer) to handle concurrent mobile app connections from field staff.
- Automate partition creation for telematics_readings.
- Consider materialized views for fleet utilization dashboards (refresh every 15 minutes).

### Scale Phase (500-2000 assets)
- Telematics data may warrant migration to TimescaleDB extension (hypertables replace manual partitioning).
- Archive completed contracts and invoices older than 3 years to a separate schema or cold storage.
- Implement logical replication for near-real-time data warehouse feeds.
- Consider Citus extension for horizontal sharding by organization_id if serving multiple large tenants.

### Data Retention
- Telematics readings: retain raw data for 12 months; aggregate to hourly summaries for 5 years.
- Audit log: retain for 7 years (regulatory compliance).
- Inspection photos: retain for contract duration + 3 years (dispute resolution).
- Completed contracts and invoices: retain indefinitely (financial records).

---

## Summary

This normalized relational model is the safest, most conventional choice for the rental equipment management domain. It enforces data integrity at the database level, supports complex billing and reporting queries natively, and runs on the most widely available and well-supported open-source database. The main trade-offs are schema rigidity for variable equipment attributes and eventual scaling concerns for high-frequency telematics data -- both of which can be addressed incrementally (JSONB columns for attributes, TimescaleDB for telematics) without a full re-architecture.
