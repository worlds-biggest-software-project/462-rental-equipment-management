# Data Model Suggestion 2: Event-Sourced / CQRS Model

> Project: Rental Equipment Management (462)
> Model Type: Event Sourcing with Command Query Responsibility Segregation (CQRS)
> Generated: 2026-05-25

---

## Overview

This model treats every state change in the rental equipment management system as an immutable event. Instead of storing the current state of an asset, contract, or invoice and overwriting it on each update, the system appends a new event to an append-only event store. Current state is derived by replaying the event stream for a given aggregate. Read-optimized projections (materialized views of the event stream) serve queries and reporting.

This architecture is particularly relevant for rental equipment management because:

1. **Complete audit trail is a business requirement.** Billing disputes, damage claims, and insurance submissions require provable, tamper-evident history of every state change -- who changed what, when, and why.
2. **Asset lifecycle is inherently event-driven.** An asset moves through purchased, available, reserved, dispatched, on-rent, returned, inspected, in-maintenance, and retired states. Each transition is a discrete business event.
3. **Concurrent booking conflicts are critical failures.** Event sourcing with optimistic concurrency control prevents double-bookings at the data layer rather than relying on application-level locks.
4. **Temporal queries are first-class.** "What was the status of excavator EX-042 at 3pm on Tuesday?" is answered by replaying events up to that timestamp -- no reconstruction from audit logs needed.

---

## Architecture Overview

```
                  ┌─────────────┐
  Commands ──────>│ Command     │──────> Event Store (append-only)
  (writes)        │ Handlers    │              │
                  └─────────────┘              │
                                               ▼
                                        ┌──────────────┐
                                        │  Event Bus   │
                                        │  (async)     │
                                        └──────┬───────┘
                                               │
                          ┌────────────────────┼──────────────────────┐
                          ▼                    ▼                      ▼
                  ┌──────────────┐    ┌──────────────┐     ┌──────────────┐
                  │ Availability │    │  Billing     │     │  Fleet       │
                  │ Projection   │    │  Projection  │     │  Analytics   │
                  └──────────────┘    └──────────────┘     │  Projection  │
                          │                    │           └──────────────┘
                          ▼                    ▼                      │
                  ┌──────────────┐    ┌──────────────┐               ▼
  Queries ───────>│ Read Models  │    │ Read Models  │     ┌──────────────┐
  (reads)         │ (PostgreSQL) │    │ (PostgreSQL) │     │ Read Models  │
                  └──────────────┘    └──────────────┘     └──────────────┘
```

---

## Event Store Schema (PostgreSQL)

The event store is the single source of truth. All other tables are derived projections that can be rebuilt from the event stream at any time.

```sql
-- ============================================================
-- EVENT STORE: Core Tables
-- ============================================================

-- Global event log: every state change in the system
CREATE TABLE events (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,          -- aggregate root ID
    stream_type     VARCHAR(50) NOT NULL,   -- aggregate type
        -- Asset, RentalContract, Customer, Invoice, WorkOrder,
        -- Depot, DeliveryOrder, Inspection
    event_type      VARCHAR(100) NOT NULL,  -- specific event name
    event_version   INT NOT NULL,           -- per-stream sequence number
    event_data      JSONB NOT NULL,         -- event payload
    metadata        JSONB NOT NULL DEFAULT '{}',
        -- { user_id, correlation_id, causation_id, ip_address,
        --   organization_id, depot_id, source }
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, event_version)  -- optimistic concurrency control
) PARTITION BY RANGE (occurred_at);

-- Partitions by month for manageability
CREATE TABLE events_2026_01 PARTITION OF events
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE events_2026_02 PARTITION OF events
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
-- ... additional partitions created by automation

CREATE INDEX idx_events_stream ON events(stream_id, event_version);
CREATE INDEX idx_events_type ON events(stream_type, event_type);
CREATE INDEX idx_events_occurred ON events(occurred_at DESC);
CREATE INDEX idx_events_metadata_org ON events USING GIN ((metadata->'organization_id'));
CREATE INDEX idx_events_correlation ON events USING GIN ((metadata->'correlation_id'));

-- Snapshot store: periodic snapshots to avoid full replay
CREATE TABLE snapshots (
    stream_id       UUID NOT NULL,
    stream_type     VARCHAR(50) NOT NULL,
    snapshot_version INT NOT NULL,        -- event_version at snapshot time
    snapshot_data   JSONB NOT NULL,       -- serialized aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, snapshot_version)
);

-- Projection checkpoints: tracks last processed event per projection
CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_occurred_at TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Event Type Catalog

### Asset Lifecycle Events

```
AssetRegistered
    { asset_number, serial_number, make, model, year, category_id,
      purchase_date, purchase_cost, home_depot_id, fuel_type }

AssetDetailsUpdated
    { fields_changed: { make: { old, new }, ... } }

AssetCategoryChanged
    { old_category_id, new_category_id, reason }

AssetTransferredBetweenDepots
    { from_depot_id, to_depot_id, reason, transfer_id }

AssetTransferCompleted
    { transfer_id, arrived_at }

AssetPlacedInMaintenance
    { work_order_id, maintenance_type, estimated_return }

AssetReturnedFromMaintenance
    { work_order_id, hour_meter, condition_grade }

AssetMarkedOutOfService
    { reason, estimated_return }

AssetReturnedToService
    { condition_grade }

AssetRetired
    { reason, disposal_method, disposal_value }

AssetMeterReadingUpdated
    { hour_meter, odometer, source: "telematics"|"manual" }

AssetLocationUpdated
    { latitude, longitude, speed_kmh, heading, source }

AssetInsuranceUpdated
    { policy_number, insurer, expiry_date }

AssetPhotoAdded
    { photo_url, caption, is_primary }

AssetDocumentAttached
    { document_type, title, file_url, expiry_date }
```

### Rental Contract Events

```
QuoteCreated
    { quote_number, customer_id, depot_id, rental_start,
      rental_end, line_items: [...] }

QuoteSentToCustomer
    { sent_via, sent_to_email }

QuoteAccepted
    { accepted_by, accepted_at }

QuoteDeclined
    { reason }

QuoteExpired
    { }

QuoteConvertedToContract
    { contract_id }

ContractCreated
    { contract_number, customer_id, depot_id, site_id,
      rental_start, rental_end_expected, rate_table_id,
      deposit_amount, line_items: [...] }

ContractLineAdded
    { asset_id, period_type, rate_amount, minimum_charge }

ContractLineRemoved
    { asset_id, reason }

ContractSignedByCustomer
    { signed_by_name, signature_url, signature_ip, signed_at }

ContractTermsAccepted
    { terms_version, accepted_at }

ContractActivated
    { activated_by }

ContractExtended
    { previous_end, new_end, reason, approved_by }

ContractEarlyReturnRequested
    { requested_return_date, reason }

AssetDispatchedOnContract
    { asset_id, contract_line_id, dispatched_by,
      hour_meter_out, odometer_out, fuel_level_out }

AssetReturnedOnContract
    { asset_id, contract_line_id, returned_at,
      hour_meter_in, odometer_in, fuel_level_in }

ContractCompleted
    { completed_by, final_total }

ContractCancelled
    { reason, cancelled_by, cancellation_fee }

ContractDisputed
    { dispute_type, description, raised_by }

ContractDisputeResolved
    { resolution, resolved_by, adjustment_amount }
```

### Inspection and Damage Events

```
InspectionStarted
    { inspection_type, asset_id, contract_line_id,
      inspector_id, template_id }

InspectionItemRecorded
    { item_label, result, rating, notes }

InspectionPhotoAttached
    { photo_url, caption, inspection_item_id }

InspectionCompleted
    { overall_condition, hour_meter, odometer, notes }

InspectionReviewed
    { reviewed_by, approved }

DamageReported
    { asset_id, contract_line_id, description, severity,
      estimated_repair_cost }

DamagePhotoAttached
    { damage_record_id, photo_url, photo_type }

DamageChargeAssessed
    { charge_amount, assessed_by }

DamageChargeApproved
    { approved_by, approved_amount }

DamageChargeDisputedByCustomer
    { dispute_reason }

DamageRepairCompleted
    { work_order_id, actual_cost }

DamageInsuranceClaimFiled
    { claim_reference, insurer }

DamageInsuranceClaimSettled
    { settlement_amount }
```

### Maintenance Events

```
MaintenanceScheduleCreated
    { asset_id, category_id, schedule_type,
      interval_days, interval_hours, description }

MaintenanceDueAlertTriggered
    { asset_id, schedule_id, due_reason, current_value, threshold }

WorkOrderCreated
    { work_order_number, asset_id, depot_id, work_order_type,
      priority, title, description }

WorkOrderAssigned
    { assigned_to, scheduled_start, scheduled_end }

WorkOrderStarted
    { started_by, hour_meter_at }

WorkOrderPartAdded
    { part_number, part_name, quantity, unit_cost }

WorkOrderLaborRecorded
    { technician_id, hours, hourly_rate, description }

WorkOrderCompleted
    { completed_by, completion_notes, total_labor_cost,
      total_parts_cost, total_cost }

WorkOrderCancelled
    { reason, cancelled_by }
```

### Invoicing and Payment Events

```
InvoiceGenerated
    { invoice_number, contract_id, customer_id, invoice_type,
      period_start, period_end, line_items: [...],
      subtotal, tax_amount, total }

InvoiceSentToCustomer
    { sent_via, sent_to_email }

InvoiceViewedByCustomer
    { viewed_at, viewed_by }

PaymentReceived
    { amount, payment_method, reference, stripe_payment_id }

PaymentFailed
    { amount, payment_method, error_code, error_message }

PaymentRefunded
    { original_payment_id, refund_amount, reason }

InvoiceMarkedOverdue
    { days_overdue }

InvoiceWrittenOff
    { reason, approved_by }

DepositCollected
    { deposit_type, amount, stripe_payment_intent_id }

DepositAppliedToCharges
    { applied_amount, applied_to_invoice_id }

DepositRefunded
    { refund_amount, refund_method }
```

### Customer Events

```
CustomerRegistered
    { customer_number, customer_type, company_name,
      first_name, last_name, email, phone }

CustomerDetailsUpdated
    { fields_changed: { ... } }

CustomerContactAdded
    { contact_id, first_name, last_name, email, is_primary }

CustomerCreditLimitSet
    { old_limit, new_limit, approved_by }

CustomerAccountSuspended
    { reason, suspended_by }

CustomerAccountReactivated
    { reactivated_by }

CustomerInsuranceCertificateUploaded
    { insurer, policy_number, expiry_date, document_url }

CustomerInsuranceCertificateExpired
    { certificate_id }

CustomerSiteAdded
    { site_name, address, latitude, longitude }

CustomerPortalEnabled
    { }

CustomerPortalPasswordChanged
    { }
```

### Delivery and Transfer Events

```
DeliveryOrderCreated
    { order_type, contract_id, depot_id, scheduled_date,
      delivery_address }

DeliveryOrderAssignedToDriver
    { driver_id, vehicle_id }

DeliveryDeparted
    { departed_at }

DeliveryCompleted
    { arrived_at, customer_signature_url, signed_by_name }

DeliveryOrderCancelled
    { reason }

DepotTransferRequested
    { asset_id, from_depot_id, to_depot_id, reason }

DepotTransferApproved
    { approved_by }

DepotTransferDeparted
    { departed_at }

DepotTransferArrived
    { arrived_at }
```

### Telematics Events

```
TelematicsProviderConnected
    { provider_type, name, api_endpoint }

AssetTelematicsLinked
    { asset_id, provider_id, device_id }

TelematicsReadingReceived
    { asset_id, latitude, longitude, engine_hours,
      odometer_km, fuel_level_pct, engine_status,
      recorded_at }

TelematicsSyncCompleted
    { provider_id, records_processed, records_failed }

TelematicsSyncFailed
    { provider_id, error_message }

GeofenceViolationDetected
    { asset_id, contract_id, boundary_id, latitude, longitude }
```

---

## Read Model Projections (PostgreSQL)

These tables are rebuilt from the event stream. They can be dropped and recreated at any time without data loss.

### Projection 1: Asset Current State

```sql
-- Materialized from Asset* events
CREATE TABLE rm_assets (
    asset_id            UUID PRIMARY KEY,
    organization_id     UUID NOT NULL,
    asset_number        VARCHAR(50) NOT NULL,
    serial_number       VARCHAR(100),
    make                VARCHAR(100),
    model               VARCHAR(100),
    year                INT,
    category_id         UUID,
    category_name       VARCHAR(255),
    purchase_date       DATE,
    purchase_cost       DECIMAL(12, 2),
    current_value       DECIMAL(12, 2),
    home_depot_id       UUID,
    home_depot_name     VARCHAR(255),
    current_depot_id    UUID,
    current_depot_name  VARCHAR(255),
    status              VARCHAR(30) NOT NULL,
    condition_grade     VARCHAR(20),
    hour_meter          DECIMAL(12, 1),
    odometer            DECIMAL(12, 1),
    fuel_type           VARCHAR(30),
    last_latitude       DECIMAL(10, 7),
    last_longitude      DECIMAL(10, 7),
    last_location_at    TIMESTAMPTZ,
    insurance_expiry    DATE,
    next_service_due_at TIMESTAMPTZ,
    next_service_hours  DECIMAL(12, 1),
    last_inspection_at  TIMESTAMPTZ,
    photo_url           VARCHAR(500),
    is_active           BOOLEAN NOT NULL DEFAULT true,
    total_rental_revenue DECIMAL(14, 2) DEFAULT 0,
    total_maintenance_cost DECIMAL(14, 2) DEFAULT 0,
    total_rental_days   INT DEFAULT 0,
    last_rented_at      TIMESTAMPTZ,
    -- Current rental info (denormalized for dashboard)
    current_contract_id UUID,
    current_contract_number VARCHAR(50),
    current_customer_name VARCHAR(255),
    current_rental_end  TIMESTAMPTZ,
    last_event_version  INT NOT NULL,
    last_updated_at     TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_assets_org_status ON rm_assets(organization_id, status);
CREATE INDEX idx_rm_assets_depot ON rm_assets(current_depot_id);
CREATE INDEX idx_rm_assets_location ON rm_assets(last_latitude, last_longitude)
    WHERE last_latitude IS NOT NULL;
```

### Projection 2: Equipment Availability

```sql
-- Materialized from Contract and Asset events
-- This is the critical projection for preventing double-bookings
CREATE TABLE rm_availability (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id            UUID NOT NULL,
    organization_id     UUID NOT NULL,
    depot_id            UUID NOT NULL,
    period_start        TIMESTAMPTZ NOT NULL,
    period_end          TIMESTAMPTZ,   -- NULL = open-ended (maintenance, etc.)
    block_type          VARCHAR(30) NOT NULL,
        -- rental, reservation, maintenance, transfer, out_of_service
    reference_type      VARCHAR(50),   -- rental_contract, work_order, depot_transfer
    reference_id        UUID,
    customer_id         UUID,          -- for rental/reservation blocks
    customer_name       VARCHAR(255),
    contract_number     VARCHAR(50),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_avail_asset_period ON rm_availability(asset_id, period_start, period_end);
CREATE INDEX idx_rm_avail_org_depot ON rm_availability(organization_id, depot_id);

-- Exclusion constraint to prevent overlapping bookings per asset
-- Requires btree_gist extension
CREATE EXTENSION IF NOT EXISTS btree_gist;
ALTER TABLE rm_availability ADD CONSTRAINT no_overlap_per_asset
    EXCLUDE USING gist (
        asset_id WITH =,
        tstzrange(period_start, period_end, '[)') WITH &&
    );
```

### Projection 3: Contract Summary

```sql
CREATE TABLE rm_contracts (
    contract_id         UUID PRIMARY KEY,
    organization_id     UUID NOT NULL,
    contract_number     VARCHAR(50) NOT NULL,
    customer_id         UUID NOT NULL,
    customer_name       VARCHAR(255),
    depot_id            UUID NOT NULL,
    depot_name          VARCHAR(255),
    status              VARCHAR(30) NOT NULL,
    rental_start        TIMESTAMPTZ NOT NULL,
    rental_end_expected TIMESTAMPTZ,
    rental_end_actual   TIMESTAMPTZ,
    delivery_method     VARCHAR(20),
    site_address        TEXT,
    asset_count         INT NOT NULL DEFAULT 0,
    subtotal            DECIMAL(12, 2) NOT NULL DEFAULT 0,
    tax_amount          DECIMAL(12, 2) NOT NULL DEFAULT 0,
    damage_charges      DECIMAL(12, 2) NOT NULL DEFAULT 0,
    extra_charges       DECIMAL(12, 2) NOT NULL DEFAULT 0,
    total               DECIMAL(12, 2) NOT NULL DEFAULT 0,
    deposit_amount      DECIMAL(12, 2) NOT NULL DEFAULT 0,
    deposit_paid        BOOLEAN NOT NULL DEFAULT false,
    is_signed           BOOLEAN NOT NULL DEFAULT false,
    signed_at           TIMESTAMPTZ,
    is_overdue          BOOLEAN NOT NULL DEFAULT false,
    days_overdue        INT DEFAULT 0,
    created_by_name     VARCHAR(200),
    created_at          TIMESTAMPTZ NOT NULL,
    last_event_version  INT NOT NULL,
    last_updated_at     TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_contracts_org_status ON rm_contracts(organization_id, status);
CREATE INDEX idx_rm_contracts_customer ON rm_contracts(customer_id);
CREATE INDEX idx_rm_contracts_overdue ON rm_contracts(organization_id)
    WHERE is_overdue = true;

CREATE TABLE rm_contract_lines (
    contract_line_id    UUID PRIMARY KEY,
    contract_id         UUID NOT NULL REFERENCES rm_contracts(contract_id),
    asset_id            UUID NOT NULL,
    asset_number        VARCHAR(50),
    asset_description   VARCHAR(500),
    period_type         VARCHAR(20) NOT NULL,
    rate_amount         DECIMAL(10, 2) NOT NULL,
    status              VARCHAR(20) NOT NULL,
    dispatched_at       TIMESTAMPTZ,
    returned_at         TIMESTAMPTZ,
    hour_meter_out      DECIMAL(12, 1),
    hour_meter_in       DECIMAL(12, 1),
    line_subtotal       DECIMAL(12, 2) NOT NULL DEFAULT 0,
    last_updated_at     TIMESTAMPTZ NOT NULL
);
```

### Projection 4: Financial Summary

```sql
CREATE TABLE rm_invoices (
    invoice_id          UUID PRIMARY KEY,
    organization_id     UUID NOT NULL,
    invoice_number      VARCHAR(50) NOT NULL,
    contract_id         UUID NOT NULL,
    contract_number     VARCHAR(50),
    customer_id         UUID NOT NULL,
    customer_name       VARCHAR(255),
    invoice_type        VARCHAR(20) NOT NULL,
    status              VARCHAR(20) NOT NULL,
    invoice_date        DATE NOT NULL,
    due_date            DATE NOT NULL,
    total               DECIMAL(12, 2) NOT NULL,
    amount_paid         DECIMAL(12, 2) NOT NULL DEFAULT 0,
    balance_due         DECIMAL(12, 2) NOT NULL,
    is_overdue          BOOLEAN NOT NULL DEFAULT false,
    days_overdue        INT DEFAULT 0,
    last_payment_at     TIMESTAMPTZ,
    quickbooks_synced   BOOLEAN NOT NULL DEFAULT false,
    xero_synced         BOOLEAN NOT NULL DEFAULT false,
    last_updated_at     TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_invoices_org_status ON rm_invoices(organization_id, status);
CREATE INDEX idx_rm_invoices_overdue ON rm_invoices(organization_id)
    WHERE is_overdue = true;
CREATE INDEX idx_rm_invoices_customer ON rm_invoices(customer_id);
```

### Projection 5: Fleet Analytics

```sql
-- Pre-aggregated fleet utilization metrics, updated continuously
CREATE TABLE rm_fleet_utilization (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL,
    depot_id            UUID,          -- NULL = org-wide
    category_id         UUID,          -- NULL = all categories
    period_date         DATE NOT NULL,
    total_assets        INT NOT NULL DEFAULT 0,
    assets_on_rent      INT NOT NULL DEFAULT 0,
    assets_available    INT NOT NULL DEFAULT 0,
    assets_in_maintenance INT NOT NULL DEFAULT 0,
    assets_in_transit   INT NOT NULL DEFAULT 0,
    assets_out_of_service INT NOT NULL DEFAULT 0,
    utilization_pct     DECIMAL(5, 2),
    rental_revenue      DECIMAL(14, 2) DEFAULT 0,
    maintenance_cost    DECIMAL(14, 2) DEFAULT 0,
    last_updated_at     TIMESTAMPTZ NOT NULL,
    UNIQUE (organization_id, depot_id, category_id, period_date)
);

CREATE INDEX idx_rm_fleet_util_period ON rm_fleet_utilization(organization_id, period_date DESC);

-- Revenue per asset over time
CREATE TABLE rm_asset_revenue (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id            UUID NOT NULL,
    organization_id     UUID NOT NULL,
    period_month        DATE NOT NULL,  -- first day of month
    rental_days         INT NOT NULL DEFAULT 0,
    rental_revenue      DECIMAL(12, 2) NOT NULL DEFAULT 0,
    damage_charges      DECIMAL(12, 2) NOT NULL DEFAULT 0,
    maintenance_cost    DECIMAL(12, 2) NOT NULL DEFAULT 0,
    net_revenue         DECIMAL(12, 2) NOT NULL DEFAULT 0,
    utilization_pct     DECIMAL(5, 2),
    last_updated_at     TIMESTAMPTZ NOT NULL,
    UNIQUE (asset_id, period_month)
);
```

### Projection 6: Customer 360

```sql
CREATE TABLE rm_customers (
    customer_id         UUID PRIMARY KEY,
    organization_id     UUID NOT NULL,
    customer_number     VARCHAR(50) NOT NULL,
    customer_type       VARCHAR(20),
    display_name        VARCHAR(255) NOT NULL,
    email               VARCHAR(255),
    phone               VARCHAR(30),
    account_status      VARCHAR(20) NOT NULL,
    credit_limit        DECIMAL(12, 2),
    total_contracts     INT NOT NULL DEFAULT 0,
    active_contracts    INT NOT NULL DEFAULT 0,
    total_revenue       DECIMAL(14, 2) NOT NULL DEFAULT 0,
    outstanding_balance DECIMAL(14, 2) NOT NULL DEFAULT 0,
    overdue_balance     DECIMAL(14, 2) NOT NULL DEFAULT 0,
    average_rental_days DECIMAL(8, 1),
    last_rental_at      TIMESTAMPTZ,
    insurance_valid     BOOLEAN NOT NULL DEFAULT false,
    insurance_expiry    DATE,
    overdue_return_count INT NOT NULL DEFAULT 0,
    damage_claim_count  INT NOT NULL DEFAULT 0,
    risk_score          DECIMAL(5, 2),  -- AI-computed
    last_updated_at     TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_customers_org ON rm_customers(organization_id);
CREATE INDEX idx_rm_customers_risk ON rm_customers(organization_id, risk_score DESC)
    WHERE risk_score IS NOT NULL;
```

### Projection 7: Maintenance Dashboard

```sql
CREATE TABLE rm_work_orders (
    work_order_id       UUID PRIMARY KEY,
    organization_id     UUID NOT NULL,
    work_order_number   VARCHAR(50) NOT NULL,
    asset_id            UUID NOT NULL,
    asset_number        VARCHAR(50),
    asset_description   VARCHAR(255),
    depot_id            UUID NOT NULL,
    depot_name          VARCHAR(255),
    work_order_type     VARCHAR(30) NOT NULL,
    priority            VARCHAR(20) NOT NULL,
    status              VARCHAR(20) NOT NULL,
    title               VARCHAR(255) NOT NULL,
    assigned_to_name    VARCHAR(200),
    scheduled_start     TIMESTAMPTZ,
    scheduled_end       TIMESTAMPTZ,
    actual_start        TIMESTAMPTZ,
    actual_end          TIMESTAMPTZ,
    labor_cost          DECIMAL(10, 2) DEFAULT 0,
    parts_cost          DECIMAL(10, 2) DEFAULT 0,
    total_cost          DECIMAL(10, 2) DEFAULT 0,
    is_blocking_rental  BOOLEAN NOT NULL DEFAULT true,
    last_updated_at     TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_wo_org_status ON rm_work_orders(organization_id, status);
CREATE INDEX idx_rm_wo_asset ON rm_work_orders(asset_id);
CREATE INDEX idx_rm_wo_blocking ON rm_work_orders(organization_id)
    WHERE is_blocking_rental = true AND status NOT IN ('completed', 'cancelled');
```

---

## Command Handlers (Pseudocode)

### Reserve Asset Command

```typescript
// Example: handling a reservation command with optimistic concurrency
async function handleReserveAsset(command: ReserveAssetCommand): Promise<void> {
    const { assetId, contractId, startDate, endDate, customerId } = command;

    // 1. Load asset aggregate from event store
    const assetStream = await eventStore.loadStream('Asset', assetId);
    const asset = AssetAggregate.rehydrate(assetStream.events);

    // 2. Load availability projection to check conflicts
    const conflicts = await readDb.query(`
        SELECT * FROM rm_availability
        WHERE asset_id = $1
        AND tstzrange(period_start, period_end, '[)') &&
            tstzrange($2, $3, '[)')
    `, [assetId, startDate, endDate]);

    if (conflicts.length > 0) {
        throw new AssetNotAvailableError(assetId, startDate, endDate);
    }

    // 3. Validate business rules on the aggregate
    asset.validateCanBeReserved(startDate, endDate);

    // 4. Emit event with expected version (optimistic concurrency)
    await eventStore.append('Asset', assetId, assetStream.version, {
        type: 'AssetReservedForContract',
        data: {
            contract_id: contractId,
            customer_id: customerId,
            start_date: startDate,
            end_date: endDate,
        },
        metadata: {
            user_id: command.userId,
            correlation_id: command.correlationId,
            organization_id: command.organizationId,
        }
    });
    // If another reservation was made between steps 1-4,
    // the UNIQUE (stream_id, event_version) constraint will
    // cause a conflict error, preventing double-booking.
}
```

### Projection Updater (Event Handler)

```typescript
// Example: updating the availability projection when events arrive
class AvailabilityProjectionHandler {

    async handle(event: DomainEvent): Promise<void> {
        switch (event.type) {
            case 'AssetReservedForContract':
                await this.db.query(`
                    INSERT INTO rm_availability
                    (asset_id, organization_id, depot_id, period_start,
                     period_end, block_type, reference_type, reference_id,
                     customer_id, contract_number)
                    VALUES ($1, $2, $3, $4, $5, 'reservation',
                            'rental_contract', $6, $7, $8)
                `, [event.data.asset_id, ...]);
                break;

            case 'ContractActivated':
                // Upgrade reservation to rental
                await this.db.query(`
                    UPDATE rm_availability
                    SET block_type = 'rental'
                    WHERE reference_id = $1
                    AND block_type = 'reservation'
                `, [event.data.contract_id]);
                break;

            case 'AssetReturnedOnContract':
                // Remove availability block
                await this.db.query(`
                    UPDATE rm_availability
                    SET period_end = $1
                    WHERE asset_id = $2
                    AND reference_id = $3
                    AND block_type = 'rental'
                `, [event.data.returned_at, event.data.asset_id,
                    event.data.contract_id]);
                break;

            case 'AssetPlacedInMaintenance':
                await this.db.query(`
                    INSERT INTO rm_availability
                    (asset_id, organization_id, depot_id, period_start,
                     period_end, block_type, reference_type, reference_id)
                    VALUES ($1, $2, $3, $4, $5, 'maintenance',
                            'work_order', $6)
                `, [event.data.asset_id, ...]);
                break;

            // ... additional event handlers
        }

        // Update checkpoint
        await this.db.query(`
            INSERT INTO projection_checkpoints
            (projection_name, last_event_id, last_occurred_at, updated_at)
            VALUES ('availability', $1, $2, now())
            ON CONFLICT (projection_name)
            DO UPDATE SET last_event_id = $1,
                          last_occurred_at = $2,
                          updated_at = now()
        `, [event.event_id, event.occurred_at]);
    }
}
```

---

## Temporal Query Examples

One of the strongest benefits of event sourcing: answering temporal questions directly.

```typescript
// "What was the status of asset EX-042 at 3pm on 2026-03-15?"
async function getAssetStateAtTime(assetId: UUID, timestamp: Date): Promise<AssetState> {
    const events = await eventStore.loadStream('Asset', assetId, {
        before: timestamp
    });
    return AssetAggregate.rehydrate(events).toState();
}

// "Show me all changes to contract RC-2026-0142 in chronological order"
async function getContractHistory(contractId: UUID): Promise<DomainEvent[]> {
    return await eventStore.loadStream('RentalContract', contractId);
}

// "What was the fleet utilization on 2026-02-14?"
// Answer from rm_fleet_utilization projection or replay events
async function getHistoricalUtilization(orgId: UUID, date: Date) {
    return await readDb.query(`
        SELECT * FROM rm_fleet_utilization
        WHERE organization_id = $1
        AND period_date = $2
    `, [orgId, date]);
}
```

---

## Pros and Cons

### Pros

1. **Complete, immutable audit trail by design.** Every state change is a permanently stored event. Billing disputes are resolved by replaying the exact sequence of events on a contract. Insurance claims are backed by timestamped, tamper-evident records of inspections, damage reports, and charge assessments. This is not a bolt-on audit log -- it is the primary data store.

2. **Temporal queries are native.** "What was the status of this asset last Thursday?" is answered by replaying the event stream to that timestamp. In a normalized relational model, this requires reconstructing state from audit log entries -- error-prone and slow. Event sourcing makes time-travel queries first-class.

3. **Double-booking prevention at the data layer.** The UNIQUE constraint on (stream_id, event_version) provides optimistic concurrency control. Two simultaneous reservation attempts for the same asset will conflict at the database level, guaranteeing that at most one succeeds. No application-level distributed locks required.

4. **Read model optimization without schema compromise.** Each projection is purpose-built for its query pattern. The availability projection uses a GiST exclusion constraint; the fleet analytics projection pre-aggregates daily metrics; the customer 360 projection denormalizes across contracts, invoices, and damage claims. These are independently optimized without affecting write performance.

5. **Event replay enables new features retroactively.** When you add a demand forecasting AI model six months after launch, you can replay the full event history to train it on actual booking patterns, seasonal demand, and asset utilization -- data that would have been lost in a mutable relational model.

6. **Natural fit for the rental lifecycle.** Equipment rental is inherently state-machine-driven: available, reserved, dispatched, on-rent, returned, inspected, in-maintenance. Each transition maps directly to an event, making the domain model and the storage model congruent.

### Cons

1. **Significantly higher implementation complexity.** Event sourcing requires building event store infrastructure, projection update pipelines, snapshot management, idempotent event handlers, and eventual consistency handling. A team unfamiliar with the pattern will spend 2-3x longer reaching MVP compared to a standard CRUD approach.

2. **Eventual consistency between write and read sides.** After a command is processed and an event stored, there is a propagation delay before read model projections are updated. A user who creates a contract and immediately navigates to the contracts list may not see it. This requires careful UX design (optimistic UI updates, "command accepted" feedback patterns) and can confuse users and support staff.

3. **Projection rebuild cost at scale.** If a projection has a bug and must be rebuilt, replaying millions of events can take hours. For a fleet of 500 assets over 5 years, the event store could contain 50-100 million events. Full replay is a significant operational event.

4. **Event schema evolution is complex.** When the structure of an event changes (e.g., adding a new required field to ContractCreated), existing events in the store still have the old schema. Upcasters -- functions that transform old event formats to new ones during replay -- must be maintained indefinitely. This is a long-term maintenance burden that grows with the age of the system.

5. **Debugging is harder.** When a read model shows incorrect data, the developer must trace the issue through event replay logic rather than inspecting a single database row. Tooling for event store inspection and event replay debugging is less mature than standard SQL debugging tools.

6. **Over-engineered for the initial target market.** Independent equipment rental operators with 50-500 assets are unlikely to need the temporal query capabilities or the scaling benefits of CQRS. The complexity cost is paid upfront, but the benefits materialize at larger scale. This pattern is more appropriate for a platform serving hundreds of rental operators (multi-tenant SaaS at scale) than for a single mid-size operator.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Event Store | PostgreSQL (events table) or EventStoreDB |
| Read Model DB | PostgreSQL 16+ |
| Event Bus | Apache Kafka, NATS JetStream, or PostgreSQL LISTEN/NOTIFY (small scale) |
| Application Framework | NestJS (TypeScript) with @nestjs/cqrs, or Axon Framework (Java) |
| Serialization | JSON (JSONB in PostgreSQL) with JSON Schema validation |
| Snapshot Storage | Same PostgreSQL instance (snapshots table) |
| Projection Runner | Custom event processor or Eventuous (.NET) / Sequent (Ruby) |
| Monitoring | Event processing lag dashboards; projection checkpoint monitoring |

---

## Migration and Scaling Considerations

### Phase 1: Start Simple
- Use PostgreSQL as both event store and read model database.
- Use PostgreSQL LISTEN/NOTIFY or a simple polling loop for projection updates (latency < 1 second at low volume).
- Take snapshots every 100 events per aggregate to keep replay times under 50ms.

### Phase 2: Separate Concerns (100+ tenants or 1000+ assets)
- Introduce Kafka or NATS as the event bus for reliable, ordered event delivery to multiple projection consumers.
- Move read model projections to separate PostgreSQL instances (or read replicas) to isolate read and write workloads.
- Implement parallel projection rebuilds using partitioned event ranges.

### Phase 3: Scale the Event Store (10M+ events)
- Partition the events table by month (already included in schema).
- Archive events older than 2 years to cold storage (S3 + Parquet) while keeping them replayable.
- Consider EventStoreDB as a dedicated event store if PostgreSQL event table performance degrades.
- Implement event stream compaction: merge old fine-grained events into coarser summary events for long-lived aggregates.

### Migration from CRUD
If starting with the normalized relational model (Suggestion 1) and later migrating to event sourcing:
1. Introduce event capture alongside CRUD writes (dual-write with outbox pattern).
2. Build new projections from the event stream while keeping existing queries against the relational tables.
3. Gradually migrate read paths to projections.
4. Once all reads use projections, make the event store the source of truth and deprecate direct table writes.

---

## Summary

The event-sourced CQRS model is the most architecturally ambitious option. It provides unmatched auditability, temporal query capabilities, and scaling flexibility -- all highly relevant to equipment rental management where billing disputes, damage claims, and regulatory compliance demand provable history. However, it carries significant implementation complexity and is likely over-engineered for an MVP targeting independent rental operators. This model is best suited for teams with event sourcing experience building a multi-tenant SaaS platform that will serve many rental operators simultaneously, where the audit trail and temporal query capabilities become strong competitive differentiators.
