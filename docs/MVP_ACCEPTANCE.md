# Commerce Operations CRM — MVP Acceptance

## 1. Purpose and release levels

This document defines bounded acceptance for the first client demonstration and the client MVP. It does not introduce enterprise-scale targets.

### Technical demonstration

```text
Live Shopify + Live WhatsApp + FakeCourierProvider
```

This release proves the workflow while Ozone access is unavailable. It must be labelled **Simulated courier — not Ozone** in the UI, shipment data, logs, and evidence. It is not a live Ozone acceptance.

### Client MVP

```text
Live Shopify + Live WhatsApp + Live Ozone Express
```

The client MVP is accepted only after official Ozone documentation, sandbox/test credentials, shipment creation, tracking, and status synchronization have passed the Ozone addendum. If those inputs are unavailable, the result remains the technical demonstration.

## 2. Acceptance governance

- Each case records build/version, environment, tester, timestamp, evidence, and result.
- Test data is synthetic or approved for the target environment.
- Provider credentials are server-side and never appear in browser source, API responses, logs, jobs, or audit metadata.
- The deployed MVP is single-tenant with one bootstrapped organization, but organization-scoped code paths are tested.
- The only MVP roles are `ADMIN` and `OPERATOR`.

## 3. Entry prerequisites

### 3.1 Common

- Web, API, worker, PostgreSQL, and Redis processes are healthy.
- Administrator bootstrap creates the one organization and first administrator.
- `ADMIN` and `OPERATOR` login, session, logout, and API authorization are configured.
- Health checks, masked structured logs, correlation IDs, transactional outbox, and background job processing are available.
- Basic backup procedure is documented.

### 3.2 Shopify

```env
SHOPIFY_SHOP_DOMAIN=
SHOPIFY_CLIENT_ID=
SHOPIFY_CLIENT_SECRET=
SHOPIFY_API_VERSION=2026-07
```

- The development app and store belong to the same Shopify organization.
- The backend uses the client-credentials grant, exchanges credentials server-side, caches the token, and renews it around its 24-hour expiry.
- No OAuth callback is required for this current store.
- The development store contains representative products, variants, locations, inventory, customers, and test orders.
- Required least-privilege scopes, webhook topics, and fulfilment permissions are approved.

### 3.3 WhatsApp

- Official Meta Cloud API access, phone/account identifiers, webhook verification, approved order-confirmation template, locale/variables, consent policy, and test recipient are configured server-side.

### 3.4 Courier

- Technical demonstration: `FakeCourierProvider` is enabled and visibly simulated.
- Client MVP: official Ozone readiness gate is complete and `OzoneCourierProvider` is enabled.

## 4. Reference test data

Prepare at least:

- two products with different variants and quantities;
- two Shopify locations with tracked inventory;
- multiple orders with distinct products, quantities, totals, and delivery addresses;
- one valid WhatsApp customer and one missing/invalid-phone customer;
- one order suitable for retry/failure and duplicate-protection scenarios.

Do not rely on 100,000 orders, 1,000,000 events, enterprise load, or production disaster simulation. Those are post-MVP verification.

## 5. Golden path

### AC-E2E-01 — Shopify order to courier tracking

1. Create a test order in Shopify.
2. Deliver a valid Shopify order webhook and process it asynchronously.
3. Confirm exactly one CRM order, customer reference, address snapshot, line snapshots, and source mappings.
4. Verify the order appears in the Orders screen and its timeline shows import and WhatsApp activity.
5. Verify the approved WhatsApp confirmation template is sent once and its provider message ID is stored.
6. An authorized employee confirms the order once.
7. Create one courier shipment only after confirmation/readiness.
8. Verify tracking and raw/normalized courier status appear in CRM.
9. Synchronize approved fulfilment/tracking information back to Shopify once.
10. Verify courier status changes remain distinct from Shopify financial, Shopify fulfilment, and CRM operational status.
11. Verify failures, retries, external events, and audit records remain visible.

The technical demonstration uses `FakeCourierProvider`; the client MVP uses live Ozone and must pass the Ozone cases below.

## 6. Authentication and authorization

### AC-AUTH-01 — Login and session

An active provisioned user can log in. The session is an opaque secure HTTP-only cookie backed by a revocable server-side record; it is not stored in local storage or URLs.

### AC-AUTH-02 — Failure and lifecycle

Invalid, disabled, expired, revoked, and stale sessions receive safe responses. Logout revokes the current session. Login is rate-limited and does not reveal whether an email exists.

### AC-AUTH-03 — Fixed role matrix

- `ADMIN` can manage users, inspect/configure environment-backed integrations, run supported synchronization, retry safe failures, confirm orders, and create shipments.
- `OPERATOR` can read operational screens and perform only the explicitly granted order/confirmation and operational actions defined by the MVP policy.
- No API call can create custom roles, edit permission mappings, or bypass the role policy.
- Direct API authorization matches UI authorization.

### AC-AUTH-04 — Organization scope and audit

Organization-scoped queries reject IDs from a seeded second test organization. Login success, user changes, order confirmation, message resend, shipment creation/retry, and manual synchronization create append-only audit records without secrets or unnecessary PII.

## 7. Shopify cases

### AC-SHP-01 — Client-credentials connection

Connection health obtains a token through the client-credentials grant, uses GraphQL Admin API `2026-07`, and renews an expired token automatically. The UI shows shop identity, configured/health state, API version, last synchronization, and safe errors; it never returns the client secret or access token.

### AC-SHP-02 — Import and rerun

From an empty CRM, import representative locations, products, variants, inventory, customers, and orders. Show progress and counts. Rerun the import and produce updates/no-ops, not duplicates.

### AC-SHP-03 — Duplicate webhook

Replay the same valid order webhook at least three times, including concurrent processing. One inbox identity and one business effect result; no duplicate order, line, customer source identity, or message intent is created.

### AC-SHP-04 — Inventory authority

After Shopify accepts an order and updates tracked inventory, CRM imports the resulting inventory state. Confirming or shipping in CRM performs no second inventory deduction or adjustment. Inventory events/reconciliation update the mirror and ledger only with synchronized source changes.

### AC-SHP-05 — Webhook security and reconciliation

Tampered/invalid signatures and unknown shop identities create no business mutation. A missed update is repaired by reconciliation. Older source events cannot overwrite newer source facts.

### AC-SHP-06 — Fulfilment/tracking idempotency

Repeated outbox/job delivery produces one approved Shopify fulfilment/tracking effect. Insufficient scope, validation, rate-limit, transient, and already-fulfilled errors remain visible and safe to retry or resolve.

## 8. Orders, catalog, inventory, and timeline

### AC-ORD-01 — Orders list

Orders supports search, pagination, operational status, Shopify financial status, Shopify fulfilment status, courier status, date, and synchronization filters. Each status is displayed separately.

### AC-ORD-02 — Details and snapshots

Order details show source facts, customer/address snapshot, lines, totals/currency, all status dimensions, message/shipment state, sync events, and a chronological timeline. Customer data is available here for operations; no dedicated customer-management screen is required.

### AC-ORD-03 — Confirmation

An authorized confirmation from `WAITING_CONFIRMATION` creates one CRM transition, actor/time audit record, and shipment-readiness intent. Repeated clicks or idempotency-key replays do not create another effect; stale versions return a conflict.

### AC-ORD-04 — Invalid transitions

Confirmation is required before shipment creation. Invalid or cancelled-order transitions are rejected without external side effects.

### AC-CAT-01 — Read-only catalog/inventory

Products, variants, SKUs, locations, quantities, and freshness match Shopify representative data. No catalog or inventory mutation action exists in the MVP. Stale data is visibly marked.

## 9. WhatsApp cases

### AC-WA-01 — Secret-safe configuration

The integration view shows configured/health state, non-secret identifiers, template mapping, last webhook, last synchronization, and safe errors. It never returns access tokens, app secrets, or verify tokens.

### AC-WA-02 — Confirmation template

An eligible imported order creates exactly one logical confirmation-message intent with a deterministic idempotency key. The accepted provider response stores the message ID and submitted time.

### AC-WA-03 — Status webhook and duplicates

Verified sent/delivered/read/failed events update the message timeline idempotently. Duplicate events and late lower-precedence events do not duplicate or regress the current projection.

### AC-WA-04 — Failure and resend

Invalid phone, missing template/consent, rate-limit, and provider failures do not block Shopify order import. Retryable failures retry with bounded backoff; manual resend requires authorization, creates an audit record, and does not create a second logical intent.

## 10. Courier and shipment cases

### AC-COURIER-01 — Confirmation prerequisite

The API and UI reject shipment creation before authorized confirmation/readiness. No provider operation is emitted.

### AC-COURIER-02 — Duplicate prevention

Repeated clicks, identical requests, duplicate outbox delivery, worker retry, and concurrent workers produce one primary shipment operation. Ambiguous timeouts reconcile before any resend; if the provider cannot support safe lookup/idempotency, the operation becomes `RECONCILIATION_REQUIRED`.

### AC-COURIER-03 — Status separation

Raw courier status, normalized courier status, and CRM operational status are stored separately. Unknown, out-of-order, and regressive statuses are retained and flagged without silently rewriting current state.

### AC-COURIER-04 — Fake-provider labelling

Technical-demo settings, shipment pages, tracking values, logs, and release evidence identify the provider as simulated. No fake result is presented as Ozone.

## 11. Operations, reliability, and security

### AC-OPS-01 — Durable work

A domain change and outbox record commit atomically. Restarting API/worker or losing Redis does not lose committed work; PostgreSQL state supports safe republish/reconciliation.

### AC-OPS-02 — Failed jobs and events

Authorized users can inspect provider, event type, attempts, correlation ID, sanitized error, and retry/reconciliation state. Eligible retry actions are audited.

### AC-OPS-03 — Security baseline

Argon2id, secure HTTP-only cookies, server-side role enforcement, environment validation, no committed secrets, webhook verification, input validation, authentication rate limiting, masked logs, and critical-action audit records pass verification.

### AC-OPS-04 — Backup

The documented basic PostgreSQL backup procedure is executed and a restore/integrity check is recorded for the controlled test environment. Automation and stronger disaster-recovery targets are post-MVP.

## 12. MVP frontend matrix

| Screen | Required acceptance |
|---|---|
| Login | AC-AUTH-01/02 |
| Dashboard | Operational counts, synchronization freshness, failures |
| Orders | AC-ORD-01 |
| Order details and timeline | AC-E2E-01, AC-ORD-02/04, message/shipment state |
| Catalog and inventory | AC-CAT-01 |
| Integration configuration and health | Shopify/WhatsApp/Ozone or fake-provider safe state |
| Integration events and failed jobs | AC-OPS-02 |
| Basic user management | AC-AUTH-03/04 for `ADMIN`/`OPERATOR` |

Every MVP screen has loading, empty, permission-denied, stale-data, validation, and recoverable-error states. It is keyboard usable with visible focus and readable status labels.

## 13. Ozone acceptance addendum

These cases are blocked until official Ozone documentation and sandbox/test credentials are approved. Do not invent any endpoint, header, request/response field, webhook signature, or status.

### AC-OZONE-01 — Contract traceability

Every implemented behavior references the approved official documentation version and an approved field/status mapping.

### AC-OZONE-02 — Connection and authentication

The official sandbox connection succeeds with server-side credentials. Invalid/expired credentials produce a safe blocked/terminal state.

### AC-OZONE-03 — Shipment creation

A representative confirmed order creates one official sandbox shipment with the approved request fields and stores the returned official shipment/tracking identifiers.

### AC-OZONE-04 — Duplicate and ambiguous-result safety

Repeated clicks, retries, and simulated timeouts follow Ozone's documented idempotency, client-reference, or lookup semantics. If safe reconciliation is impossible, creation stops and requires an explicit operational decision.

### AC-OZONE-05 — Tracking and status synchronization

The official webhook or polling mechanism is verified. Duplicate/out-of-order events are safe, raw and normalized statuses are stored, and approved status mappings update CRM and Shopify without conflating status domains.

Only after AC-OZONE-01 through AC-OZONE-05 pass may the release carry the **Client MVP — live Ozone** label.

## 14. Rejection conditions

Reject the release if it has duplicate CRM orders, duplicate WhatsApp logical sends, duplicate courier shipments, CRM inventory deductions, unverified webhook mutations, leaked credentials/PII, missing durable failures, shipment creation before confirmation, conflated statuses, fake courier presented as Ozone, guessed Ozone behavior, or an API authorization bypass.

## 15. Sign-off record

```text
Release label:
Environment:
Database migration:
Shopify shop / API version:
WhatsApp account / template:
Courier provider (FAKE_COURIER or OZONE):
Acceptance evidence:
Known accepted defects:
Product owner:
Operations representative:
Technical lead:
Deployment owner:
Date:
```
