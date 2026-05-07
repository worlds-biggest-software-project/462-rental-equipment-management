# Standards & API Reference

> Project: Rental Equipment Management · Generated: 2026-05-07

## Industry Standards & Specifications

### ISO Standards

**ISO/TS 15143-3:2020 — Earth-moving Machinery Worksite Data Exchange: Telematics Data (AEMP 2.0)**
- URL: https://www.iso.org/standard/76394.html
- The primary interoperability standard for mixed-fleet telematics. Defines a common JSON/XML payload structure for key parameters — position (latitude/longitude/timestamp), cumulative engine hours, total fuel consumed, and machine status — exposed over an HTTPS REST interface. All major OEMs (Caterpillar, John Deere, Komatsu, Volvo) and third-party telematics providers (Geotab, Samsara, Motive) implement this standard, enabling a rental management system to ingest asset telemetry from any compliant source via one normalised schema.

**ISO 55001:2014 (updated 2024) — Asset Management: Management Systems — Requirements**
- URL: https://www.iso.org/standard/55088.html (2014); https://www.iso.org/standard/83053.html (2024 revision)
- Defines requirements for an asset management system covering the full asset lifecycle: acquisition, operation, maintenance, and disposal. Provides the conceptual framework for equipment registry, condition tracking, maintenance scheduling, and disposal decisions that underpin a rental management platform. Companion standards ISO 55000 (vocabulary and overview) and ISO 55002 (implementation guidance) round out the series.

**ISO 27001:2022 — Information Security Management Systems**
- URL: https://www.iso.org/standard/27001
- The international standard for information security management. Required by enterprise rental operators evaluating SaaS vendors. Governs controls over customer data, access management, encryption, audit logging, and incident response. Leading rental software vendors (Texada, EZO) maintain ISO 27001 certification.

### W3C & IETF Standards

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://www.rfc-editor.org/rfc/rfc6749
- The foundational standard for delegated API authorization. Required for integrations with QuickBooks, Xero, Docusign, Geotab, and Samsara, all of which use OAuth 2.0. The Authorization Code + PKCE flow (per RFC 7636) is the recommended pattern for web and mobile applications.

**RFC 9700 — Best Current Practice for OAuth 2.0 Security (January 2025)**
- URL: https://datatracker.ietf.org/doc/rfc9700/
- Updated threat model and security guidance for OAuth 2.0 implementations, superseding earlier security advice in RFC 6819. Mandates PKCE for all Authorization Code flows, deprecates the Implicit and Resource Owner Password Credentials grants, and recommends sender-constrained tokens (DPoP or mTLS) for high-value integrations. Directly applicable when building the accounting and telematics integration layer.

**RFC 5545 — Internet Calendaring and Scheduling Core Object Specification (iCalendar)**
- URL: https://www.rfc-editor.org/rfc/rfc5545
- Defines the iCalendar data format for exchanging scheduling and availability information (VEVENT, VFREEBUSY components). Relevant for exporting equipment availability calendars to customer calendar applications (Google Calendar, Outlook) and for the driver dispatch scheduling module. RFC 7953 extends iCalendar with a VAVAILABILITY component for publishing available/unavailable time periods.

**RFC 7231 — Hypertext Transfer Protocol (HTTP/1.1): Semantics and Content**
- URL: https://www.rfc-editor.org/rfc/rfc7231
- Baseline HTTP semantics governing REST API design — GET, POST, PUT, PATCH, DELETE verbs and standard status codes. All external integrations (accounting, telematics, payment, e-signature) consume or expose HTTP REST APIs governed by this specification.

### Data Model & API Specifications

**OpenAPI Specification 3.1 (OAS 3.1)**
- URL: https://spec.openapis.org/oas/v3.2.0.html
- The industry-standard machine-readable format for describing REST APIs in JSON or YAML. Adopted JSON Schema 2020-12 in v3.1, ending schema fragmentation. Defines the contract-first design approach recommended for the platform's own public API. Enables auto-generation of client SDKs, interactive documentation, and API test suites. AI agents and integration platforms increasingly consume OpenAPI descriptions directly.

**JSON Schema 2020-12**
- URL: https://json-schema.org/specification
- Vocabulary for annotating and validating JSON documents. Used within OpenAPI 3.1 for request/response schema validation. Directly applicable to defining rental order, asset, and customer data models.

### Security & Authentication Standards

**OWASP API Security Top 10 (2023)**
- URL: https://owasp.org/API-Security/editions/2023/en/0x11-t10/
- Defines the ten most critical API security risks including Broken Object Level Authorization (BOLA), Broken Authentication, and Excessive Data Exposure. Essential reference for securing customer data, rental contract records, and telematics feeds against common API attack patterns.

**US ESIGN Act (Electronic Signatures in Global and National Commerce Act, 2000)**
- URL: https://www.govinfo.gov/content/pkg/PLAW-106publ229/pdf/PLAW-106publ229.pdf
- US federal law giving legal validity to electronic signatures and contracts, including equipment rental agreements and damage waivers. Requires a record of the signing party's intent to sign and their consent to do business electronically.

**EU eIDAS Regulation (910/2014)**
- URL: https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=uriserv:OJ.L_.2014.257.01.0073.01.ENG
- European Union framework for electronic signatures, establishing three tiers (Simple, Advanced, Qualified). Equipment rental contracts executed within the EU must comply with eIDAS; Simple Electronic Signatures satisfy the standard for most rental agreement contexts. Intersects with GDPR for customer personal data embedded in signed contracts.

**GDPR — General Data Protection Regulation (EU 2016/679)**
- URL: https://gdpr-info.eu/
- Applies to any rental platform processing personal data of EU customers, regardless of where the platform is hosted. Governs collection, storage, and retention of customer identity data, GPS location data for assets, and signed contract records. The EU AI Act (August 2026 compliance deadline) creates additional obligations for AI-driven features such as automated damage assessment or dynamic pricing.

**SOC 2 (AICPA Trust Services Criteria)**
- URL: https://www.aicpa-cima.com/resources/landing/soc-2
- The standard US security audit framework for SaaS companies handling customer data. SOC 2 Type II certification (covering Security, Availability, and Confidentiality trust criteria) is increasingly required by enterprise rental operators before procuring cloud software. Type II audit cost typically USD 25,000–50,000.

### MCP Server Specifications

**Model Context Protocol (MCP)**
- URL: https://modelcontextprotocol.io/
- Anthropic's open standard for exposing structured context and tool calls to AI models. Relevant for AI-native features: an MCP server wrapping the rental platform API would enable AI agents to query asset availability, check maintenance schedules, or trigger damage-charge generation through natural language workflows. Applicable to the demand forecasting, dynamic pricing, and automated damage assessment capabilities identified in the features research.

---

## Similar Products — Developer Documentation & APIs

### Geotab (MyGeotab API)
- **Description:** Market-leading fleet telematics platform providing GPS location, engine hours, fuel consumption, and diagnostics for mixed-brand fleets. Used by equipment rental operators to track asset location and meter-based maintenance intervals.
- **API Documentation:** https://developers.geotab.com/
- **API Reference:** https://geotab.github.io/sdk/software/api/reference/
- **SDKs/Libraries:** JavaScript, C# (.NET), Python (mygeotab-python on ReadTheDocs); GitHub: https://github.com/Geotab/sdk
- **Developer Guide:** https://geotab.github.io/sdk/software/guides/concepts/
- **Standards:** JSON-RPC over HTTPS (HTTP POST); AEMP 2.0 / ISO 15143-3 output available
- **Authentication:** API token (session-based); OAuth 2.0 supported for third-party integrations

### Samsara Fleet API
- **Description:** Cloud-connected operations platform providing real-time GPS tracking, safety monitoring, and equipment telematics for construction and industrial fleets. Widely used by rental operators alongside Geotab.
- **API Documentation:** https://developers.samsara.com/docs/rest-api-overview
- **API Reference:** https://developers.samsara.com/reference
- **SDKs/Libraries:** Python SDK (https://github.com/samsarahq/samsara-python); Node.js and Go SDKs available
- **Developer Guide:** https://developers.samsara.com/docs/getting-started
- **Standards:** REST/JSON; OpenAPI-described endpoints; AEMP 2.0 / ISO 15143-3 compliant output
- **Authentication:** Bearer tokens (API tokens from dashboard or OAuth access tokens)

### Caterpillar VisionLink / ISO 15143-3 API
- **Description:** Caterpillar's OEM telematics platform (VisionLink) exposes equipment location, hours, and fault codes via the standardised ISO 15143-3 (AEMP 2.0) API, enabling integration of Cat equipment data into third-party rental management systems.
- **API Documentation:** https://digital.cat.com/knowledge-hub/articles/iso-15143-3-aemp-20-api-developer-guide
- **FAQs:** https://digital.cat.com/knowledge-hub/faq/iso-15143-3-aemp-20-api-faqs
- **Standards:** ISO 15143-3 / AEMP 2.0; HTTPS GET; JSON and XML payloads
- **Authentication:** API key / OAuth 2.0 via Caterpillar Digital Marketplace

### John Deere Operations Center API
- **Description:** John Deere's developer platform exposing Equipment API endpoints for JDLink-connected machinery. Provides location, hours, and diagnostic data for John Deere construction and agricultural equipment in rental fleets.
- **API Documentation:** https://developer.deere.com/
- **Equipment API Docs:** https://developer.deere.com/dev-docs/equipment
- **Standards:** REST/JSON; OAuth 2.0 required; ISO 15143-3 compliant output
- **Authentication:** OAuth 2.0 (Authorization Code flow)

### QuickBooks Online API (Intuit)
- **Description:** Cloud accounting platform used by the majority of independent equipment rental businesses in North America. The REST API supports invoice creation, payment recording, customer sync, and deposit management — the key integration points for rental billing workflows.
- **API Documentation:** https://developer.intuit.com/app/developer/qbo/docs/develop
- **Standards:** REST/JSON; OpenAPI-described; webhook support for real-time sync
- **Authentication:** OAuth 2.0 (Authorization Code flow); access tokens expire after 1 hour; refresh token rotation required
- **Notes:** Writing data (Core API calls) is free; reading data (CorePlus calls) is metered under a tiered pricing model introduced in 2025.

### Xero Accounting API
- **Description:** Cloud accounting platform with strong adoption among equipment rental operators in the UK, Australia, and New Zealand. Supports invoice creation, credit notes, payment recording, and contact management via REST API.
- **API Documentation:** https://developer.xero.com/documentation/api/accounting/overview
- **Invoice Endpoints:** https://developer.xero.com/documentation/api/accounting/invoices
- **SDKs/Libraries:** Official SDKs for Python (xero-python), Node.js, .NET, PHP, Java, Ruby; GitHub: https://github.com/XeroAPI/xero-python
- **Standards:** REST/JSON; OAuth 2.0 required for all new integrations
- **Authentication:** OAuth 2.0 (Authorization Code flow); access tokens expire after 30 minutes; refresh token rotation required

### Docusign eSignature REST API
- **Description:** The leading e-signature platform. REST API enables embedding digital signature capture for rental agreements and damage waivers into the rental management workflow, with audit trail and tamper-evident document packaging.
- **API Documentation:** https://developers.docusign.com/docs/esign-rest-api/
- **REST API Reference:** https://developers.docusign.com/docs/esign-rest-api/reference/
- **Developer Center:** https://developers.docusign.com/
- **Standards:** REST/JSON; OAuth 2.0 (JWT Grant or Authorization Code); ESIGN Act and eIDAS compliant
- **Authentication:** OAuth 2.0 JWT Grant (server-to-server) or Authorization Code (user-interactive)

### Dropbox Sign (formerly HelloSign) API
- **Description:** Simpler, lower-cost e-signature alternative to Docusign with a clean REST API v3. Supports embedded signing, template-based workflows for standard rental agreements, and webhook delivery of signing events.
- **API Documentation:** https://developers.hellosign.com/
- **Standards:** REST/JSON; ESIGN Act and eIDAS compliant
- **Authentication:** API key or OAuth 2.0

### Stripe Payments API
- **Description:** Payment processing platform used by most equipment rental SaaS products for collecting deposits, rental charges, and damage fees. Supports pre-authorisation holds (critical for damage deposits), partial captures, and refunds — all essential patterns for equipment rental billing.
- **API Documentation:** https://docs.stripe.com/api
- **Payments Guide:** https://docs.stripe.com/payments
- **Standards:** REST/JSON; OpenAPI-described; webhook delivery of payment events; PCI DSS Level 1 compliant
- **Authentication:** API keys (publishable and secret); OAuth 2.0 for Connect (marketplace) use cases
- **Notes:** Payment Intents API is the recommended integration pattern; supports pre-auth holds via `capture_method: manual` for damage deposit workflows.

### EZRentOut (EZO) REST API
- **Description:** REST API for the EZRentOut rental management platform. Enables programmatic access to orders, assets, customers, and invoices for custom integrations or data migration.
- **API Documentation:** https://ezo.io/ezrentout/developers/
- **Standards:** REST/JSON
- **Authentication:** Token-based (access token in HTTP headers); available to paying customers only; trial accounts limited to ~1,000 requests/day

### Rentman API
- **Description:** REST API for the Rentman rental and production management platform. Covers projects, equipment reservations, customer records, and crew scheduling.
- **API Documentation:** https://api.rentman.net/
- **Developer Guide:** https://support.rentman.io/hc/en-us/articles/360013767839-The-Rentman-API
- **Standards:** REST/JSON
- **Authentication:** OAuth 2.0

### Booqable API v4
- **Description:** REST API for the Booqable online rental platform. Provides endpoints for products, orders, customers, and availability, supporting custom storefronts and third-party integrations.
- **API Documentation:** https://developers.booqable.com/
- **Source:** https://github.com/booqable/api-documentation
- **Standards:** REST/JSON; predictable resource-oriented URLs; standard HTTP verbs and status codes
- **Authentication:** HS256 signing tokens (keep secret; not for client-side use)

### Twilio Programmable Messaging API
- **Description:** Cloud communications platform used for automated SMS and WhatsApp notifications — rental confirmation, dispatch reminders, overdue-return alerts — with webhook delivery of inbound replies and message status callbacks.
- **API Documentation:** https://www.twilio.com/docs/messaging
- **Webhooks Reference:** https://www.twilio.com/docs/usage/webhooks/messaging-webhooks
- **Standards:** REST/JSON; webhook callbacks via HTTP POST; HMAC-SHA1 request validation
- **Authentication:** Account SID and Auth Token (HTTP Basic Auth); API keys for production use

---

## Notes

- **AEMP 2.0 / ISO 15143-3 is the cornerstone telematics integration standard.** Any rental platform intending to ingest GPS, hours, and fuel data from mixed OEM and third-party telematics sources should implement a single AEMP 2.0 ingestion pipeline rather than bespoke per-vendor adapters.
- **OAuth 2.0 (RFC 6749) + PKCE (RFC 7636) is universal.** Every major integration target — Geotab, Samsara, QuickBooks, Xero, Docusign, John Deere — requires OAuth 2.0. Implementing a shared OAuth client library with per-provider configuration is strongly recommended.
- **E-signature legal frameworks vary by jurisdiction.** Rental businesses operating in the EU must comply with eIDAS in addition to any local national requirements; those in the US rely on ESIGN/UETA. Where both apply, the more stringent (eIDAS) should govern.
- **No open-source rental management platform with a published API was identified.** The entire competitive landscape is proprietary SaaS, leaving a clear gap for an open-core or open-source offering with a well-documented public API.
- **MCP integration is an emerging differentiator.** Exposing a Model Context Protocol server over the rental management API would allow AI agents to orchestrate rental workflows — availability checks, maintenance scheduling, damage assessment — without custom tool development, positioning the platform ahead of incumbents in the AI-native space.
