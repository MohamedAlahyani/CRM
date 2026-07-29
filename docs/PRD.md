# Commerce Operations CRM — Product Requirements Document

## 1. Document purpose

This document defines the product boundary and requirements for the first client-testable release of a self-hosted commerce operations CRM. It is the product baseline for design, implementation, testing, and acceptance.

The MVP integrates Shopify, the official WhatsApp Business Cloud API, and a courier abstraction. A live Ozone Express adapter is gated on official Ozone documentation and test credentials. Until those inputs exist, local and client workflow testing may use a visibly labelled `FakeCourierProvider`.

## 2. Product vision

E-commerce operations employees should manage the lifecycle from new order through delivery from one system instead of switching between Shopify, courier portals, spreadsheets, and WhatsApp.

The CRM is an independent application. It is not embedded in Shopify Admin. Shopify continues to own storefront, checkout, and original commerce records; the CRM owns the operational workflow, communication, delivery orchestration, synchronization visibility, and employee audit history.

## 3. Deployment and commercial model

- MVP deployment: self-hosted, one client installation, one organization.
- Commercial model: one-time licence.
- Architecture requirement: every business record is organization-scoped so future multi-tenancy is feasible.
- Explicit non-goal: tenant self-service, tenant isolation controls for shared infrastructure, subscriptions, metering, and SaaS billing are not implemented in the MVP.
- The initial organization is provisioned through a deployment/bootstrap procedure, not a public sign-up flow.

## 4. Current baseline

- A Shopify development app named `CRM Connector` exists outside this repository.
- It is configured as non-embedded, has a published version, and is installed on a Shopify test store.
- Shopify Client ID, Client Secret, and permanent `.myshopify.com` domain are expected through server-side environment configuration.
- The current development app and development store belong to the same Shopify organization, so the MVP uses Shopify's client-credentials grant. The backend exchanges the client credentials for a server-side access token, caches it securely, and renews it around its 24-hour expiry.
- The pinned Admin API is Shopify GraphQL `2026-07`; no OAuth callback is required for this current same-organization development store.
- Source of truth: [Shopify client-credentials grant](https://shopify.dev/docs/apps/build/authentication-authorization/access-tokens/client-credentials-grant) and [GraphQL Admin API `2026-07`](https://shopify.dev/docs/api/admin-graphql/2026-07).
- This repository was empty at the start of the documentation run: no web, API, worker, database, migration, integration, or infrastructure implementation exists.
- No WhatsApp or Ozone credentials/contracts are present.

## 5. Future-capability context

After the MVP, the product vision includes:

- YouCan and WooCommerce commerce adapters.
- Vitex and Ameex courier adapters.
- Inbound WhatsApp conversations, AI-assisted support, and explicit human handoff.
- Multi-channel inventory views and policies.
- Deeper returns/refusals workflows and operational analytics/reporting.
- Shared-infrastructure multi-tenancy and SaaS billing.

These items influence provider boundaries and organization scoping only. They are not authorized implementation scope in the MVP.

## 6. Users and roles

The MVP has exactly two roles:

| Role | Primary responsibilities |
|---|---|
| Administrator | Configure environment-backed integrations, manage users, inspect all operational data, retry safe failures, and perform all MVP actions |
| Operator | View orders/catalog/inventory and confirm orders; shipment creation, message resend, and other consequential actions are explicitly permission-checked within the fixed MVP policy |

Roles are organization-scoped and enforced by the API. Editable permission systems, custom roles, and additional role templates are post-MVP.

## 7. Source-of-truth model

| Data or state | System of record | CRM behavior |
|---|---|---|
| Shopify order facts | Shopify | Mirror and update from Shopify events/reconciliation |
| Shopify financial status | Shopify | Store separately; never infer CRM operational status from it |
| Shopify fulfilment status | Shopify | Store separately and update when valid fulfilment actions succeed |
| Product, variant, location, tracked inventory | Shopify | Mirror; do not independently decrement Shopify stock |
| CRM operational status | CRM | Change only through allowed CRM transitions and recorded automation |
| Shipment/tracking identifier | Courier | Mirror after confirmed courier creation |
| Courier movement/status | Courier | Preserve raw value and map to a CRM-normalized courier status |
| WhatsApp delivery status | WhatsApp | Mirror message lifecycle events |
| Users, permissions, audit records | CRM | Authoritative |

## 8. MVP outcome

A test user can demonstrate this end-to-end path:

1. A customer places an order in Shopify.
2. Shopify sends a verified webhook and the order appears once in the CRM.
3. Customer, order lines, and relevant catalog/inventory references are visible.
4. A configured, approved WhatsApp confirmation template is queued and sent once.
5. An authorized employee reviews and confirms the order.
6. The order becomes ready for shipment.
7. The CRM requests shipment creation from the configured courier provider.
8. With the fake provider, or with Ozone only after its live contract is approved, one shipment and tracking number appear.
9. Courier status changes are ingested or polled and shown in the timeline.
10. Relevant tracking and fulfilment information is sent to Shopify.
11. Failures, retries, user actions, and external events remain inspectable.

## 9. Scope

### 9.1 Included in the MVP

- Administrator bootstrap, secure login/logout, server-side sessions, and the fixed `ADMIN`/`OPERATOR` role policy.
- One bootstrapped organization per installation.
- Shopify connection configuration and health.
- Initial and incremental import of Shopify products, variants, locations, inventory levels, customers, orders, and required fulfilment references.
- Verified, durable, idempotent Shopify webhook ingestion.
- Order list, details, operational state transitions, and timeline.
- Customer data needed by order import and order details; dedicated customer-management screens are deferred.
- Read-only catalog and inventory views by Shopify location.
- Official WhatsApp Business Cloud API outbound approved-template messages and message status webhooks.
- Manual order confirmation before shipment creation.
- `CourierProvider`, `FakeCourierProvider`, and an `OzoneCourierProvider` skeleton gated by official documentation.
- Ozone shipment creation, tracking, and status synchronization only for the client-MVP release level after the Ozone gate passes.
- Shipment/tracking visibility in order details and the operational timeline.
- Appropriate tracking/fulfilment synchronization back to Shopify.
- Background jobs, retries, dead-letter visibility, synchronization health, and manual retry controls.
- Dashboard with the minimum operational counts and failure indicators.
- Integration-event visibility, failed-job visibility/retry, critical-action audit records, health endpoints, and structured logs.

### 9.2 Excluded from the MVP

- YouCan, WooCommerce, Vitex, and Ameex implementations.
- Inbound WhatsApp conversations, shared inbox, free-form support chat, and human handoff.
- AI agent or AI-assisted support.
- Multi-tenant SaaS operation, public organization registration, billing, plans, and usage metering.
- Shopify embedded application UI.
- CRM-originated product/catalog editing.
- CRM-managed stock deductions, reservations, purchasing, or warehouse transfers.
- Editable permission systems, roles beyond `ADMIN` and `OPERATOR`, and a dedicated customer-management area.
- Advanced analytics, large reporting dashboards, custom dashboard builders, and native mobile applications.
- Advanced privacy-operation workflows and per-field customer-data encryption.
- Large-scale performance targets and disaster-recovery automation beyond a documented basic backup procedure.
- Advanced BI, accounting, invoicing, payment processing, marketing automation, and returns-merchandise authorization workflows.
- Mobile-native applications.
- Live Ozone behavior based on assumptions or reverse-engineering.

## 10. Operational state machine

The CRM operational status is independent of Shopify and courier statuses.

```text
NEW
  → WAITING_CONFIRMATION
    → CONFIRMED
      → READY_FOR_SHIPMENT
        → SHIPPED
          → OUT_FOR_DELIVERY
            → DELIVERED
            → DELIVERY_FAILED
            → REFUSED
          → DELIVERY_FAILED
          → REFUSED
          → RETURNED

NEW / WAITING_CONFIRMATION / CONFIRMED / READY_FOR_SHIPMENT
  → CANCELLED

DELIVERY_FAILED / REFUSED
  → RETURNED
```

The precise automatic mapping from Ozone statuses to these transitions is blocked until the official Ozone status catalogue is supplied. Invalid or regressive external transitions are retained as shipment events and flagged for review rather than silently rewriting CRM state.

| Status | Meaning and entry rule |
|---|---|
| `NEW` | Shopify order was first persisted; CRM eligibility checks have not completed |
| `WAITING_CONFIRMATION` | Import checks completed and the order awaits an employee decision; confirmation-message failure does not prevent this state |
| `CONFIRMED` | An authorized employee confirmed the order and the action/version was recorded |
| `READY_FOR_SHIPMENT` | Generic and configured-provider readiness checks passed and one shipment intent exists |
| `SHIPPED` | An approved courier mapping establishes that the parcel entered the shipment flow; creating a label/tracking record alone does not imply physical handover unless the approved provider contract says so |
| `OUT_FOR_DELIVERY` | Approved courier status indicates an active final delivery attempt |
| `DELIVERED` | Approved courier terminal status indicates successful delivery |
| `DELIVERY_FAILED` | Delivery attempt failed without an explicit customer refusal; retry/return policy may still be required |
| `REFUSED` | Approved courier status explicitly indicates customer refusal |
| `RETURNED` | Approved courier status indicates the shipment returned to the merchant/origin according to the provider contract |
| `CANCELLED` | CRM operational handling stopped before an active delivery lifecycle; this does not by itself claim Shopify or courier cancellation |

An imported eligible order transitions `NEW → WAITING_CONFIRMATION` as part of idempotent order processing. `WAITING_CONFIRMATION → CONFIRMED` is always an authorized human action in the MVP. Courier-driven transitions occur only through a reviewed raw-to-normalized mapping.

## 11. Functional requirements

Requirement identifiers are stable references for acceptance tests.

### 11.1 Authentication and organization

- **FR-AUTH-01:** A provisioned active user can sign in with email and password.
- **FR-AUTH-02:** Authentication uses an opaque secure HTTP-only cookie and a revocable server-side session.
- **FR-AUTH-03:** A user can sign out of the current session; administrators can revoke a user's sessions.
- **FR-AUTH-04:** Disabled users cannot establish or continue sessions.
- **FR-AUTH-05:** Every business query and mutation is scoped to the authenticated organization.
- **FR-AUTH-06:** Authorization is enforced by the API using permissions, not hidden UI elements alone.
- **FR-AUTH-07:** Administrators can create, update, disable, and assign only the fixed `ADMIN` and `OPERATOR` roles to users.
- **FR-AUTH-08:** The API rejects role or permission mutations outside the fixed MVP policy; editable permission systems are post-MVP.

### 11.2 Dashboard

- **FR-DASH-01:** Show counts for new/waiting, confirmed/ready, shipped/in transit, delivered, delivery exceptions, and integration failures.
- **FR-DASH-02:** Dashboard filters use an explicit organization time zone and date range.
- **FR-DASH-03:** Each count links to the corresponding filtered operational list.
- **FR-DASH-04:** Data freshness and the latest successful synchronization are visible.

### 11.3 Shopify synchronization

- **FR-SHP-01:** An administrator can inspect Shopify connection and synchronization health without exposing credentials.
- **FR-SHP-02:** The system can run an initial, resumable import of required Shopify resources.
- **FR-SHP-03:** Incremental updates are accepted through verified webhooks and supplemented by scheduled reconciliation.
- **FR-SHP-04:** Duplicate delivery of the same webhook produces one business effect.
- **FR-SHP-05:** The same Shopify order appears once for a given Shopify connection.
- **FR-SHP-06:** Raw external identifiers and sufficient source metadata are retained for diagnosis.
- **FR-SHP-07:** Deletions/redactions from Shopify are handled according to Shopify requirements and the approved privacy policy.
- **FR-SHP-08:** Inventory is mirrored by Shopify location and variant; CRM order handling never performs a second stock deduction.
- **FR-SHP-09:** Tracking/fulfilment updates are sent to Shopify only when prerequisites and permissions are satisfied.
- **FR-SHP-10:** Bulk sync progress, record counts, cursor, last success, and last error are visible.

### 11.4 Orders

- **FR-ORD-01:** Operators can list, search, filter, sort, and paginate orders.
- **FR-ORD-02:** Filters include operational status, financial status, fulfilment status, date, shipment status, and synchronization health.
- **FR-ORD-03:** Order detail shows source facts, customer/address snapshot, line snapshots, totals/currency, statuses, messages, shipment, sync events, and audit timeline.
- **FR-ORD-04:** Source snapshots preserve the facts used operationally even if the customer or catalog later changes.
- **FR-ORD-05:** Authorized users can make only allowed operational transitions.
- **FR-ORD-06:** Confirmation records actor and time and can occur only once per order version.
- **FR-ORD-07:** Repeated confirmation requests are idempotent.
- **FR-ORD-08:** Cancellation does not silently cancel Shopify or courier records; outbound side effects require explicit supported workflows.
- **FR-ORD-09:** Concurrency conflicts return a clear error and do not overwrite a newer change.

### 11.5 Customers, products, and inventory

- **FR-CUS-01:** Authorized order-detail views can show the imported customer contact, address, and related-order references needed for operations; a dedicated customer-management screen is post-MVP.
- **FR-CUS-02:** Customer duplicates are prevented per external source identity; no automatic cross-source identity merge is in MVP.
- **FR-CAT-01:** Users can search and view Shopify products, variants, SKUs, and source state.
- **FR-INV-01:** Users can view available inventory by variant and Shopify location with last-synchronized time.
- **FR-INV-02:** Stale inventory is visibly marked.
- **FR-INV-03:** MVP catalog and inventory screens are read-only.

### 11.6 WhatsApp messaging

- **FR-MSG-01:** An administrator can inspect WhatsApp connection health and configured template mappings without seeing secrets.
- **FR-MSG-02:** The system can send only configured approved templates in the MVP.
- **FR-MSG-03:** Order confirmation messaging uses a deterministic idempotency key so one trigger does not create duplicate CRM messages.
- **FR-MSG-04:** Message records expose queued, submitted, sent, delivered, read, and failed states when reported by WhatsApp.
- **FR-MSG-05:** Signed/verified WhatsApp webhook events are stored durably and processed idempotently.
- **FR-MSG-06:** Missing consent, invalid phone number, missing template, or provider failure is visible and does not block order import.
- **FR-MSG-07:** Operators can retry eligible failed messages subject to permission and rate limits.

### 11.7 Shipments and courier

- **FR-SHPMT-01:** A shipment cannot be requested before authorized order confirmation and readiness.
- **FR-SHPMT-02:** Repeated clicks, requests, or job retries cannot intentionally create two courier shipments for the same shipment request.
- **FR-SHPMT-03:** Shipment creation stores the request operation, response reference, tracking identifier, and timestamps without logging sensitive payloads.
- **FR-SHPMT-04:** Shipment and tracking events are visible on order and shipment timelines.
- **FR-SHPMT-05:** Raw courier statuses are preserved separately from normalized courier and CRM operational statuses.
- **FR-SHPMT-06:** Unsupported or unknown courier statuses are retained and flagged.
- **FR-SHPMT-07:** Ambiguous shipment creation timeouts enter a reconciliation-required state; the system must not blindly retry when duplication cannot be excluded.
- **FR-SHPMT-08:** `FakeCourierProvider` is clearly marked in settings, logs, shipment data, and test evidence.
- **FR-SHPMT-09:** Live Ozone is disabled until its readiness gate is passed.

### 11.8 Reliability and operations

- **FR-OPS-01:** All external side effects originate from a transactional outbox record.
- **FR-OPS-02:** Retryable failures use bounded exponential backoff with jitter and a configured maximum.
- **FR-OPS-03:** Terminal failures are queryable and manually retryable where safe.
- **FR-OPS-04:** Integration events expose provider, event type, received time, processing state, attempts, and sanitized error.
- **FR-OPS-05:** Health checks distinguish process liveness from dependency readiness.
- **FR-OPS-06:** Every request, job, webhook, and outbound operation has a correlation identifier.
- **FR-OPS-07:** Administrators can trigger supported reconciliation without creating duplicate effects.

### 11.9 Audit

- **FR-AUD-01:** Security-sensitive and operational mutations record organization, actor, action, target, time, correlation ID, and sanitized before/after metadata.
- **FR-AUD-02:** Audit records are append-only to application users.
- **FR-AUD-03:** Audit log access is permission-gated, searchable, filterable, and paginated.
- **FR-AUD-04:** Secret values, passwords, session tokens, message bodies containing unnecessary PII, and full provider credentials never enter audit metadata.

## 12. Frontend requirements

| Screen | Minimum MVP capability |
|---|---|
| Login | Email/password authentication, generic errors, rate-limit feedback |
| Dashboard | Operational counts, failure summary, sync freshness |
| Orders | Search, filters, pagination, status badges, sync indicators |
| Order details | Facts, independent statuses, confirmation action, shipment/message panels, unified timeline |
| Catalog and inventory | Read-only product/variant and location inventory view |
| Integration configuration and health | Configured/not configured, connection health, last synchronization, and safe errors; never existing secrets |
| Integration events | Inbox/outbox processing state, sanitized errors, retry action |
| Failed jobs | Queue/job class, attempts, failure, next action |
| Users and roles | User lifecycle, role and permission assignment |

All screens require loading, empty, permission-denied, stale-data, validation, and recoverable error states. Destructive or externally consequential actions require clear confirmation.

## 13. Non-functional requirements

- **NFR-01 Availability:** A single process restart does not lose a webhook accepted with a success response or an outbox event committed to PostgreSQL.
- **NFR-02 Performance:** Core lists are paginated and responsive on the development-store dataset; large-scale performance targets are post-MVP.
- **NFR-03 Webhook response:** After signature verification and durable inbox write, webhook endpoints respond within 5 seconds and defer business processing.
- **NFR-04 MVP verification:** Validate the development-store dataset, multiple orders with different products and quantities, duplicate/retry scenarios, token renewal, and database/idempotency constraints. Enterprise-scale load testing is post-MVP.
- **NFR-05 Consistency:** Domain updates and outbox writes are atomic; consumer processing is at-least-once and handlers are idempotent.
- **NFR-06 Recovery:** PostgreSQL backups are automated and restore-tested before production acceptance. Target RPO is 24 hours and target RTO is 4 hours unless the client contracts stricter values.
- **NFR-07 Security:** Meet the controls in `SECURITY.md`, including Argon2id, secure cookies, CSRF protection, least privilege, secret redaction, encryption in transit, and dependency scanning.
- **NFR-08 Privacy:** Collect and expose only operationally necessary personal data; retention and deletion rules require client approval before production.
- **NFR-09 Accessibility:** Core workflows target WCAG 2.2 AA, including keyboard operation, focus visibility, labels, errors, and contrast.
- **NFR-10 Browser support:** Current and previous major versions of Chrome, Edge, Firefox, and Safari at release time.
- **NFR-11 Observability:** Structured logs, metrics, health, correlation, job state, sync state, and audit data permit diagnosis without credential or customer-data leakage.
- **NFR-12 Maintainability:** Provider-specific code remains in adapters and is contract-tested; domain modules do not import provider SDK types.
- **NFR-13 Deployability:** Local development runs through Docker Compose; production uses versioned images, migrations as a controlled release step, and documented rollback.
- **NFR-14 Time handling:** Persist instants in UTC; render in organization time zone; retain source time zone/offset where legally or operationally relevant.
- **NFR-15 Monetary handling:** Persist monetary values in integer minor units plus ISO currency; never use binary floating point.

## 14. Product success indicators

- Imported Shopify orders have no duplicate CRM order records.
- No duplicate shipment is attributable to an ordinary repeated click or worker retry.
- At least 99% of valid verified webhooks are processed without manual action during acceptance testing.
- Operators can identify the reason and recovery action for every injected provider failure.
- The full acceptance workflow can be completed without using a spreadsheet.
- Median time from CRM order visibility to operator confirmation is measurable.
- Shipment delivery/refusal rates and integration failure counts are reportable from normalized data.

These are MVP validation indicators, not contractual production service levels.

## 15. Assumptions requiring confirmation

- MVP permits one primary shipment per order; partial and split shipments are not an accepted workflow even though the data model allows multiple shipments later.
- The client supplies the organization name, time zone, default locale, admin email, and secure bootstrap password procedure.
- Confirmation template sending is automatic on new eligible order; if the client wants operator-triggered sending, the workflow and acceptance tests must change.
- Customer phone normalization uses an agreed default country and E.164 rules.
- Shopify remains the inventory authority and the CRM provides a read-only mirror.
- Updating Shopify fulfilment is permitted for the store and compatible with the store's fulfilment-order ownership model.

## 16. Contradictions, missing inputs, and product decisions

### Blocking external inputs

1. **Shopify access:** The MVP uses `SHOPIFY_SHOP_DOMAIN`, `SHOPIFY_CLIENT_ID`, and `SHOPIFY_CLIENT_SECRET` server-side. The API token is obtained with client credentials, expires after 24 hours, and is renewed automatically. No callback is required for the current same-organization store.
2. **Future Shopify authentication:** If the app is later installed on stores belonging to unrelated merchant organizations, implement Shopify's authorization-code flow. That onboarding is post-MVP and must not block this test-store integration.
3. **Shopify contract:** Required least-privilege scopes, webhook subscriptions, privacy/compliance webhooks, fulfilment-order behavior, and test-store scenarios must be confirmed against GraphQL Admin API `2026-07`.
3. **WhatsApp onboarding:** Meta app ID/secret, business account ID, phone number ID, permanent system-user access token, webhook verify token, approved templates/locales/variables, recipient consent policy, and test numbers are absent.
4. **Ozone:** Official base URLs, authentication, endpoint schemas, status catalogue, webhook/polling behavior, idempotency support, rate limits, sandbox, and test credentials are absent. Live implementation and acceptance are blocked.
5. **Privacy:** Data retention, deletion/anonymization, subject-request process, backup retention, data residency, and controller/processor responsibilities need client/legal approval.
6. **Operations:** Production host, domain, TLS termination, password recovery method, backup destination, monitoring destination, incident contacts, and basic backup procedure are not supplied.

### Decisions needed before the affected milestone

- Automatic versus manual initial WhatsApp confirmation trigger.
- Allowed order-edit behavior after Shopify import.
- Whether cancelled/refunded Shopify changes may automatically transition eligible CRM orders to `CANCELLED`.
- Policy for repeat delivery attempts after `DELIVERY_FAILED`.
- Whether refusal and return are distinct courier events for the selected Ozone service.
- Which Shopify event should mark fulfilment and whether delivery status should ever close/archive an order in Shopify.
- Required dashboard periods, business time zone, and report export format.
- Password recovery approach for a self-hosted deployment.

None of these unanswered items authorizes expanding the MVP. The implementation plan places each behind an explicit decision or readiness gate.
