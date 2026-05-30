# Data Model Suggestion 3: Hybrid Relational + Document (JSONB) Model

> Project: Rental Equipment Management (462)
> Model Type: PostgreSQL with strategic JSONB columns for flexible/variable data
> Generated: 2026-05-25

---

## Overview

This model takes the best of the normalized relational approach (Suggestion 1) and injects JSONB document columns at specific points where the data is inherently variable, sparse, or evolving. The core business entities -- contracts, invoices, customers, payments -- remain fully normalized with strict foreign keys and typed columns. Equipment attributes, inspection templates, rate configurations, telematics payloads, and integration metadata use JSONB columns to absorb variability without schema migrations.

This hybrid approach directly addresses the biggest weakness of the pure relational model: **equipment attribute variability**. A 20-ton excavator and a portable generator share almost no specifications beyond make, model, and serial number. In a pure relational schema, you either create a wide table with 50+ nullable columns (most empty for any given row), or you use Entity-Attribute-Value tables (slow to query, painful to report on). JSONB columns solve this cleanly: common fields stay in typed columns for indexing and joins, while category-specific attributes live in a JSONB column with GIN indexing for fast lookups.

---

## Design Principles

1. **Typed columns for data that participates in joins, foreign keys, or financial calculations.** Asset ID, contract status, invoice total, payment amount -- these must be typed columns with constraints.

2. **JSONB columns for data that varies by category, tenant, or integration.** Equipment specifications, custom inspection checklists, rate rule configurations, telematics raw payloads, webhook payloads from accounting systems.

3. **GIN indexes on JSONB columns that are queried frequently.** Not every JSONB column needs an index; some are write-heavy and rarely queried (raw telematics payloads).

4. **JSON Schema validation at the application layer.** PostgreSQL does not enforce JSONB structure natively. The application layer validates JSONB content against JSON Schema definitions before writing.

5. **Avoid JSONB for relationships.** Never store foreign keys or arrays of IDs inside JSONB. Relationships are always explicit join tables.

---

## Schema Design

### 1. Organization and Location (Normalized)

```sql
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    legal_name      VARCHAR(255),
    tax_id          VARCHAR(50),
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    timezone        VARCHAR(50) NOT NULL DEFAULT 'America/New_York',
    -- JSONB: tenant-specific configuration that evolves frequently
    settings        JSONB NOT NULL DEFAULT '{}'::jsonb,
    /*  settings example:
        {
            "branding": { "logo_url": "...", "primary_color": "#1a56db" },
            "billing": {
                "default_payment_terms_days": 30,
                "tax_rate": 0.0875,
                "weekend_billing": "exclude",
                "holiday_calendar": "US_federal"
            },
            "notifications": {
                "overdue_reminder_days": [7, 14, 30],
                "insurance_expiry_warning_days": 30,
                "maintenance_due_warning_days": 7
            },
            "integrations": {
                "accounting_provider": "quickbooks",
                "telematics_default": "geotab"
            }
        }
    */
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
    latitude        DECIMAL(10, 7),
    longitude       DECIMAL(10, 7),
    phone           VARCHAR(30),
    email           VARCHAR(255),
    -- JSONB: operating hours, holiday schedules, capacity limits
    config          JSONB NOT NULL DEFAULT '{}'::jsonb,
    /*  config example:
        {
            "operating_hours": {
                "mon": {"open": "07:00", "close": "17:00"},
                "tue": {"open": "07:00", "close": "17:00"},
                "sat": {"open": "08:00", "close": "12:00"},
                "sun": null
            },
            "max_storage_capacity": 150,
            "delivery_radius_km": 80,
            "accepts_customer_dropoff": true
        }
    */
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, code)
);
```

### 2. Equipment Categories with Attribute Schemas

```sql
-- Categories define what attributes their equipment should have
CREATE TABLE equipment_categories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    parent_id       UUID REFERENCES equipment_categories(id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50) NOT NULL,
    description     TEXT,
    sort_order      INT NOT NULL DEFAULT 0,
    -- JSONB: defines the expected attribute schema for this category
    -- Used for validation and dynamic form generation
    attribute_schema JSONB NOT NULL DEFAULT '[]'::jsonb,
    /*  attribute_schema example for "Excavators":
        [
            {
                "key": "operating_weight_kg",
                "label": "Operating Weight (kg)",
                "type": "number",
                "required": true,
                "unit": "kg"
            },
            {
                "key": "bucket_capacity_m3",
                "label": "Bucket Capacity (m3)",
                "type": "number",
                "required": true,
                "unit": "m3"
            },
            {
                "key": "max_dig_depth_m",
                "label": "Max Dig Depth (m)",
                "type": "number",
                "required": false,
                "unit": "m"
            },
            {
                "key": "boom_length_m",
                "label": "Boom Length (m)",
                "type": "number",
                "required": false,
                "unit": "m"
            },
            {
                "key": "tail_swing",
                "label": "Tail Swing Type",
                "type": "enum",
                "options": ["conventional", "reduced", "zero"],
                "required": false
            },
            {
                "key": "thumb_attachment",
                "label": "Has Thumb Attachment",
                "type": "boolean",
                "required": false,
                "default": false
            }
        ]

        Example for "Generators":
        [
            {
                "key": "power_kva",
                "label": "Power Output (kVA)",
                "type": "number",
                "required": true,
                "unit": "kVA"
            },
            {
                "key": "fuel_tank_capacity_l",
                "label": "Fuel Tank (litres)",
                "type": "number",
                "required": true,
                "unit": "L"
            },
            {
                "key": "voltage",
                "label": "Voltage",
                "type": "enum",
                "options": ["120V", "240V", "480V", "120/240V"],
                "required": true
            },
            {
                "key": "phase",
                "label": "Phase",
                "type": "enum",
                "options": ["single", "three"],
                "required": true
            },
            {
                "key": "noise_level_db",
                "label": "Noise Level (dB)",
                "type": "number",
                "required": false,
                "unit": "dB"
            },
            {
                "key": "is_towable",
                "label": "Towable",
                "type": "boolean",
                "required": false,
                "default": false
            }
        ]
    */
    -- JSONB: default inspection checklist items for this category
    default_inspection_items JSONB NOT NULL DEFAULT '[]'::jsonb,
    /*  default_inspection_items example:
        [
            {"label": "Engine oil level", "type": "pass_fail", "required": true},
            {"label": "Hydraulic fluid level", "type": "pass_fail", "required": true},
            {"label": "Tracks/tires condition", "type": "rating_1_5", "required": true},
            {"label": "Bucket/attachment condition", "type": "rating_1_5", "required": true},
            {"label": "Cab interior", "type": "pass_fail", "required": false},
            {"label": "Overall photo", "type": "photo_required", "required": true}
        ]
    */
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, code)
);

CREATE INDEX idx_eqcat_parent ON equipment_categories(parent_id);
```

### 3. Assets with Dynamic Attributes

```sql
CREATE TABLE assets (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    category_id         UUID NOT NULL REFERENCES equipment_categories(id),
    -- Typed columns: indexed, constrained, used in joins and calculations
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
    barcode             VARCHAR(100),
    last_latitude       DECIMAL(10, 7),
    last_longitude      DECIMAL(10, 7),
    last_location_at    TIMESTAMPTZ,
    is_active           BOOLEAN NOT NULL DEFAULT true,

    -- JSONB: category-specific technical specifications
    -- Validated against equipment_categories.attribute_schema
    specifications      JSONB NOT NULL DEFAULT '{}'::jsonb,
    /*  Example for an excavator (CAT 320):
        {
            "operating_weight_kg": 22000,
            "bucket_capacity_m3": 1.19,
            "max_dig_depth_m": 6.71,
            "boom_length_m": 5.7,
            "tail_swing": "conventional",
            "thumb_attachment": true
        }

        Example for a generator (CAT XQ230):
        {
            "power_kva": 230,
            "fuel_tank_capacity_l": 681,
            "voltage": "480V",
            "phase": "three",
            "noise_level_db": 71,
            "is_towable": true
        }
    */

    -- JSONB: attachments and accessories currently with this asset
    attachments         JSONB NOT NULL DEFAULT '[]'::jsonb,
    /*  Example:
        [
            {"name": "36-inch bucket", "serial": "BK-2024-881", "type": "bucket"},
            {"name": "hydraulic thumb", "serial": "TH-2023-442", "type": "attachment"},
            {"name": "quick coupler", "serial": "QC-2024-019", "type": "coupler"}
        ]
    */

    -- JSONB: insurance and registration details (varies by jurisdiction)
    compliance          JSONB NOT NULL DEFAULT '{}'::jsonb,
    /*  Example:
        {
            "insurance": {
                "policy_number": "EQ-2026-44821",
                "insurer": "Hartford Equipment Insurance",
                "coverage_amount": 250000,
                "expiry_date": "2027-03-15"
            },
            "registration": {
                "number": "AZ-EQ-88412",
                "state": "AZ",
                "expiry_date": "2026-12-31"
            },
            "certifications": [
                {
                    "type": "annual_inspection",
                    "issued_date": "2026-01-15",
                    "expiry_date": "2027-01-15",
                    "inspector": "ABC Equipment Services"
                }
            ]
        }
    */

    -- JSONB: depreciation and financial tracking
    financials          JSONB NOT NULL DEFAULT '{}'::jsonb,
    /*  Example:
        {
            "depreciation_method": "straight_line",
            "useful_life_months": 120,
            "salvage_value": 15000,
            "monthly_depreciation": 1958.33,
            "accumulated_depreciation": 23500,
            "book_value": 211500,
            "replacement_cost": 285000
        }
    */

    -- JSONB: custom fields defined per organization
    custom_fields       JSONB NOT NULL DEFAULT '{}'::jsonb,
    /*  Example:
        {
            "internal_project_code": "PROJ-2026-EAST",
            "preferred_operator": "Mike Johnson",
            "gps_device_imei": "358673011223344"
        }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, asset_number)
);

-- Standard indexes on typed columns
CREATE INDEX idx_assets_org_status ON assets(organization_id, status);
CREATE INDEX idx_assets_category ON assets(category_id);
CREATE INDEX idx_assets_current_depot ON assets(current_depot_id);
CREATE INDEX idx_assets_serial ON assets(serial_number) WHERE serial_number IS NOT NULL;

-- GIN indexes on JSONB columns for attribute-based queries
CREATE INDEX idx_assets_specs ON assets USING GIN (specifications jsonb_path_ops);
CREATE INDEX idx_assets_compliance ON assets USING GIN (compliance jsonb_path_ops);

-- Functional indexes for frequently queried JSONB paths
CREATE INDEX idx_assets_insurance_expiry ON assets (
    (compliance->'insurance'->>'expiry_date')
) WHERE compliance->'insurance'->>'expiry_date' IS NOT NULL;

-- Asset photos and documents (normalized -- relationship data)
CREATE TABLE asset_photos (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID NOT NULL REFERENCES assets(id) ON DELETE CASCADE,
    photo_url       VARCHAR(500) NOT NULL,
    caption         VARCHAR(255),
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    taken_at        TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_asset_photos ON asset_photos(asset_id);
```

### 4. Customer Management (Normalized + JSONB for Custom Fields)

```sql
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
    mobile              VARCHAR(30),
    -- Typed billing address (used in invoices)
    billing_address_line1 VARCHAR(255),
    billing_address_line2 VARCHAR(255),
    billing_city        VARCHAR(100),
    billing_state       VARCHAR(100),
    billing_postal_code VARCHAR(20),
    billing_country     CHAR(2) DEFAULT 'US',
    -- Financial columns (used in calculations and constraints)
    tax_exempt          BOOLEAN NOT NULL DEFAULT false,
    tax_id              VARCHAR(50),
    credit_limit        DECIMAL(12, 2),
    payment_terms_days  INT NOT NULL DEFAULT 30,
    account_status      VARCHAR(20) NOT NULL DEFAULT 'active',

    -- JSONB: multiple contacts with varying roles
    contacts            JSONB NOT NULL DEFAULT '[]'::jsonb,
    /*  Example:
        [
            {
                "id": "uuid-1",
                "first_name": "Sarah",
                "last_name": "Chen",
                "email": "sarah@acmeconstruction.com",
                "phone": "+1-480-555-0101",
                "job_title": "Fleet Manager",
                "is_primary": true,
                "can_sign_contracts": true
            },
            {
                "id": "uuid-2",
                "first_name": "Tom",
                "last_name": "Rodriguez",
                "email": "tom@acmeconstruction.com",
                "phone": "+1-480-555-0102",
                "job_title": "Site Foreman",
                "is_primary": false,
                "can_sign_contracts": false,
                "receives_dispatch_notifications": true
            }
        ]
    */

    -- JSONB: delivery site addresses (multiple, varying detail)
    sites               JSONB NOT NULL DEFAULT '[]'::jsonb,
    /*  Example:
        [
            {
                "id": "uuid-a",
                "name": "Phoenix Main Site",
                "address": "4200 N Central Ave, Phoenix, AZ 85012",
                "latitude": 33.4942,
                "longitude": -112.0740,
                "contact_name": "Tom Rodriguez",
                "contact_phone": "+1-480-555-0102",
                "access_notes": "Gate code: 4488. Enter from north side.",
                "is_active": true
            }
        ]
    */

    -- JSONB: insurance certificates (expire, need tracking)
    insurance_certificates JSONB NOT NULL DEFAULT '[]'::jsonb,
    /*  Example:
        [
            {
                "id": "uuid-x",
                "insurer": "State Farm Commercial",
                "policy_number": "CF-2026-887412",
                "coverage_type": "General Liability",
                "coverage_amount": 2000000,
                "effective_date": "2026-01-01",
                "expiry_date": "2027-01-01",
                "document_url": "/docs/insurance/uuid-x.pdf",
                "verified": true,
                "verified_by": "uuid-user",
                "verified_at": "2026-01-05T14:30:00Z"
            }
        ]
    */

    -- JSONB: organization-specific custom fields
    custom_fields       JSONB NOT NULL DEFAULT '{}'::jsonb,

    portal_enabled      BOOLEAN NOT NULL DEFAULT false,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, customer_number)
);

CREATE INDEX idx_customers_org ON customers(organization_id);
CREATE INDEX idx_customers_email ON customers(email) WHERE email IS NOT NULL;
CREATE INDEX idx_customers_name ON customers(organization_id, company_name, last_name);
-- GIN index for querying inside contacts and insurance arrays
CREATE INDEX idx_customers_contacts ON customers USING GIN (contacts jsonb_path_ops);
CREATE INDEX idx_customers_insurance ON customers USING GIN (insurance_certificates jsonb_path_ops);
```

### 5. Rate Tables with Flexible Rule Configurations

```sql
CREATE TABLE rate_tables (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    is_default      BOOLEAN NOT NULL DEFAULT false,
    effective_from  DATE NOT NULL,
    effective_to    DATE,

    -- JSONB: complex pricing rules that vary enormously by operator
    pricing_rules   JSONB NOT NULL DEFAULT '{}'::jsonb,
    /*  Example:
        {
            "billing_calendar": "business_days",
            "weekend_handling": "free",
            "holiday_calendar": "US_federal",
            "minimum_rental_period": {"value": 1, "unit": "day"},
            "late_return_penalty": {
                "grace_period_hours": 4,
                "penalty_type": "additional_day",
                "penalty_multiplier": 1.0
            },
            "volume_discounts": [
                {"min_items": 3, "discount_pct": 5},
                {"min_items": 5, "discount_pct": 10},
                {"min_items": 10, "discount_pct": 15}
            ],
            "long_term_discounts": [
                {"min_days": 7, "discount_pct": 5},
                {"min_days": 30, "discount_pct": 15},
                {"min_days": 90, "discount_pct": 25}
            ],
            "shift_pricing": {
                "enabled": true,
                "standard_shift_hours": 8,
                "overtime_multiplier": 1.5,
                "double_time_after_hours": 12
            },
            "fuel_surcharge": {
                "enabled": true,
                "method": "refill_charge",
                "rate_per_gallon": 6.50
            }
        }
    */

    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rate_tables_org ON rate_tables(organization_id);

-- Rate entries: per-category or per-asset base rates (normalized)
CREATE TABLE rate_entries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rate_table_id   UUID NOT NULL REFERENCES rate_tables(id) ON DELETE CASCADE,
    category_id     UUID REFERENCES equipment_categories(id),
    asset_id        UUID REFERENCES assets(id),
    hourly_rate     DECIMAL(10, 2),
    daily_rate      DECIMAL(10, 2),
    weekly_rate     DECIMAL(10, 2),
    monthly_rate    DECIMAL(10, 2),
    -- JSONB: additional rate tiers and special pricing
    special_rates   JSONB NOT NULL DEFAULT '{}'::jsonb,
    /*  Example:
        {
            "weekend_daily_rate": 180.00,
            "holiday_daily_rate": 250.00,
            "operator_hourly_rate": 85.00,
            "delivery_base_fee": 150.00,
            "delivery_per_km": 3.50,
            "damage_waiver_daily": 25.00,
            "environmental_fee_daily": 5.00
        }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT chk_rate_target CHECK (
        (category_id IS NOT NULL AND asset_id IS NULL)
        OR (category_id IS NULL AND asset_id IS NOT NULL)
    )
);

CREATE INDEX idx_rate_entries_table ON rate_entries(rate_table_id);
CREATE INDEX idx_rate_entries_category ON rate_entries(category_id);

-- Customer-specific rate overrides
CREATE TABLE customer_rate_overrides (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    rate_table_id   UUID REFERENCES rate_tables(id),
    -- JSONB: flexible customer-specific pricing adjustments
    overrides       JSONB NOT NULL DEFAULT '{}'::jsonb,
    /*  Example:
        {
            "global_discount_pct": 10,
            "category_discounts": {
                "excavators": 12,
                "generators": 8
            },
            "free_delivery_radius_km": 40,
            "waive_fuel_surcharge": true,
            "custom_payment_terms_days": 45,
            "volume_pricing_tier": "premium"
        }
    */
    effective_from  DATE NOT NULL,
    effective_to    DATE,
    approved_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (customer_id, effective_from)
);
```

### 6. Rental Contracts (Normalized Core + JSONB Details)

```sql
CREATE TABLE rental_contracts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    contract_number VARCHAR(50) NOT NULL,
    customer_id     UUID NOT NULL REFERENCES customers(id),
    depot_id        UUID NOT NULL REFERENCES depots(id),
    -- Core typed columns: used in queries, reports, constraints
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
    rental_start    TIMESTAMPTZ NOT NULL,
    rental_end_expected TIMESTAMPTZ,
    rental_end_actual   TIMESTAMPTZ,
    rate_table_id   UUID REFERENCES rate_tables(id),
    deposit_amount  DECIMAL(12, 2) NOT NULL DEFAULT 0,
    deposit_paid    BOOLEAN NOT NULL DEFAULT false,
    subtotal        DECIMAL(12, 2) NOT NULL DEFAULT 0,
    tax_amount      DECIMAL(12, 2) NOT NULL DEFAULT 0,
    damage_charges  DECIMAL(12, 2) NOT NULL DEFAULT 0,
    extra_charges   DECIMAL(12, 2) NOT NULL DEFAULT 0,
    total           DECIMAL(12, 2) NOT NULL DEFAULT 0,

    -- JSONB: delivery and site details (snapshot at contract time)
    delivery_details JSONB NOT NULL DEFAULT '{}'::jsonb,
    /*  Example:
        {
            "method": "delivery",
            "site_name": "Phoenix Main Site",
            "address": "4200 N Central Ave, Phoenix, AZ 85012",
            "latitude": 33.4942,
            "longitude": -112.0740,
            "contact_name": "Tom Rodriguez",
            "contact_phone": "+1-480-555-0102",
            "access_notes": "Gate code: 4488",
            "delivery_date": "2026-06-01T08:00:00Z",
            "return_method": "pickup",
            "return_date": null
        }
    */

    -- JSONB: customer and contact snapshot at contract time
    -- (frozen copy so changes to customer record don't affect contract)
    customer_snapshot JSONB NOT NULL DEFAULT '{}'::jsonb,
    /*  Example:
        {
            "company_name": "Acme Construction LLC",
            "contact_name": "Sarah Chen",
            "contact_email": "sarah@acmeconstruction.com",
            "contact_phone": "+1-480-555-0101",
            "billing_address": "1200 W Washington St, Phoenix, AZ 85007",
            "tax_id": "87-4421889",
            "tax_exempt": false
        }
    */

    -- JSONB: e-signature details
    signature_details JSONB,
    /*  Example:
        {
            "signed_by_name": "Sarah Chen",
            "signed_at": "2026-05-28T14:22:00Z",
            "signature_url": "/signatures/contract-uuid.png",
            "ip_address": "72.134.88.201",
            "user_agent": "Mozilla/5.0 ...",
            "terms_version": "2026-01",
            "esign_provider": "internal"
        }
    */

    -- JSONB: terms and conditions snapshot
    terms           JSONB NOT NULL DEFAULT '{}'::jsonb,

    notes           TEXT,
    internal_notes  TEXT,
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, contract_number)
);

CREATE INDEX idx_contracts_org_status ON rental_contracts(organization_id, status);
CREATE INDEX idx_contracts_customer ON rental_contracts(customer_id);
CREATE INDEX idx_contracts_dates ON rental_contracts(rental_start, rental_end_expected);

-- Contract line items (normalized -- each asset on the contract)
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
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    dispatched_at   TIMESTAMPTZ,
    returned_at     TIMESTAMPTZ,
    -- JSONB: meter readings and fuel levels at dispatch/return
    dispatch_readings JSONB,
    /*  Example:
        {
            "hour_meter": 4521.3,
            "odometer_km": null,
            "fuel_level_pct": 95,
            "condition_grade": "good",
            "dispatched_by": "uuid-user",
            "dispatch_photos": ["url1", "url2"]
        }
    */
    return_readings JSONB,
    /*  Example:
        {
            "hour_meter": 4589.7,
            "odometer_km": null,
            "fuel_level_pct": 42,
            "condition_grade": "good",
            "hours_used": 68.4,
            "returned_by": "uuid-user",
            "return_photos": ["url3", "url4"]
        }
    */
    sort_order      INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_contract_lines_contract ON contract_lines(contract_id);
CREATE INDEX idx_contract_lines_asset ON contract_lines(asset_id);

-- Contract extras (normalized -- financial data)
CREATE TABLE contract_extras (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id     UUID NOT NULL REFERENCES rental_contracts(id) ON DELETE CASCADE,
    contract_line_id UUID REFERENCES contract_lines(id),
    charge_code     VARCHAR(50) NOT NULL,
    description     VARCHAR(500),
    quantity        DECIMAL(10, 2) NOT NULL DEFAULT 1,
    unit_amount     DECIMAL(10, 2) NOT NULL,
    total_amount    DECIMAL(12, 2) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Contract history (simple audit trail without full event sourcing)
CREATE TABLE contract_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id     UUID NOT NULL REFERENCES rental_contracts(id),
    action          VARCHAR(50) NOT NULL,
    -- JSONB: what changed
    changes         JSONB NOT NULL,
    /*  Example:
        {
            "action": "extended",
            "previous_end": "2026-06-15T17:00:00Z",
            "new_end": "2026-06-22T17:00:00Z",
            "reason": "Customer requested 1-week extension",
            "approved_by": "uuid-user"
        }
    */
    performed_by    UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_contract_history ON contract_history(contract_id, created_at DESC);
```

### 7. Inspections (JSONB Checklists)

```sql
CREATE TABLE inspections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_line_id UUID REFERENCES contract_lines(id),
    asset_id        UUID NOT NULL REFERENCES assets(id),
    inspection_type VARCHAR(20) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'in_progress',
    inspector_id    UUID NOT NULL REFERENCES users(id),
    inspected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    overall_condition VARCHAR(20),

    -- JSONB: checklist results (flexible per category)
    checklist       JSONB NOT NULL DEFAULT '[]'::jsonb,
    /*  Example:
        [
            {
                "label": "Engine oil level",
                "type": "pass_fail",
                "result": "pass",
                "notes": null
            },
            {
                "label": "Hydraulic fluid level",
                "type": "pass_fail",
                "result": "fail",
                "notes": "Low - needs top-up before next rental"
            },
            {
                "label": "Tracks condition",
                "type": "rating_1_5",
                "result": 4,
                "notes": "Minor wear, within acceptable range"
            },
            {
                "label": "Bucket teeth",
                "type": "rating_1_5",
                "result": 2,
                "notes": "Significant wear on two teeth - replacement recommended"
            },
            {
                "label": "Overall front photo",
                "type": "photo_required",
                "result": "/photos/insp-uuid/front.jpg",
                "notes": null
            }
        ]
    */

    -- JSONB: location and meter data at inspection time
    readings        JSONB NOT NULL DEFAULT '{}'::jsonb,
    /*  Example:
        {
            "hour_meter": 4589.7,
            "odometer_km": null,
            "fuel_level_pct": 42,
            "latitude": 33.4942,
            "longitude": -112.0740,
            "gps_accuracy_m": 3.2
        }
    */

    -- JSONB: photos (array of URLs with metadata)
    photos          JSONB NOT NULL DEFAULT '[]'::jsonb,
    /*  Example:
        [
            {"url": "/photos/insp-uuid/front.jpg", "caption": "Front view", "taken_at": "..."},
            {"url": "/photos/insp-uuid/damage1.jpg", "caption": "Scratch on right panel", "taken_at": "..."}
        ]
    */

    notes           TEXT,
    reviewed_by     UUID REFERENCES users(id),
    reviewed_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_inspections_asset ON inspections(asset_id);
CREATE INDEX idx_inspections_contract_line ON inspections(contract_line_id);

-- Damage records (normalized -- financial/legal significance)
CREATE TABLE damage_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    inspection_id   UUID NOT NULL REFERENCES inspections(id),
    asset_id        UUID NOT NULL REFERENCES assets(id),
    contract_line_id UUID REFERENCES contract_lines(id),
    description     TEXT NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    estimated_repair_cost DECIMAL(10, 2),
    charge_to_customer    DECIMAL(10, 2),
    charge_approved       BOOLEAN NOT NULL DEFAULT false,
    status          VARCHAR(20) NOT NULL DEFAULT 'reported',
    -- JSONB: photos, insurance claim details, repair history
    details         JSONB NOT NULL DEFAULT '{}'::jsonb,
    /*  Example:
        {
            "photos": {
                "pre_rental": ["/photos/pre/front.jpg"],
                "post_rental": ["/photos/post/front.jpg", "/photos/post/damage1.jpg"],
                "comparison": "/photos/ai-comparison/uuid.jpg"
            },
            "ai_assessment": {
                "model_version": "damage-detect-v2.1",
                "confidence": 0.87,
                "damage_type": "surface_scratch",
                "recommended_charge": 450.00,
                "assessed_at": "2026-06-10T09:15:00Z"
            },
            "insurance_claim": {
                "claim_reference": "CLM-2026-44821",
                "insurer": "Hartford Equipment Insurance",
                "filed_at": "2026-06-11T10:00:00Z",
                "settlement_amount": null,
                "status": "filed"
            },
            "repair": {
                "work_order_id": "uuid-wo",
                "repaired_at": "2026-06-15T16:00:00Z",
                "actual_cost": 380.00,
                "technician": "Mike's Equipment Repair"
            }
        }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_damage_asset ON damage_records(asset_id);
CREATE INDEX idx_damage_status ON damage_records(status);
```

### 8. Maintenance and Work Orders

```sql
-- Maintenance schedules with flexible rule definitions
CREATE TABLE maintenance_schedules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            VARCHAR(255) NOT NULL,
    category_id     UUID REFERENCES equipment_categories(id),
    asset_id        UUID REFERENCES assets(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- JSONB: flexible scheduling rules
    schedule_rules  JSONB NOT NULL,
    /*  Example:
        {
            "triggers": [
                {
                    "type": "hours",
                    "interval": 250,
                    "description": "Every 250 engine hours"
                },
                {
                    "type": "calendar",
                    "interval_days": 90,
                    "description": "Every 90 days (quarterly)"
                }
            ],
            "whichever_comes_first": true,
            "tasks": [
                "Change engine oil and filter",
                "Check hydraulic fluid level and condition",
                "Inspect all hoses for leaks and wear",
                "Grease all pivot points",
                "Check and adjust track tension"
            ],
            "estimated_duration_hours": 3,
            "estimated_cost": 450.00,
            "parts_needed": [
                {"part": "Engine oil filter", "part_number": "CAT-1R-0749"},
                {"part": "Engine oil 15W-40", "quantity": "12 quarts"},
                {"part": "Hydraulic filter", "part_number": "CAT-5I-8670"}
            ],
            "requires_specialist": false
        }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Work orders (normalized core + JSONB for variable details)
CREATE TABLE work_orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    work_order_number VARCHAR(50) NOT NULL,
    asset_id        UUID NOT NULL REFERENCES assets(id),
    depot_id        UUID NOT NULL REFERENCES depots(id),
    maintenance_schedule_id UUID REFERENCES maintenance_schedules(id),
    damage_record_id UUID REFERENCES damage_records(id),
    work_order_type VARCHAR(30) NOT NULL,
    priority        VARCHAR(20) NOT NULL DEFAULT 'normal',
    status          VARCHAR(20) NOT NULL DEFAULT 'open',
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    assigned_to     UUID REFERENCES users(id),
    scheduled_start TIMESTAMPTZ,
    scheduled_end   TIMESTAMPTZ,
    actual_start    TIMESTAMPTZ,
    actual_end      TIMESTAMPTZ,
    hour_meter_at   DECIMAL(12, 1),
    labor_cost      DECIMAL(10, 2) NOT NULL DEFAULT 0,
    parts_cost      DECIMAL(10, 2) NOT NULL DEFAULT 0,
    total_cost      DECIMAL(10, 2) NOT NULL DEFAULT 0,

    -- JSONB: labor entries, parts used, completion details
    work_details    JSONB NOT NULL DEFAULT '{}'::jsonb,
    /*  Example:
        {
            "labor": [
                {
                    "technician_id": "uuid",
                    "technician_name": "Dave Wilson",
                    "date": "2026-06-12",
                    "hours": 2.5,
                    "hourly_rate": 75.00,
                    "description": "Oil change and hydraulic inspection"
                }
            ],
            "parts": [
                {
                    "part_number": "CAT-1R-0749",
                    "name": "Engine oil filter",
                    "quantity": 1,
                    "unit_cost": 28.50,
                    "total_cost": 28.50,
                    "supplier": "Caterpillar Parts Direct"
                },
                {
                    "part_number": "OIL-15W40-QT",
                    "name": "Engine oil 15W-40 (quart)",
                    "quantity": 12,
                    "unit_cost": 8.75,
                    "total_cost": 105.00,
                    "supplier": "Local Supply Co"
                }
            ],
            "completion_notes": "Oil changed. Hydraulic hose on boom cylinder showing early signs of wear - flagged for next service.",
            "follow_up_required": true,
            "follow_up_description": "Replace boom cylinder hydraulic hose at next PM interval"
        }
    */

    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, work_order_number)
);

CREATE INDEX idx_wo_asset ON work_orders(asset_id);
CREATE INDEX idx_wo_status ON work_orders(organization_id, status);
CREATE INDEX idx_wo_assigned ON work_orders(assigned_to) WHERE assigned_to IS NOT NULL;
```

### 9. Telematics (Normalized Time-Series + JSONB Raw Payload)

```sql
CREATE TABLE telematics_providers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    provider_type   VARCHAR(50) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    -- JSONB: provider-specific configuration and credentials
    config          JSONB NOT NULL DEFAULT '{}'::jsonb,
    /*  Example for Geotab:
        {
            "api_endpoint": "https://my.geotab.com/apiv1",
            "database": "acme_rental",
            "credentials_secret_ref": "vault://geotab/acme",
            "sync_interval_seconds": 300,
            "device_filter_groups": ["Construction Fleet"],
            "data_points": ["Position", "EngineHours", "FuelLevel"]
        }
    */
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE asset_telematics_links (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID NOT NULL REFERENCES assets(id),
    provider_id     UUID NOT NULL REFERENCES telematics_providers(id),
    device_id       VARCHAR(255) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    linked_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (provider_id, device_id)
);

-- Telematics readings: typed columns for common data, JSONB for raw
CREATE TABLE telematics_readings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID NOT NULL REFERENCES assets(id),
    provider_id     UUID NOT NULL REFERENCES telematics_providers(id),
    recorded_at     TIMESTAMPTZ NOT NULL,
    received_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- Typed: the data points we actually query and report on
    latitude        DECIMAL(10, 7),
    longitude       DECIMAL(10, 7),
    engine_hours    DECIMAL(12, 1),
    odometer_km     DECIMAL(12, 1),
    fuel_level_pct  DECIMAL(5, 2),
    engine_status   VARCHAR(20),
    speed_kmh       DECIMAL(6, 1),
    -- JSONB: full raw payload from provider (debugging, future use)
    raw_data        JSONB
    /*  Example raw_data from Geotab:
        {
            "deviceId": "b1234",
            "dateTime": "2026-06-01T14:30:00Z",
            "latitude": 33.4942,
            "longitude": -112.0740,
            "speed": 0,
            "engineHours": 4589.7,
            "fuelUsed": 12847.3,
            "currentStateDuration": "PT2H15M",
            "diagnosticCodes": [],
            "isDeviceCommunicating": true,
            "batteryVoltage": 12.8
        }
    */
) PARTITION BY RANGE (recorded_at);

CREATE TABLE telematics_readings_2026_q1 PARTITION OF telematics_readings
    FOR VALUES FROM ('2026-01-01') TO ('2026-04-01');
CREATE TABLE telematics_readings_2026_q2 PARTITION OF telematics_readings
    FOR VALUES FROM ('2026-04-01') TO ('2026-07-01');
CREATE TABLE telematics_readings_2026_q3 PARTITION OF telematics_readings
    FOR VALUES FROM ('2026-07-01') TO ('2026-10-01');
CREATE TABLE telematics_readings_2026_q4 PARTITION OF telematics_readings
    FOR VALUES FROM ('2026-10-01') TO ('2027-01-01');

CREATE INDEX idx_telem_asset_time ON telematics_readings(asset_id, recorded_at DESC);
CREATE INDEX idx_telem_recorded ON telematics_readings(recorded_at DESC);
```

### 10. Invoicing and Payments (Fully Normalized)

```sql
-- Invoices: all typed columns (financial data, never JSONB)
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
    period_start    DATE,
    period_end      DATE,
    subtotal        DECIMAL(12, 2) NOT NULL DEFAULT 0,
    tax_amount      DECIMAL(12, 2) NOT NULL DEFAULT 0,
    total           DECIMAL(12, 2) NOT NULL DEFAULT 0,
    amount_paid     DECIMAL(12, 2) NOT NULL DEFAULT 0,
    balance_due     DECIMAL(12, 2) NOT NULL DEFAULT 0,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    notes           TEXT,
    -- JSONB: accounting sync metadata
    accounting_sync JSONB NOT NULL DEFAULT '{}'::jsonb,
    /*  Example:
        {
            "quickbooks": {
                "id": "1234",
                "synced_at": "2026-06-01T10:00:00Z",
                "sync_status": "success"
            }
        }
    */
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
    sort_order      INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE payments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    invoice_id      UUID NOT NULL REFERENCES invoices(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    payment_method  VARCHAR(30) NOT NULL,
    amount          DECIMAL(12, 2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    payment_date    DATE NOT NULL DEFAULT CURRENT_DATE,
    reference       VARCHAR(100),
    status          VARCHAR(20) NOT NULL DEFAULT 'completed',
    -- JSONB: payment gateway details
    gateway_details JSONB NOT NULL DEFAULT '{}'::jsonb,
    /*  Example:
        {
            "provider": "stripe",
            "payment_intent_id": "pi_3abc...",
            "charge_id": "ch_3def...",
            "card_last4": "4242",
            "card_brand": "visa"
        }
    */
    recorded_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payments_invoice ON payments(invoice_id);
CREATE INDEX idx_payments_customer ON payments(customer_id);
```

### 11. Notifications and Audit

```sql
CREATE TABLE notifications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    recipient_type  VARCHAR(20) NOT NULL,
    recipient_id    UUID NOT NULL,
    channel         VARCHAR(20) NOT NULL,
    notification_type VARCHAR(50) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    -- JSONB: rendered content and delivery metadata
    content         JSONB NOT NULL,
    /*  Example:
        {
            "subject": "Rental Contract RC-2026-0142 - Equipment Delivery Tomorrow",
            "body_text": "Your rental of CAT 320 Excavator ...",
            "body_html": "<html>...",
            "template_name": "dispatch_reminder",
            "template_vars": {
                "customer_name": "Acme Construction",
                "equipment": "CAT 320 Excavator",
                "delivery_date": "2026-06-01",
                "delivery_window": "8:00 AM - 10:00 AM"
            }
        }
    */
    sent_at         TIMESTAMPTZ,
    error_message   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_notifications_status ON notifications(status) WHERE status = 'pending';

-- Audit log with JSONB change tracking
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    user_id         UUID REFERENCES users(id),
    action          VARCHAR(50) NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    changes         JSONB NOT NULL,
    /*  Example:
        {
            "before": {"status": "draft", "total": 0},
            "after": {"status": "active", "total": 4250.00},
            "reason": "Contract signed by customer"
        }
    */
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_created ON audit_log(created_at DESC);
```

---

## Query Examples

### Find all excavators over 20 tons available at the Phoenix depot

```sql
SELECT a.asset_number, a.make, a.model, a.year,
       a.specifications->>'operating_weight_kg' AS weight_kg,
       a.specifications->>'bucket_capacity_m3' AS bucket_m3,
       a.specifications->>'max_dig_depth_m' AS dig_depth_m
FROM assets a
JOIN equipment_categories c ON a.category_id = c.id
JOIN depots d ON a.current_depot_id = d.id
WHERE c.code = 'excavators'
AND a.status = 'available'
AND d.name = 'Phoenix Depot'
AND (a.specifications->>'operating_weight_kg')::numeric > 20000
ORDER BY (a.specifications->>'operating_weight_kg')::numeric;
```

### Find customers with insurance expiring in the next 30 days

```sql
SELECT c.customer_number, c.company_name,
       cert->>'insurer' AS insurer,
       cert->>'policy_number' AS policy_number,
       (cert->>'expiry_date')::date AS expiry_date
FROM customers c,
     jsonb_array_elements(c.insurance_certificates) AS cert
WHERE c.organization_id = 'uuid'
AND c.account_status = 'active'
AND (cert->>'expiry_date')::date BETWEEN CURRENT_DATE AND CURRENT_DATE + INTERVAL '30 days'
ORDER BY (cert->>'expiry_date')::date;
```

### Get rate with all applicable discounts for a customer

```sql
SELECT re.daily_rate,
       re.weekly_rate,
       re.monthly_rate,
       re.special_rates->>'damage_waiver_daily' AS damage_waiver,
       cro.overrides->>'global_discount_pct' AS customer_discount_pct,
       rt.pricing_rules->'long_term_discounts' AS long_term_discounts,
       rt.pricing_rules->'shift_pricing' AS shift_pricing
FROM rate_entries re
JOIN rate_tables rt ON re.rate_table_id = rt.id
LEFT JOIN customer_rate_overrides cro ON cro.rate_table_id = rt.id
    AND cro.customer_id = 'customer-uuid'
    AND cro.effective_from <= CURRENT_DATE
    AND (cro.effective_to IS NULL OR cro.effective_to >= CURRENT_DATE)
WHERE re.category_id = 'excavator-category-uuid'
AND rt.is_active = true
AND rt.effective_from <= CURRENT_DATE
AND (rt.effective_to IS NULL OR rt.effective_to >= CURRENT_DATE);
```

---

## Pros and Cons

### Pros

1. **Equipment attribute variability is solved cleanly.** The `specifications` JSONB column on `assets` absorbs the enormous variation between equipment types -- excavators, generators, compressors, light towers, scaffolding, trucks -- without schema migrations. New equipment categories with entirely different specs can be added by updating the `attribute_schema` on the category, not by altering the assets table. This is the single biggest improvement over the pure normalized approach.

2. **Rate table flexibility without schema explosion.** Rental billing rules are notoriously complex and vary enormously between operators. Shift pricing, volume discounts, holiday calendars, fuel surcharges, late-return penalties -- these would require 10+ additional tables in a normalized schema, most of which would be sparsely populated. The JSONB `pricing_rules` column captures all of this with room for operator-specific customizations.

3. **Inspection checklists adapt per category without extra tables.** A generator inspection checks voltage output and fuel tank integrity; an excavator inspection checks track tension and hydraulic hose condition. The `checklist` JSONB array on inspections absorbs this variability while still being fully queryable.

4. **Financial data remains fully constrained.** Invoice totals, payment amounts, deposit balances -- everything financial stays in typed DECIMAL columns with standard SQL constraints. JSONB is never used for data that participates in financial calculations or regulatory reporting.

5. **Simpler schema with fewer tables.** Compared to the normalized model (Suggestion 1), this schema has roughly 30% fewer tables. Customer contacts, sites, and insurance certificates are JSONB arrays instead of separate tables. Work order labor and parts are JSONB inside the work order. This reduces join complexity for common read operations.

6. **Evolution without migrations.** When a new equipment specification field is needed, or a new rate rule type is invented, no ALTER TABLE is required. The JSONB column absorbs the change. This is particularly valuable for a product with many tenants who each want custom fields.

7. **Full PostgreSQL ecosystem.** Unlike a document database (MongoDB), this model runs on PostgreSQL with full ACID transactions, foreign key constraints, PostGIS support, and access to every PostgreSQL extension and tool.

### Cons

1. **JSONB data is not schema-enforced at the database level.** PostgreSQL does not validate that the `specifications` JSONB matches the `attribute_schema` defined on the category. A developer could write `{"operating_weight_kg": "twenty tons"}` (string instead of number) and the database would accept it. Schema validation must be enforced in the application layer.

2. **JSONB queries are slower than typed column queries for complex aggregations.** `WHERE (specifications->>'operating_weight_kg')::numeric > 20000` requires a cast and cannot use a standard B-tree index. GIN indexes help for containment queries (`@>`) but not for range queries on extracted values. Functional indexes can be created for specific paths, but each requires anticipating the query pattern.

3. **Reporting tools may struggle with JSONB.** While PostgreSQL natively supports JSONB extraction in SQL, many BI tools (Metabase, Tableau, Power BI) work best with flat relational tables. Reporting queries against JSONB columns require JSONB path extraction functions (`->>`, `jsonb_array_elements`) that are less intuitive for report builders and analysts.

4. **JSONB arrays lose referential integrity.** Customer contacts stored as a JSONB array inside the customers table cannot have foreign keys pointing to them. If a contract references a contact by their JSONB `id`, there is no database-level guarantee that the contact still exists or that the ID is valid.

5. **Document size growth risk.** If damage records, inspection histories, or work order details accumulate in JSONB arrays, individual rows can grow large. PostgreSQL handles TOAST compression for oversized values, but very large JSONB documents (>1MB) can degrade UPDATE performance because the entire JSONB value is rewritten on each change.

6. **Harder to enforce uniqueness within JSONB.** Ensuring no two customer contacts have the same email address within a customer's contacts array requires application-level validation or a partial unique index with complex expression -- neither is as clean as a UNIQUE constraint on a normalized table.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Database | PostgreSQL 16+ |
| Extensions | PostGIS (spatial), pg_cron (partition management), pgcrypto |
| JSONB validation | JSON Schema (ajv for TypeScript, jsonschema for Python) at application layer |
| ORM | Prisma (excellent JSONB support in TypeScript), Drizzle, or SQLAlchemy |
| Migration tool | Prisma Migrate, Flyway, or golang-migrate |
| JSONB indexing | GIN with jsonb_path_ops for containment queries; functional B-tree indexes for range queries on specific paths |
| BI/Reporting | dbt for transforming JSONB into flat reporting tables; Metabase or Apache Superset for dashboards |
| Hosting | Same as Suggestion 1: AWS RDS, Google Cloud SQL, Neon, Supabase |

---

## Migration and Scaling Considerations

### JSONB Schema Versioning

Each JSONB column should include a `_schema_version` field to track format changes:

```json
{
    "_schema_version": 2,
    "operating_weight_kg": 22000,
    "bucket_capacity_m3": 1.19
}
```

When the schema evolves, the application reads `_schema_version` and applies transformation logic for older formats. This is simpler than event upcasting (Suggestion 2) because it is per-document rather than per-event-stream.

### Scaling Path

- **Phase 1 (MVP):** Single PostgreSQL instance. JSONB columns handle attribute variability. Application-layer validation with JSON Schema.
- **Phase 2 (Growth):** Add GIN and functional indexes as query patterns crystallize. Create materialized views that flatten JSONB into relational columns for reporting. Read replica for analytics.
- **Phase 3 (Scale):** Extract high-query-volume JSONB fields into typed columns when the query pattern is proven stable (e.g., `operating_weight_kg` might graduate from JSONB to a typed column if it appears in 80% of search queries). This is a one-way migration that trades flexibility for performance.

### Migration from Pure Relational (Suggestion 1)

If starting with the normalized model and migrating to the hybrid:
1. Add JSONB columns to existing tables (`ALTER TABLE assets ADD COLUMN specifications JSONB DEFAULT '{}'`).
2. Migrate existing typed columns into JSONB with a backfill script.
3. Keep the original typed columns as a read-through cache during transition.
4. Drop the original columns after all application code uses JSONB.
5. Drop normalized child tables (e.g., `inspection_items`, `work_order_parts`) after migrating their data into parent JSONB columns.

---

## Summary

The hybrid relational + JSONB model is the pragmatic middle ground. It preserves relational integrity for financial transactions, contracts, and entity relationships while using JSONB to absorb the variability that makes pure normalization painful in the equipment rental domain: diverse equipment specifications, flexible rate rules, adaptable inspection checklists, and evolving integration payloads. The trade-off is that JSONB data lacks database-level schema enforcement and can be harder to query for reporting -- both manageable with application-level validation and dbt-based reporting transformations. For a team building an open-source rental platform that must support diverse equipment types and operator-specific configurations without constant schema migrations, this is likely the best starting point.
