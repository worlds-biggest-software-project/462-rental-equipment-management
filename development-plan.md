# Rental Equipment Management — Phased Development Plan

> Project: 462-rental-equipment-management · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the
hybrid relational + JSONB data model (`data-model-suggestion-3.md`) into a concrete,
phased build. The product is an open-source, AI-native platform for equipment rental
operators with 50–500 assets: asset registry, availability and bookings, inspections,
maintenance, billing, and accounting/telematics integrations, with AI capabilities
(demand forecasting, damage assessment, overdue prediction) and an MCP server layered on
top of a well-documented public API.

The hybrid data model (Suggestion 3) is adopted as the canonical schema: core financial
and relational entities stay in typed PostgreSQL columns; equipment specs, rate rules,
inspection checklists, and integration payloads live in JSONB validated at the
application layer.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | TypeScript (Node 22 LTS) | The domain is API/integration/frontend-heavy (booking portal, dashboard, OAuth integrations, webhooks), not ML-training-heavy. One language across API, web UI, and MCP server reduces context-switching. AI features call hosted LLM APIs rather than training models, so Python's ML edge is not needed here. |
| Runtime | Node.js 22 LTS | Stable LTS through 2027; native `fetch`, test runner, and `--watch`. Broadest library support for the OAuth/accounting/telematics SDKs (Geotab, Xero, Stripe all ship official Node SDKs). |
| API framework | Fastify 5 + `@fastify/swagger` | Fastify is fast, schema-first (JSON Schema native), and auto-generates an **OpenAPI 3.1** document from route schemas — directly satisfying the contract-first requirement from standards.md. Its JSON Schema validation doubles as the application-layer JSONB validation the data model requires. |
| Database | PostgreSQL 16 | Mandated by the chosen data model. ACID for financial data, JSONB + GIN for equipment-attribute variability, native partitioning for telematics time-series, PostGIS for depot/site geo. |
| DB extensions | PostGIS, pgcrypto, pg_cron | PostGIS for delivery-radius and nearest-depot queries; pgcrypto for `gen_random_uuid()`; pg_cron for telematics partition rotation. |
| ORM / query layer | Drizzle ORM | First-class TypeScript types, excellent JSONB column support, raw-SQL escape hatch for JSONB path queries and partitioned tables, and migrations that read like SQL (important when the schema is the spec). |
| Migrations | Drizzle Kit | Generates and applies SQL migrations from the Drizzle schema; deterministic and reviewable. |
| JSONB validation | Ajv (JSON Schema 2020-12) | Validates JSONB payloads (`specifications`, `pricing_rules`, `checklist`) against per-category/per-org JSON Schemas before write, as the data model mandates. Same dialect Fastify uses. |
| Task queue | BullMQ on Redis 7 | Async workloads exist: telematics sync, accounting sync, notification dispatch, AI inference, webhook processing. BullMQ gives retries, scheduling (for reminders), rate-limiting (for metered QuickBooks reads), and a dashboard. |
| Cache / queue store | Redis 7 | Backs BullMQ and caches availability/rate computations. |
| Object storage | S3-compatible (AWS S3 or MinIO self-hosted) | Inspection photos, signed-contract PDFs, damage evidence. MinIO keeps the self-hosted deployment mode fully open-source. |
| Frontend | Next.js 15 (App Router) + React 19 | Two surfaces: an internal operations dashboard and a customer self-service booking portal. Next.js server components suit the data-dense dashboard; one framework serves both. shadcn/ui + Tailwind for the component layer. |
| Mobile inspection | PWA (offline-first, Workbox + IndexedDB) | features.md requires offline-capable field inspection. A PWA avoids native app-store overhead for MVP while supporting offline photo capture and deferred sync. Native wrapper deferred to backlog. |
| Auth (staff) | Lucia-style session auth + RBAC | Self-hosted-friendly, no external IdP dependency; org-scoped roles (admin, operator, technician, viewer). |
| Auth (integrations) | OAuth 2.0 Authorization Code + PKCE (RFC 7636), per RFC 9700 | Every integration target (Geotab, Samsara, QuickBooks, Xero, Docusign, John Deere) requires OAuth 2.0. A shared OAuth client library with per-provider config is built once. |
| Payments | Stripe (Payment Intents, `capture_method: manual`) | Pre-auth holds for damage deposits is a core rental pattern; Payment Intents is the recommended Stripe integration. |
| E-signature | Internal SES + optional Dropbox Sign / Docusign adapter | An internal Simple Electronic Signature (ESIGN/eIDAS-compliant for most rental agreements) ships in MVP; pluggable adapter allows Docusign/Dropbox Sign later. |
| Telematics | AEMP 2.0 / ISO 15143-3 normalised ingestion | Single normalised pipeline; per-provider adapters (Geotab JSON-RPC, Samsara REST) map into the AEMP schema, as standards.md recommends. |
| AI / LLM | Vercel AI SDK + provider-agnostic gateway | Damage assessment (vision), demand forecasting prompts, overdue-risk explanation. Provider-agnostic so self-hosters can point at any OpenAI-compatible or Anthropic endpoint. |
| AI forecasting | Lightweight statistical model (Holt-Winters / Prophet-in-JS) + LLM narration | Demand forecasting is time-series; a statistical core is cheaper and more reliable than an LLM, with the LLM used to narrate recommendations. |
| MCP server | `@modelcontextprotocol/sdk` (TypeScript) | Exposes availability, maintenance, and damage tools per standards.md MCP note — a stated differentiator. |
| Notifications | Twilio (SMS/WhatsApp) + SMTP (email) | features.md requires confirmation/reminder/overdue notifications; Twilio per standards.md. |
| Testing | Vitest (unit/integration) + Playwright (E2E) + Testcontainers (real Postgres/Redis) | Vitest is fast and native to the TS stack; Testcontainers spins ephemeral Postgres/Redis for integration tests; Playwright drives the booking-portal and dashboard E2E flows. |
| Code quality | Biome (lint + format) + `tsc --noEmit` | Single fast tool for lint+format; strict TypeScript for type checking. |
| Package manager | pnpm (workspaces) | Monorepo with shared packages (`@rem/db`, `@rem/core`, `@rem/integrations`); pnpm workspaces are the standard efficient choice. |
| Containerisation | Docker + docker-compose | Self-hosted deployment mode is a core differentiator; compose brings up Postgres, Redis, MinIO, API, worker, and web in one command. |
| API security | OWASP API Security Top 10 (2023) controls | Org-scoped object authorization (BOLA defence), rate limiting, audit log, no excessive data exposure via serializers. |

### Project Structure

```
rental-equipment-management/
├── package.json                  # pnpm workspace root
├── pnpm-workspace.yaml
├── biome.json
├── tsconfig.base.json
├── docker-compose.yml            # postgres, redis, minio, api, worker, web
├── Dockerfile.api
├── Dockerfile.worker
├── Dockerfile.web
├── .env.example
├── README.md
├── docs/
│   └── openapi.json              # generated, committed for SDK/client use
├── packages/
│   ├── db/                       # @rem/db — Drizzle schema, migrations, client
│   │   ├── src/
│   │   │   ├── schema/           # one file per domain group (assets.ts, contracts.ts…)
│   │   │   ├── client.ts
│   │   │   ├── seed.ts
│   │   │   └── jsonb-schemas/    # Ajv JSON Schemas for each JSONB column
│   │   ├── drizzle/              # generated migrations (SQL)
│   │   └── drizzle.config.ts
│   ├── core/                     # @rem/core — domain logic, framework-agnostic
│   │   ├── src/
│   │   │   ├── assets/
│   │   │   ├── availability/     # conflict detection, calendar
│   │   │   ├── contracts/        # state machine, lifecycle
│   │   │   ├── billing/          # rate engine, invoice builder
│   │   │   ├── inspections/
│   │   │   ├── maintenance/      # scheduler, lockout
│   │   │   ├── pricing/          # rate resolution
│   │   │   ├── notifications/
│   │   │   └── errors.ts
│   │   └── test/
│   ├── integrations/             # @rem/integrations — external systems
│   │   ├── src/
│   │   │   ├── oauth/            # shared OAuth2 + PKCE client
│   │   │   ├── telematics/       # aemp/, geotab/, samsara/
│   │   │   ├── accounting/       # quickbooks/, xero/
│   │   │   ├── payments/         # stripe/
│   │   │   ├── esign/            # internal/, docusign/, dropboxsign/
│   │   │   └── messaging/        # twilio/, smtp/
│   │   └── test/
│   ├── ai/                       # @rem/ai — forecasting, damage, overdue
│   │   ├── src/
│   │   │   ├── forecasting/
│   │   │   ├── damage/
│   │   │   ├── overdue/
│   │   │   └── llm-client.ts
│   │   └── test/
│   └── contracts-types/          # @rem/types — shared zod/JSON Schema DTOs
├── apps/
│   ├── api/                      # Fastify HTTP API + OpenAPI
│   │   ├── src/
│   │   │   ├── server.ts
│   │   │   ├── plugins/          # auth, rbac, error-handler, swagger
│   │   │   ├── routes/           # one folder per resource
│   │   │   └── webhooks/         # stripe, twilio, accounting
│   │   └── test/
│   ├── worker/                   # BullMQ workers
│   │   └── src/
│   │       ├── queues.ts
│   │       └── processors/      # telematics-sync, accounting-sync, notify, ai
│   ├── web/                      # Next.js dashboard + booking portal
│   │   └── src/app/
│   │       ├── (dashboard)/
│   │       ├── (portal)/
│   │       └── (inspect)/        # offline PWA inspection
│   └── mcp/                      # MCP server over the rental API
│       └── src/server.ts
└── tests/
    └── e2e/                      # Playwright cross-app flows
```

The structure is additive per phase: early phases populate `packages/db` and `packages/core`;
integration phases fill `packages/integrations`; later phases add `packages/ai` and `apps/mcp`.

---

## Phase 1: Foundation — Monorepo, Database, and Multi-Tenant Auth

### Purpose
Stand up the repository, the PostgreSQL schema, the database client, and org-scoped
authentication/RBAC. After this phase the system can be deployed via docker-compose,
the full hybrid schema exists, JSONB validation is wired up, and a staff user can log in
and be scoped to an organization. Everything later builds on these primitives.

### Tasks

#### 1.1 — Monorepo, tooling, and docker-compose
**What**: Initialise the pnpm workspace, Biome, strict TypeScript, and a docker-compose stack.

**Design**:
- `pnpm-workspace.yaml` includes `packages/*` and `apps/*`.
- `tsconfig.base.json`: `"strict": true`, `"noUncheckedIndexedAccess": true`, `"module": "NodeNext"`.
- `biome.json`: enable recommended lint rules + formatter; line width 100.
- `docker-compose.yml` services: `postgres` (postgres:16 with PostGIS image `postgis/postgis:16-3.4`), `redis` (redis:7-alpine), `minio` (minio/minio), plus placeholders for `api`, `worker`, `web` built from their Dockerfiles.
- `.env.example` enumerates: `DATABASE_URL`, `REDIS_URL`, `S3_ENDPOINT`, `S3_ACCESS_KEY`, `S3_SECRET_KEY`, `S3_BUCKET`, `JWT_SECRET`, `SESSION_SECRET`, `APP_BASE_URL`.
- Root scripts: `pnpm dev`, `pnpm test`, `pnpm lint`, `pnpm typecheck`, `pnpm db:migrate`, `pnpm db:seed`.

**Testing**:
- `Smoke: docker compose up` → postgres, redis, minio reach healthy state (compose healthchecks).
- `Unit: pnpm typecheck` on empty packages → exit 0.
- `Unit: pnpm lint` → exit 0 on scaffold.

#### 1.2 — Database schema (`@rem/db`) from the hybrid model
**What**: Implement the entire Suggestion 3 schema as Drizzle schema files plus the initial migration.

**Design**:
- Translate each SQL block from `data-model-suggestion-3.md` into Drizzle table definitions, grouped by file:
  - `organizations.ts`: `organizations`, `depots`
  - `users.ts`: `users` (referenced throughout — see structure below)
  - `equipment.ts`: `equipment_categories`, `assets`, `asset_photos`
  - `customers.ts`: `customers`
  - `rates.ts`: `rate_tables`, `rate_entries`, `customer_rate_overrides`
  - `contracts.ts`: `rental_contracts`, `contract_lines`, `contract_extras`, `contract_history`
  - `inspections.ts`: `inspections`, `damage_records`
  - `maintenance.ts`: `maintenance_schedules`, `work_orders`
  - `telematics.ts`: `telematics_providers`, `asset_telematics_links`, `telematics_readings` (partitioned)
  - `billing.ts`: `invoices`, `invoice_lines`, `payments`
  - `system.ts`: `notifications`, `audit_log`
- The data model references a `users` table not fully specified; define it:

```typescript
// users table
export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  organizationId: uuid('organization_id').notNull().references(() => organizations.id),
  email: varchar('email', { length: 255 }).notNull(),
  passwordHash: varchar('password_hash', { length: 255 }).notNull(),
  fullName: varchar('full_name', { length: 200 }).notNull(),
  role: varchar('role', { length: 20 }).notNull().default('operator'), // admin|operator|technician|viewer
  isActive: boolean('is_active').notNull().default(true),
  lastLoginAt: timestamp('last_login_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (t) => ({ uniqEmail: unique().on(t.organizationId, t.email) }));
```

- All typed columns, constraints, UNIQUE keys, and indexes from the model are reproduced exactly.
- `telematics_readings` is created as a partitioned table via a raw-SQL migration (Drizzle does not model partitions); ship a `create_partition(date)` SQL function and a quarterly pg_cron job.
- Enable extensions in migration `0000_extensions.sql`: `CREATE EXTENSION IF NOT EXISTS postgis, pgcrypto, pg_cron;`.
- `client.ts` exports a singleton Drizzle client from `DATABASE_URL` with connection pooling.

**Testing**:
- `Integration (Testcontainers Postgres): run all migrations → 0 errors; \dt shows all 24 tables.`
- `Integration: insert organization + depot + asset with valid FKs → success.`
- `Integration: insert asset referencing non-existent category → FK violation.`
- `Integration: insert two assets same (organization_id, asset_number) → unique violation.`
- `Integration: insert telematics_reading dated 2026-05-15 → lands in *_2026_q2 partition.`
- `Unit: chk_rate_target — rate_entry with both category_id and asset_id null → CHECK violation.`

#### 1.3 — JSONB schema validation (`@rem/db/jsonb-schemas`)
**What**: Ajv-based validators for each JSONB column, enforced before write.

**Design**:
- One JSON Schema (2020-12) per JSONB column: `assetSpecifications` (validated dynamically against the category's `attribute_schema`), `ratePricingRules`, `inspectionChecklist`, `organizationSettings`, `depotConfig`, etc.
- `validateAssetSpecifications(spec, attributeSchema)`: builds an Ajv schema on the fly from the category's `attribute_schema` array (mapping `type: number|enum|boolean` → JSON Schema), enforcing `required` keys and `options` enums.
- Every JSONB document carries a `_schema_version` integer (per the model's versioning note); validators reject unknown versions and an `upcast(doc)` migrates old versions.
- Signature: `function validate<T>(schemaKey: JsonbSchemaKey, value: unknown): Result<T, ValidationError[]>`.

**Testing**:
- `Unit: excavator spec {operating_weight_kg: 22000, bucket_capacity_m3: 1.19, tail_swing: "conventional"} against excavator attribute_schema → valid.`
- `Unit: spec missing required "power_kva" for generator → error naming "power_kva".`
- `Unit: spec {operating_weight_kg: "twenty tons"} (string) → type error.`
- `Unit: tail_swing: "diagonal" (not in options) → enum error.`
- `Unit: pricing_rules with malformed volume_discounts → error.`
- `Unit: document with _schema_version: 1 → upcast to v2 produces current shape.`

#### 1.4 — Session auth, RBAC, and org scoping (`apps/api` plugins)
**What**: Email/password session auth, role-based access control, and a request-level org scope guard.

**Design**:
- `auth` plugin: `POST /auth/login` (argon2id verify, issue httpOnly session cookie backed by Redis), `POST /auth/logout`, `GET /auth/me`.
- `rbac` plugin: `request.user = { id, organizationId, role }`. `requireRole(...roles)` route guard.
- **BOLA defence (OWASP API #1)**: a `withOrgScope` helper that injects `organization_id = request.user.organizationId` into every query; route handlers cannot fetch cross-org rows. All list/get queries go through repository functions that take `organizationId` as a non-optional first argument.
- Roles: `admin` (all), `operator` (CRUD contracts/assets/customers/invoices), `technician` (inspections/work orders), `viewer` (read-only).

**Testing**:
- `Integration (mocked): valid credentials → 200 + session cookie; /auth/me returns user.`
- `Integration: wrong password → 401, no cookie.`
- `Integration: viewer calls POST /contracts → 403.`
- `Integration (BOLA): user in org A requests GET /assets/{id} for an asset in org B → 404 (not 403, to avoid existence disclosure).`
- `Unit: argon2id hash round-trips; tampered hash fails.`

---

## Phase 2: Asset Registry & Equipment Catalogue

### Purpose
Deliver the foundational inventory: equipment categories with dynamic attribute schemas,
assets with validated JSONB specifications, photos, and asset status lifecycle. This is
the substrate that bookings, inspections, and maintenance all reference. After this phase,
an operator can model their entire fleet.

### Tasks

#### 2.1 — Equipment categories with attribute schemas
**What**: CRUD for `equipment_categories`, including the `attribute_schema` and `default_inspection_items` JSONB.

**Design**:
- Endpoints: `POST/GET/PATCH/DELETE /categories`, `GET /categories/{id}`. Tree support via `parent_id`; `GET /categories?tree=true` returns nested.
- On write, validate `attribute_schema` array entries conform to the attribute-descriptor meta-schema (`key`, `label`, `type ∈ {number,string,enum,boolean,date}`, optional `unit`, `options`, `required`, `default`).
- Deleting a category with assets → 409 with count.

**Testing**:
- `Unit: valid excavator attribute_schema → persisted.`
- `Unit: attribute descriptor with type "color" → 400 invalid type.`
- `Integration: delete category with 3 assets → 409.`
- `Integration: GET /categories?tree=true → nested parent/child structure.`

#### 2.2 — Asset CRUD with dynamic specification validation
**What**: Full asset lifecycle CRUD with `specifications` validated against the asset's category schema.

**Design**:
- Endpoints: `POST/GET/PATCH /assets`, `GET /assets/{id}`, `GET /assets` (filter by category, depot, status, free-text on make/model/serial, and JSONB spec predicates e.g. `?spec.operating_weight_kg[gte]=20000`).
- Spec filter translation: query params like `spec.power_kva[gte]=200` compile to `(specifications->>'power_kva')::numeric >= 200`, leveraging functional indexes where defined.
- Asset status enum and allowed transitions:

```
available → reserved → on_rent → returned → available
available → out_of_service → available          (maintenance)
any        → retired                             (terminal)
```

- Status transitions go through `core/assets/transition.ts`; illegal transitions throw `InvalidStatusTransition`.
- `current_depot_id` vs `home_depot_id` tracked separately (supports later inter-depot transfers).

**Testing**:
- `Integration: create CAT 320 excavator with valid specs → 201, status "available".`
- `Integration: create with specifications failing category schema → 400 listing offending fields.`
- `Integration: GET /assets?spec.operating_weight_kg[gte]=20000 → only assets ≥20t.`
- `Unit: transition available→on_rent → allowed; retired→available → InvalidStatusTransition.`
- `Integration (BOLA): fetch asset from another org → 404.`

#### 2.3 — Asset photos and document storage
**What**: Upload/list/delete asset photos to S3/MinIO with primary-photo selection.

**Design**:
- `POST /assets/{id}/photos` accepts multipart; stores object under `org/{orgId}/assets/{assetId}/{uuid}.jpg`; writes `asset_photos` row.
- Presigned GET URLs (15-min TTL) returned in responses — images are never proxied through the API.
- Setting `is_primary=true` on one photo clears it on the others (transaction).

**Testing**:
- `Integration (MinIO container): upload JPEG → object in bucket, row created, presigned URL resolves.`
- `Integration: set photo B primary → photo A is_primary becomes false.`
- `Unit: reject upload >10MB or non-image MIME → 400.`

---

## Phase 3: Availability, Bookings & Rental Contracts (Core Value)

### Purpose
The heart of the product: real-time availability with cross-location conflict detection,
quote-to-contract workflow, and the contract lifecycle (draft → reserved → active →
returned → closed) including extensions and early returns. After this phase the platform
can actually run a rental from quote to return.

### Tasks

#### 3.1 — Availability engine and conflict detection
**What**: Compute whether an asset is bookable for a date range, detecting conflicts across all contracts.

**Design**:
- `core/availability/check.ts`:
  - `isAvailable(assetId, start, end): { available: boolean; conflicts: ConflictRef[] }`.
  - An asset is unavailable if any `contract_lines` (via `rental_contracts` of status in `reserved|active`) overlap `[start,end)`, OR any `work_orders` with status `open|in_progress` overlap, OR `assets.status = out_of_service|retired`.
  - Overlap test: `tstzrange(existing_start, existing_end) && tstzrange(start, end)` using a Postgres `&&` range overlap query (add an exclusion GiST index later for hard guarantees).
- `getAvailabilityCalendar(categoryId|assetId, from, to)` → array of `{ date, available_count, total_count }` for the calendar UI.
- Endpoint: `GET /availability?categoryId=&start=&end=&depotId=`.

**Testing**:
- `Unit: asset free in range → available true, conflicts [].`
- `Unit: asset with overlapping active contract → available false, conflict references the contract.`
- `Unit: adjacent ranges (end == next start) → no conflict (half-open intervals).`
- `Unit: asset out_of_service → unavailable regardless of contracts.`
- `Integration: calendar over a month with 2 bookings → correct daily available_count.`

#### 3.2 — Quote-to-contract workflow and lifecycle state machine
**What**: Create quotes, convert to contracts, and drive the contract lifecycle with a guarded state machine.

**Design**:
- Contract status machine (typed columns from the model):

```
draft → quoted → reserved → active → returned → closed
draft → cancelled
quoted → cancelled
reserved → cancelled
active → active (extension; updates rental_end_expected, writes contract_history)
```

- `POST /contracts` (draft) with `customer_id`, `depot_id`, line items (`asset_id`, `period_type`, dates). On create:
  1. Validate every asset is available for the requested range (3.1).
  2. Snapshot customer into `customer_snapshot` JSONB (frozen per the model).
  3. Compute pricing (Phase 4 plugs in here; until then use base `rate_entries`).
- `POST /contracts/{id}/reserve` → sets assets to `reserved`, status `reserved`.
- `POST /contracts/{id}/dispatch` → records dispatch readings on lines, asset → `on_rent`, status `active`.
- `POST /contracts/{id}/return` → records return readings, asset → `returned` then `available`, status `returned`.
- `POST /contracts/{id}/extend` body `{ new_end, reason }` → validates no new conflict in extended window, writes `contract_history` entry.
- `POST /contracts/{id}/cancel`, `POST /contracts/{id}/close`.
- Every transition writes a `contract_history` row and an `audit_log` row.

```typescript
interface CreateContractInput {
  customerId: string;
  depotId: string;
  rateTableId?: string;
  rentalStart: string;   // ISO
  rentalEndExpected: string;
  lines: Array<{ assetId: string; periodType: 'hourly'|'daily'|'weekly'|'monthly'; estimatedPeriods?: number }>;
  deliveryDetails?: DeliveryDetails;
  depositAmount?: number;
}
```

**Testing**:
- `Integration: create draft with available assets → 201, customer_snapshot populated.`
- `Integration: create with a double-booked asset → 409 conflict.`
- `Unit: reserve→active→returned→closed → all allowed; closed→active → InvalidStatusTransition.`
- `Integration: extend into a window where the asset is booked by another contract → 409.`
- `Integration: dispatch records hour_meter into contract_lines.dispatch_readings JSONB.`
- `Integration: every transition appends a contract_history row.`

#### 3.3 — Digital signature capture (internal SES, ESIGN/eIDAS)
**What**: Capture a legally-valid Simple Electronic Signature on a contract.

**Design**:
- `POST /contracts/{id}/sign` body `{ signed_by_name, signature_image (base64 PNG), consent: true }`.
- Records intent + consent (ESIGN Act requirements): stores `signature_details` JSONB `{ signed_by_name, signed_at, signature_url, ip_address, user_agent, terms_version, esign_provider: "internal" }`; signature image uploaded to S3.
- Captures `terms` snapshot JSONB at signing time. Signing is only permitted from `quoted`/`reserved` status; sets a `signed` flag.
- Adapter interface `EsignProvider` so Docusign/Dropbox Sign can be slotted in (backlog).

**Testing**:
- `Unit: sign without consent: true → 400.`
- `Integration: sign → signature_details populated with ip_address and user_agent; image in S3.`
- `Integration: attempt to sign a draft (not quoted/reserved) → 409.`

---

## Phase 4: Billing Engine, Invoicing & Payments

### Purpose
Turn rentals into money: the rate-resolution and billing engine (handling rate tables,
customer overrides, weekend/holiday calendars, minimum charges, volume/long-term
discounts, fuel surcharges), invoice generation, and Stripe payments including damage-
deposit pre-auth holds. After this phase the platform can quote accurately and collect.

### Tasks

#### 4.1 — Rate resolution and billing engine
**What**: Given a contract line and a rate context, compute the charge applying all rules.

**Design**:
- `core/pricing/resolveRate(assetId, customerId, date)`:
  1. Find applicable `rate_table` (effective-dated, default fallback).
  2. Find `rate_entry` (asset-specific overrides category).
  3. Apply `customer_rate_overrides.overrides` (global + per-category discounts).
- `core/billing/computeLineCharge(line, rateTable, context)`:
  - Counts billable periods per `pricing_rules.billing_calendar` (`calendar_days` | `business_days`) honouring `weekend_handling` (`free|charged|half`) and `holiday_calendar`.
  - Applies `minimum_rental_period`, `minimum_charge`, `long_term_discounts`, `volume_discounts`, `shift_pricing`, `fuel_surcharge`, `late_return_penalty` (grace period → additional day).
  - Returns `{ basePeriods, billablePeriods, subtotal, discountsApplied, surcharges, lineTotal }` with an itemised breakdown for transparency.
- Holiday calendars: ship `US_federal` and `UK_bank` as data tables; selectable via `pricing_rules.holiday_calendar`.
- All money handled as integer minor units internally; DECIMAL at the DB boundary. **No JSONB for computed financials.**

**Testing**:
- `Unit: 5 calendar days, daily rate $200, no discounts → $1000.`
- `Unit: 5 days spanning a weekend, weekend_handling "free" → 3 billable days → $600.`
- `Unit: 30 days, long_term_discount 15% at ≥30 → $4250 from $5000.`
- `Unit: rental shorter than minimum_rental_period → minimum charge applied.`
- `Unit: late return 3h within 4h grace → no penalty; 6h → +1 day.`
- `Unit: 5 items with volume_discount 10% at ≥5 → discount on all lines.`
- `Unit: customer override global_discount_pct 10 stacks correctly with table rules (documented order).`

#### 4.2 — Invoice generation
**What**: Generate invoices from contracts (full, partial/progress, final-with-damages).

**Design**:
- `POST /contracts/{id}/invoices` body `{ type: 'rental'|'deposit'|'damage'|'final', period_start?, period_end? }`.
- Builds `invoices` + `invoice_lines` from contract lines, extras, and approved damage charges; computes `subtotal`, `tax_amount` (org `tax_rate` or per-line), `total`, `balance_due`.
- Tax-exempt customers (`customers.tax_exempt`) → zero tax with a noted exemption.
- Invoice numbering: per-org monotonic `INV-{YYYY}-{seq}` via a sequence table to avoid gaps under concurrency.
- `POST /invoices/{id}/send` → status `sent`, enqueue notification (Phase 7).
- Status machine: `draft → sent → partially_paid → paid`; plus `void`.

**Testing**:
- `Integration: invoice a returned contract → lines match contract lines + extras; total correct.`
- `Unit: tax_exempt customer → tax_amount 0.`
- `Integration: two concurrent invoice creations → no duplicate invoice_number (sequence).`
- `Integration: final invoice includes approved damage_records charges.`

#### 4.3 — Stripe payments with deposit pre-authorisation
**What**: Collect payments and place/capture/release damage-deposit holds via Stripe Payment Intents.

**Design**:
- `POST /invoices/{id}/pay` → creates a Payment Intent (`capture_method: automatic`), returns client secret for the portal.
- Deposit holds: `POST /contracts/{id}/deposit/authorize` → Payment Intent with `capture_method: manual` (pre-auth hold); `.../deposit/capture` (on damage) and `.../deposit/release` (clean return).
- `POST /webhooks/stripe`: verifies signature; on `payment_intent.succeeded` writes a `payments` row with `gateway_details` JSONB and updates invoice `amount_paid`/`balance_due`/`status`; idempotent on `payment_intent.id`.
- Refunds: `POST /payments/{id}/refund`.

**Testing**:
- `Integration (Stripe mock): pay invoice → payment row, balance_due decremented, status paid.`
- `Integration: deposit authorize → manual-capture PI created; release → PI cancelled.`
- `Integration (webhook): valid signature payment_intent.succeeded → payment recorded once; replayed event → no duplicate (idempotency).`
- `Integration (webhook): invalid signature → 400, no DB write.`

---

## Phase 5: Inspections, Maintenance & Operations Dashboard

### Purpose
Close the operational loop: pre/post-rental inspections with photos and damage capture,
calendar/meter-based maintenance scheduling with out-of-service lockout, and the
operations dashboard plus offline-capable inspection PWA. After this phase, field and
shop operations are fully supported.

### Tasks

#### 5.1 — Inspection workflow with checklists and damage capture
**What**: Conduct pre/post inspections against category-derived checklists, attach photos, and raise damage records.

**Design**:
- `POST /inspections` body `{ asset_id, contract_line_id?, inspection_type: 'pre'|'post'|'periodic' }` → seeds `checklist` JSONB from category `default_inspection_items`.
- `PATCH /inspections/{id}` records results per item (`pass_fail`, `rating_1_5`, `photo_required`), `readings` (hour meter, fuel), and photos to S3.
- `POST /inspections/{id}/complete` → validates all `required` items answered; sets `overall_condition`.
- On a `post` inspection, any item flagged as damage creates a `damage_records` row (normalized, financial significance) linked to inspection + contract line; `estimated_repair_cost` and `charge_to_customer` feed final invoicing (4.2).

**Testing**:
- `Integration: create pre-inspection for an excavator → checklist seeded from category defaults.`
- `Unit: complete with a required photo_required item missing → 400.`
- `Integration: post-inspection with damage → damage_record created, linked to contract_line.`
- `Integration: damage charge_approved=true flows into final invoice.`

#### 5.2 — Maintenance scheduling and out-of-service lockout
**What**: Define calendar/meter maintenance schedules, generate due work orders, and block bookings during service.

**Design**:
- `maintenance_schedules.schedule_rules` JSONB defines triggers (`type: hours|calendar`, interval, `whichever_comes_first`).
- `core/maintenance/scanDue()` (runs as a daily worker job): for each active schedule, compare `assets.hour_meter_reading` and last completed work order date against triggers; when due, create a `work_orders` row (`work_order_type: 'preventive'`, status `open`).
- Scheduling a work order with `scheduled_start/end` → `availability.check` treats the window as a conflict; setting asset `status = out_of_service` removes it from the bookable pool entirely.
- `POST /work-orders/{id}/complete` records `work_details` (labor, parts) JSONB, updates `total_cost`, sets asset back to `available`, and resets the maintenance clock.

**Testing**:
- `Unit: asset at 248h, trigger every 250h → not due; at 252h → due.`
- `Unit: calendar trigger 90d, last service 91d ago → due.`
- `Integration: open work order overlapping a booking window → availability returns conflict.`
- `Integration: asset out_of_service → excluded from GET /availability.`
- `Integration: complete work order → asset available, hour_meter baseline reset.`

#### 5.3 — Operations dashboard and booking portal (web)
**What**: Next.js dashboard (reservations, dispatch schedule, upcoming returns, fleet status) and customer self-service booking portal.

**Design**:
- Dashboard `(dashboard)` routes (RBAC-gated): Today view (dispatches due, returns due, overdue), Fleet status board (available/on-rent/out-of-service counts), Asset detail, Contract detail with lifecycle actions.
- Portal `(portal)` routes (customer auth via `portal_enabled` accounts): browse availability calendar, request a booking, view/sign contracts, pay invoices (Stripe Elements).
- Server components fetch via the API using a server-side session; mutations via Server Actions calling the API.

**Testing**:
- `E2E (Playwright): operator logs in → Today view shows due dispatches.`
- `E2E: customer books available excavator → draft contract created, appears in dashboard.`
- `E2E: customer signs contract and pays deposit → contract reserved, hold placed.`
- `E2E: viewer role sees read-only dashboard (no action buttons).`

#### 5.4 — Offline-capable inspection PWA
**What**: A PWA surface for field staff to complete inspections offline and sync when reconnected.

**Design**:
- `(inspect)` route group registered as an installable PWA (Workbox service worker).
- Inspection data and photos queued in IndexedDB while offline; a background sync replays `PATCH /inspections` and photo uploads on reconnect, with conflict resolution by `inspected_at` timestamp.
- Photos captured at reduced resolution client-side before queueing.

**Testing**:
- `E2E (offline emulation): complete inspection offline → stored in IndexedDB; go online → synced, server record matches.`
- `Unit: queued photo compressed below size threshold before upload.`

---

## Phase 6: Public API, OpenAPI & MCP Server

### Purpose
Make the platform programmable and AI-orchestratable — a stated differentiator, since no
incumbent ships an open API or MCP server. Finalise the OpenAPI 3.1 contract, add API-key
auth for machine clients, and expose an MCP server over the rental domain.

### Tasks

#### 6.1 — OpenAPI 3.1 spec and API-key authentication
**What**: Generate and publish the OpenAPI 3.1 document; add scoped API keys for machine clients.

**Design**:
- `@fastify/swagger` produces `docs/openapi.json` (OAS 3.1, JSON Schema 2020-12) from route schemas; served at `/docs` (Swagger UI) and committed for SDK generation.
- API keys: `api_keys` table (org-scoped, hashed, scopes array). `Authorization: Bearer rem_sk_...`; same org-scope and RBAC guards apply (OWASP API #2/#5).
- Rate limiting per key via `@fastify/rate-limit` backed by Redis.

**Testing**:
- `Integration: GET /docs/openapi.json → valid OAS 3.1 (validated with a schema validator).`
- `Integration: request with valid API key → 200; revoked key → 401.`
- `Integration: API key scoped read-only calls POST → 403.`
- `Integration: exceed rate limit → 429 with Retry-After.`

#### 6.2 — MCP server over the rental domain
**What**: An MCP server exposing rental operations as tools for AI agents.

**Design**:
- `apps/mcp` using `@modelcontextprotocol/sdk`. Tools (each calling the public API with a scoped key):
  - `check_availability(category, start, end, depot?)`
  - `get_asset(asset_number)` / `search_assets(query)`
  - `list_maintenance_due(depot?)`
  - `get_contract(contract_number)`
  - `create_quote(customer, lines, start, end)`
  - `assess_damage(inspection_id)` (delegates to Phase 7 AI)
- Tool inputs/outputs declared as JSON Schema; resources expose read-only fleet snapshots.

**Testing**:
- `Integration: MCP check_availability tool → returns availability matching the REST endpoint.`
- `Integration: create_quote tool → draft contract created (verified via DB).`
- `Unit: tool input schema rejects malformed args before API call.`

---

## Phase 7: AI-Native Features

### Purpose
Deliver the differentiating AI capabilities from the README and features research: demand
forecasting, automated damage assessment, overdue-return prediction, and dynamic-pricing
recommendations. These run as worker jobs and surface recommendations into the dashboard
and MCP tools.

### Tasks

#### 7.1 — Demand forecasting
**What**: Forecast utilisation by category, season, and region for fleet-planning decisions.

**Design**:
- `packages/ai/forecasting`: builds a per-category time series from historical `contract_lines` (on-rent days per week) and fits Holt-Winters seasonal smoothing.
- Output: `{ categoryId, depotId, horizonWeeks, forecast: [{week, expectedOnRent, ciLow, ciHigh}], utilizationPct }`. An LLM narrates a recommendation ("consider acquiring 2 more 20t excavators for the Phoenix region ahead of Q3").
- Endpoint `GET /analytics/forecast?categoryId=&depotId=&weeks=12`; nightly worker refreshes cached forecasts.

**Testing**:
- `Unit: synthetic seasonal series → forecast tracks the seasonal pattern within CI bounds.`
- `Unit: <8 weeks of history → returns low-confidence flag, no LLM narration.`
- `Integration (mocked LLM): forecast endpoint returns narration string.`

#### 7.2 — Automated damage assessment (vision)
**What**: Compare pre/post inspection photos to recommend a damage charge.

**Design**:
- `packages/ai/damage`: on a post-inspection with flagged items, calls a vision LLM with the pre/post photo pairs and a structured prompt requesting `{ damage_type, severity, confidence, recommended_charge, rationale }`.
- Writes results into `damage_records.details.ai_assessment` JSONB (`model_version`, `confidence`, `recommended_charge`, `assessed_at`). A human approves before `charge_approved=true` (EU AI Act: human-in-the-loop for automated financial decisions).
- Prompt template enforces JSON output and instructs the model to return `confidence: 0` when photos are insufficient.

**Testing**:
- `Integration (mocked vision LLM): pre/post pair → ai_assessment written with recommended_charge.`
- `Unit: low-confidence (<0.5) result → not auto-applied, flagged for manual review.`
- `Integration: charge only enters invoice after human charge_approved=true.`

#### 7.3 — Overdue-return prediction
**What**: Score active rentals for late-return risk and surface high-risk ones for follow-up.

**Design**:
- `packages/ai/overdue`: features per active contract — customer late-return history, rental duration vs category norm, days-to-expected-return, prior extensions. Logistic-regression-style score (transparent weights, no opaque model) producing `risk: 0–1` + top contributing factors.
- `GET /analytics/overdue-risk` lists active contracts sorted by risk; nightly worker recomputes and triggers proactive reminder notifications above a threshold.

**Testing**:
- `Unit: customer with 3 prior late returns + long rental → high risk score.`
- `Unit: first-time customer, short rental → low risk.`
- `Integration: risk above threshold → reminder notification enqueued.`

#### 7.4 — Dynamic pricing recommendations
**What**: Recommend rate adjustments based on utilisation and demand (advisory, not auto-applied).

**Design**:
- Combines current category utilisation (from availability) with the 7.1 forecast to suggest a rate multiplier (`high utilisation + rising demand → +X%`; `low → -Y%`).
- `GET /analytics/pricing-recommendations` returns suggestions; an operator applies them by creating a new effective-dated `rate_table` (never auto-mutates live rates — financial safety + EU AI Act transparency).

**Testing**:
- `Unit: 95% utilisation + rising forecast → positive multiplier recommendation.`
- `Unit: 30% utilisation → discount recommendation.`
- `Integration: applying a recommendation creates a new effective-dated rate_table, leaves the old one intact.`

---

## Phase 8: Telematics Integration (AEMP 2.0)

### Purpose
Ingest real-time location and hour-meter data from mixed telematics providers via a single
normalised AEMP 2.0 / ISO 15143-3 pipeline, linking telemetry to assets and feeding
meter-based maintenance. This is a primary differentiator over mid-market incumbents.

### Tasks

#### 8.1 — Shared OAuth 2.0 + PKCE client
**What**: A reusable OAuth client for all integrations (telematics and accounting).

**Design**:
- `packages/integrations/oauth`: Authorization Code + PKCE (RFC 7636) per RFC 9700. Per-provider config (auth URL, token URL, scopes, client id/secret). Token storage encrypted (pgcrypto) in an `integration_credentials` table; automatic refresh-token rotation.
- `getAccessToken(orgId, provider)` transparently refreshes expired tokens (QuickBooks 1h, Xero 30m).

**Testing**:
- `Integration (mock IdP): full auth-code+PKCE exchange → tokens stored encrypted.`
- `Unit: expired access token → refresh invoked, new token returned and persisted.`
- `Unit: PKCE verifier/challenge generated per RFC 7636.`

#### 8.2 — AEMP 2.0 normalisation pipeline + provider adapters
**What**: Normalise Geotab and Samsara payloads into the AEMP schema and persist readings.

**Design**:
- `packages/integrations/telematics/aemp`: canonical `AempReading { equipmentId, datetime, location{lat,lon}, cumulativeOperatingHours, fuelUsed, distanceTravelled, status }`.
- Adapters: `geotab` (JSON-RPC `Get` for Device/StatusData, token session auth) and `samsara` (REST, bearer token) each implement `fetchReadings(since): AempReading[]`.
- A `telematics-sync` worker job (interval per `telematics_providers.config.sync_interval_seconds`) pulls per linked device, writes typed columns into the partitioned `telematics_readings`, stores the raw payload in `raw_data` JSONB, and updates `assets.hour_meter_reading`/`last_latitude`/`last_longitude`.
- Hour-meter updates trigger the meter-based maintenance scan (5.2).

**Testing**:
- `Unit: Geotab StatusData payload → correct AempReading.`
- `Unit: Samsara payload → correct AempReading.`
- `Integration (mocked provider): sync run → readings persisted to correct partition, asset hour_meter updated.`
- `Integration: hour-meter crossing a maintenance threshold → work order created.`

#### 8.3 — Asset map and location history
**What**: Surface real-time asset locations and movement history on the dashboard.

**Design**:
- `GET /assets/{id}/location` (latest) and `GET /assets/{id}/location/history?from=&to=` (from partitioned readings).
- Dashboard map view (Leaflet/MapLibre) plots `current_depot` and live asset positions; PostGIS used for nearest-depot and delivery-radius queries.

**Testing**:
- `Integration: location history over a range → ordered readings within bounds.`
- `Unit: nearest-depot PostGIS query returns closest active depot.`

---

## Phase 9: Accounting Sync, Notifications, COI & Multi-Depot

### Purpose
Complete the v1.1 "should-have" set and key backlog items: QuickBooks/Xero invoice sync,
automated customer notifications, certificate-of-insurance expiry tracking, credit limits,
and multi-depot inter-depot transfers.

### Tasks

#### 9.1 — QuickBooks & Xero invoice sync
**What**: Push invoices, payments, and customers to QuickBooks Online and Xero without duplication.

**Design**:
- `packages/integrations/accounting` with a common `AccountingProvider` interface (`upsertCustomer`, `createInvoice`, `recordPayment`) and `quickbooks`/`xero` implementations using the shared OAuth client (8.1).
- An `accounting-sync` worker job triggered on invoice `sent` and payment recorded; idempotency via the external id stored in `invoices.accounting_sync` JSONB (`{provider: {id, synced_at, sync_status}}`). Respects QuickBooks metered-read pricing by caching reads.
- Conflict/error → `sync_status: failed` + retry with backoff; surfaced in dashboard.

**Testing**:
- `Integration (mocked QBO): send invoice → external id stored in accounting_sync; re-sync → no duplicate (idempotent).`
- `Integration (mocked Xero): record payment → payment synced.`
- `Integration: provider 500 → status failed, retried with backoff.`

#### 9.2 — Automated notifications (email + SMS/WhatsApp)
**What**: Confirmation, dispatch reminder, overdue-return, and insurance-expiry notifications.

**Design**:
- `packages/integrations/messaging` with `email` (SMTP) and `twilio` (SMS/WhatsApp) channels; templates render into `notifications.content` JSONB.
- Event-driven enqueue (contract reserved → confirmation; day-before dispatch → reminder; overdue → escalation per `settings.notifications.overdue_reminder_days`). Twilio status callbacks update `notifications.status`.

**Testing**:
- `Integration (mocked Twilio/SMTP): contract reserved → confirmation enqueued and sent.`
- `Integration: overdue contract → reminders fire on configured day offsets.`
- `Integration (webhook): Twilio status callback (validated HMAC) → notification status updated.`

#### 9.3 — Certificate-of-insurance tracking and credit limits
**What**: Track customer COIs with expiry alerts, and block rentals when insurance expired or credit exceeded.

**Design**:
- COIs live in `customers.insurance_certificates` JSONB. A daily scan finds COIs expiring within `settings.notifications.insurance_expiry_warning_days` → notification.
- Contract creation guard: refuse `reserve` if no verified COI covers the rental period, or if `credit_limit` would be exceeded by open balances + this contract (configurable hard/soft block).

**Testing**:
- `Integration: COI expiring in 20 days (warn window 30) → alert enqueued.`
- `Integration: reserve with expired COI → 409 (hard block).`
- `Integration: contract exceeding credit_limit → 409 / warning per org config.`

#### 9.4 — Multi-depot inventory and inter-depot transfers
**What**: Book from the nearest available depot and manage in-transit inter-depot transfers.

**Design**:
- Availability (3.1) extended to consider all depots; `GET /availability` can return fulfil-from-another-depot options ranked by PostGIS distance.
- `POST /transfers` `{ asset_id, from_depot, to_depot }` → asset `status: in_transit`, excluded from booking until received; `POST /transfers/{id}/receive` → `current_depot_id` updated, asset available.

**Testing**:
- `Integration: no asset at requested depot but one at a nearby depot → availability suggests transfer.`
- `Integration: asset in_transit → excluded from availability until received.`
- `Integration: receive transfer → current_depot_id updated, asset available.`

---

## Phase 10: Hardening, Compliance & Release

### Purpose
Production-readiness: security hardening against the OWASP API Top 10, GDPR data-subject
tooling, audit completeness, observability, performance, and a documented release with
seed data and deployment guides for both self-hosted and cloud modes.

### Tasks

#### 10.1 — Security hardening (OWASP API Top 10 2023)
**What**: Systematic pass against the OWASP API risks.

**Design**:
- BOLA/BFLA: audit every route for org-scope + role checks (automated test that hits each route cross-org expecting 404/403).
- Excessive data exposure: response serializers whitelist fields; secrets/credentials never serialized.
- Security misconfig: helmet headers, CORS allowlist, no stack traces in prod.
- Inventory: `/docs` reflects every route; deprecated routes flagged.

**Testing**:
- `Integration (automated matrix): every mutating route called cross-org → 404; called by insufficient role → 403.`
- `Integration: error response in prod mode → no stack trace, generic message.`
- `Unit: serializer omits passwordHash, credentials, raw_data from public responses.`

#### 10.2 — GDPR & audit completeness
**What**: Data-subject export/erasure and a complete audit trail.

**Design**:
- `GET /customers/{id}/data-export` (GDPR portability) → JSON of all customer-linked records.
- `POST /customers/{id}/erase` → anonymises PII while preserving financial records (legal retention), logging the action to `audit_log`.
- Verify every create/update/delete on financial and contract entities writes an `audit_log` row with before/after.

**Testing**:
- `Integration: data-export → includes contracts, invoices, inspections for that customer only.`
- `Integration: erase → PII anonymised, invoice totals retained, audit_log entry written.`
- `Integration: contract status change → audit_log before/after recorded.`

#### 10.3 — Observability, performance & seed data
**What**: Structured logging, metrics, health checks, load validation, and demo seed data.

**Design**:
- Pino structured logs with request ids; `/health` (liveness) and `/ready` (DB/Redis/S3 checks); Prometheus `/metrics`.
- `pnpm db:seed` creates a demo org with depots, ~50 assets across categories, customers, rate tables, and historical contracts (also powers AI forecasting tests).
- Load test the availability and rate engines (k6) for a 500-asset fleet; add functional JSONB indexes where profiling shows hot paths.

**Testing**:
- `Integration: /ready returns 200 only when DB, Redis, S3 all reachable.`
- `Integration: db:seed → 50 assets, valid contracts, no constraint violations.`
- `Perf (k6): availability check p95 < 200ms at 500 assets / 100 rps.`

#### 10.4 — Deployment, docs & release
**What**: Finalise docker-compose / Helm, deployment guides, and a tagged release.

**Design**:
- `docker-compose.yml` brings up the entire stack; optional Helm chart for Kubernetes (cloud mode).
- Docs: self-hosted quickstart, env var reference, integration setup (OAuth app creation for QBO/Xero/Geotab/Samsara/Stripe), backup/restore, partition maintenance.
- CI (GitHub Actions): lint, typecheck, unit+integration (Testcontainers), build images, generate `openapi.json`. Tag `v1.0.0`.

**Testing**:
- `Smoke: fresh docker compose up + db:migrate + db:seed → login and complete a rental E2E.`
- `CI: full pipeline green on a clean checkout.`
- `Integration: generated openapi.json matches committed copy (drift check).`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (DB, auth, JSONB validation) ─── required by everything
    │
Phase 2: Asset Registry & Catalogue ─── requires 1
    │
Phase 3: Availability, Bookings & Contracts (CORE) ─── requires 2
    │
    ├── Phase 4: Billing, Invoicing & Payments ─── requires 3
    │       │
    │       └── Phase 5: Inspections, Maintenance & Dashboard ─── requires 4
    │               │
    │               ├── Phase 6: Public API & MCP ─── requires 3 (full surface after 5)
    │               │       │
    │               │       └── Phase 7: AI Features ─── requires 4,5 (data) + 6 (MCP exposure)
    │               │
    │               ├── Phase 8: Telematics (AEMP 2.0) ─── requires 2,5 (feeds maintenance)
    │               │
    │               └── Phase 9: Accounting, Notifications, COI, Multi-Depot ─── requires 4,5
    │
Phase 10: Hardening, Compliance & Release ─── requires all
```

**Parallelism opportunities** (after their dependencies are met):
- **Phases 6, 8, and 9 can be developed concurrently** once Phase 5 is complete — they touch different packages (`apps/mcp`, `integrations/telematics`, `integrations/accounting+messaging`) with minimal overlap.
- **Phase 7 (AI)** can begin its forecasting/overdue work (which need only Phase 4/5 data) in parallel with Phase 8, deferring only the MCP tool exposure until Phase 6 lands.
- Within Phase 5, the **PWA (5.4)** can be built in parallel with maintenance (5.2) once inspections (5.1) exist.

**MVP cut line**: Phases 1–5 constitute a shippable MVP matching features.md's "Must-have" set
(asset registry, booking portal with availability + e-signature, contract lifecycle,
inspections with damage charges, time-based invoicing + Stripe, calendar maintenance with
lockout). Phases 6, 8, 9 deliver the v1.1 "should-have" set; Phase 7 and the multi-depot/
analytics items deliver the "nice-to-have" backlog and AI-native differentiators.

---

## Definition of Done (per phase)

Every phase must satisfy all of the following before it is considered complete:

1. All tasks in the phase are implemented.
2. All unit and integration tests for the phase pass (`pnpm test`).
3. Integration tests run against real Postgres/Redis/MinIO via Testcontainers (not mocks) for data-layer behaviour.
4. Linting and formatting pass (`pnpm lint` / Biome).
5. Type checking passes (`pnpm typecheck` / `tsc --noEmit`, strict mode, zero errors).
6. The feature works end-to-end (Playwright E2E for any user-facing flow added in the phase).
7. New configuration options are added to `.env.example` and documented.
8. New API endpoints appear in the generated `docs/openapi.json` (OAS 3.1) with request/response schemas.
9. New or changed tables have a reviewed Drizzle migration; `pnpm db:migrate` runs cleanly on a fresh database.
10. JSONB writes added in the phase are validated by an Ajv JSON Schema and carry `_schema_version`.
11. Mutating operations on financial/contract entities write an `audit_log` row.
12. Docker images build successfully (`docker compose build`) and the stack starts (`docker compose up`).
13. New external integrations use the shared OAuth 2.0 + PKCE client and store credentials encrypted.
```
