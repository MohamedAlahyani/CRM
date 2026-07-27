# Commerce Operations CRM — MVP Acceptance

## 1. Purpose and release labels

This document defines objective acceptance for the first client-testable MVP.

Two distinct release labels are allowed:

1. **Core MVP — simulated courier:** Shopify and WhatsApp Cloud are live/test-sandbox integrations; courier behavior uses the clearly labelled `FakeCourierProvider`.
2. **Core MVP — live Ozone:** all core criteria pass and the Ozone addendum passes against the official Ozone test environment.

The first label must never be described as a live Ozone integration. If Shopify or WhatsApp prerequisites are unavailable, their affected cases remain blocked and the release is not an accepted end-to-end MVP.

## 2. Acceptance governance

- Product owner, operations representative, technical lead, and deployment owner record sign-off.
- Every case records build/image version, environment, date/time, tester, evidence, and result.
- `PASS` means expected outcome occurred with no critical defect.
- `FAIL` means expected outcome did not occur or data/security integrity was compromised.
- `BLOCKED` is permitted only for a documented missing external dependency and is not equivalent to pass.
- Severity 1 or 2 defects block release. Lower-severity defects require recorded disposition.
- Test customer data must be synthetic or approved for the environment.
- Provider credentials must be test/sandbox credentials where supported; production effects require explicit controlled approval.

## 3. Entry prerequisites

### 3.1 Common environment

- Versioned web, API, and worker images deployed from one release.
- PostgreSQL and Redis are private, healthy, and use persistent/recoverable configuration.
- TLS and production cookie policy are active.
- Database migrations and secure organization/admin bootstrap completed.
- Logs/metrics/alerts are accessible to authorized operators.
- Encrypted backup completed and a restore drill has passed for the release candidate or compatible schema.
- Organization name, time zone, locale, default phone country, roles, and permissions are configured.

### 3.2 Shopify

- Approved Shopify authorization method has yielded usable server-side Admin API access.
- Shop domain, pinned API version, granted scopes, webhook topic matrix, and callback URL are recorded.
- App is installed on the designated test store.
- Test store supports representative COD/order, customer, product, inventory, cancellation, and fulfilment scenarios.

### 3.3 WhatsApp

- Approved Meta app/business configuration, server-held token, phone number ID, app secret, and webhook verification settings are configured.
- At least one order-confirmation template with locale and exact parameters is approved.
- Approved test recipients and consent policy are recorded.

### 3.4 Courier

- For simulated acceptance: active `FAKE_COURIER` connection and visible simulation warning.
- For live Ozone acceptance: every readiness item in `INTEGRATIONS.md` is approved and sandbox credentials are configured.

## 4. Reference test data

Create:

- Administrator, operations manager, order operator, fulfilment operator, and read-only analyst users.
- A second test organization used only for automated isolation tests.
- Shopify product with at least two variants.
- At least two Shopify locations with distinct inventory.
- Customer with valid E.164 WhatsApp number and complete shipping address.
- Customer with invalid/missing phone to test non-blocking message failure.
- New COD order with two shippable lines.
- Cancelled order.
- Already fulfilled or partially allocated order representative of the store's fulfilment model.
- Provider failure scripts/fixtures for retryable, terminal, rate-limit, out-of-order, duplicate, and ambiguous-result behavior.

No test relies on undocumented Ozone fields/statuses.

## 5. End-to-end golden path

### AC-E2E-01 — Shopify order to tracked fulfilment

**Given**

- Shopify, WhatsApp, and selected courier provider are healthy;
- a valid approved confirmation template is configured;
- no CRM record exists for the new Shopify order;
- the operator has confirmation and shipment permissions.

**When**

1. A customer places the representative Shopify order.
2. Shopify delivers the order event.
3. The operator opens the order and confirms it.
4. The worker creates the shipment.
5. Courier status advances to out for delivery and delivered.

**Then**

- the order appears once in the CRM;
- customer, address snapshot, lines, amounts, source timestamps, Shopify financial status, Shopify fulfilment status, CRM operational status, and courier status are correct and separate;
- Shopify inventory values are mirrored and are not decremented by CRM confirmation or shipment;
- one confirmation message intent exists, has a provider message ID when accepted, and reflects reported delivery statuses;
- one active primary shipment exists with provider identity/tracking;
- order/shipment timeline shows import, message, employee confirmation, shipment creation, courier events, and Shopify outbound result with actors/times;
- tracking/fulfilment appears in Shopify once according to the approved fulfilment policy;
- no credential or unredacted customer payload appears in browser source, API errors, or logs;
- all asynchronous operations finish or expose a clear recoverable/terminal status.

## 6. Authentication, authorization, and organization cases

### AC-AUTH-01 — Valid login and cookie

An active provisioned user can log in. The session cookie is opaque, `HttpOnly`, `Secure` in TLS environment, has the approved `SameSite` and path policy, and is absent from local storage and URLs.

### AC-AUTH-02 — Generic invalid login

Unknown email, wrong password, disabled user, and disabled membership return the same user-facing authentication failure shape. No case reveals whether an account exists.

### AC-AUTH-03 — Session lifecycle

Logout revokes the current session. Expired, revoked, password-policy-invalidated, or administrator-revoked sessions cannot access protected endpoints. Session fixation test confirms a new token after login.

### AC-AUTH-04 — CSRF

Every unsafe cookie-authenticated REST endpoint rejects absent/invalid CSRF token and disallowed Origin. Valid same-origin requests succeed. Webhooks do not depend on browser CSRF tokens.

### AC-AUTH-05 — Permission matrix

- Read-only analyst cannot confirm/cancel orders, create/retry shipments/messages, or modify integrations/users/roles.
- Order operator can confirm but cannot manage credentials/roles.
- Fulfilment operator can create/retry shipments according to role but cannot manage users.
- Administrator can perform configured administrative actions.
- Direct API calls are denied identically to hidden/disabled UI actions.

### AC-AUTH-06 — Cross-organization isolation

Automated tests using IDs from the second organization receive not-found/forbidden behavior and cannot read or mutate orders, customers, messages, shipments, integrations, jobs, users, roles, or audit logs across organizations.

### AC-AUTH-07 — Audit of sensitive actions

Login success, user/role changes, integration setting/secret rotation, order confirmation, shipment retry, message retry, and manual sync/reconciliation create append-only audit records with actor, target, time, outcome, and correlation ID but no secret/PII payload.

## 7. Shopify acceptance cases

### AC-SHP-01 — Connection health

Settings shows the correct non-secret shop identity, pinned API version, granted/required capability result, last health time, and safe failure if token/scope is invalid. It never returns the token or client secret.

### AC-SHP-02 — Initial synchronization

From an empty CRM, a resumable initial sync imports representative locations, products, variants, inventory levels, customers, orders, and required fulfilment references. Counts and progress are visible. Rerunning produces updates/no-ops, not duplicates.

### AC-SHP-03 — Duplicate order webhook

Deliver the identical valid order webhook at least three times and run multiple workers. One inbox identity and one CRM order business effect result; attempts/delivery metadata may show repeats without duplicating order, lines, customer source identity, or message intent.

### AC-SHP-04 — Out-of-order events

Deliver a newer order update followed by an older event. Current source facts do not regress. Both verified events remain diagnosable with processing result.

### AC-SHP-05 — Invalid webhook

Tampered body, invalid signature, unknown shop/account, oversized body, and unsupported method are rejected without business mutation or trusted inbox event. Logs contain safe metadata only.

### AC-SHP-06 — Reconciliation

Withhold a supported update webhook, change the source, then run scheduled/manual reconciliation. CRM converges to Shopify without duplicate records, and the sync run reports the correction.

### AC-SHP-07 — Inventory non-deduction

Record Shopify inventory after order creation/import. Confirm and ship in CRM. CRM performs no additional inventory adjustment call and its mirror changes only in response to Shopify source/reconciliation.

### AC-SHP-08 — Status separation

Changing Shopify financial/fulfilment status updates only the corresponding mirrored fields unless an explicitly approved rule proposes a valid CRM transition. Courier and CRM statuses remain separately visible.

### AC-SHP-09 — Fulfilment/tracking idempotency

A shipment tracking intent sent repeatedly due to duplicate outbox/job delivery produces the approved single Shopify fulfilment/tracking effect. Stored external fulfilment identity permits reconciliation.

### AC-SHP-10 — Fulfilment exceptions

Test insufficient scope, already fulfilled allocation, cancelled order, provider validation error, rate limit, and transient error. CRM shipment remains intact; operation is retryable or terminal according to classification and shows a safe action.

### AC-SHP-11 — Uninstall/privacy handling

Applicable uninstall and mandatory privacy/compliance webhook cases for the approved Shopify app model are verified, idempotent, auditable, and follow the approved retention/anonymization policy.

## 8. Orders, customers, catalog, and inventory cases

### AC-ORD-01 — Orders list

Search, operational/financial/fulfilment/shipment/date/sync filters, sorting, and pagination return organization-scoped correct results. URL/query state is shareable where safe. Loading, empty, stale, permission, and error states are usable.

### AC-ORD-02 — Order detail and snapshots

Order detail shows source identity, amounts/currency, customer and immutable order address snapshot, lines/SKUs/quantities, all independent statuses, message/shipment panels, freshness, and chronological timeline.

### AC-ORD-03 — Valid confirmation

Authorized confirmation from the allowed state with expected version records one transition, actor/time, audit entry, and shipment-readiness intent.

### AC-ORD-04 — Concurrent/repeated confirmation

Two simultaneous confirmations and a repeated idempotency key yield one transition/business effect. Same key with a different request fingerprint is rejected. A stale expected version returns a conflict without overwriting newer state.

### AC-ORD-05 — Invalid transition

Attempted transition outside the defined state machine is rejected and does not create an external side effect.

### AC-ORD-06 — Cancellation separation

CRM cancellation of an eligible order records the local transition but does not claim courier/Shopify cancellation unless a separate officially supported action succeeds. Source cancellation is displayed independently.

### AC-CUS-01 — Customer list/detail

Authorized user can search approved fields and view customer, addresses, related orders, and freshness. Unauthorized roles cannot retrieve restricted detail through direct API calls.

### AC-CAT-01 — Product/variant mirror

Product and variant values match Shopify and are read-only. SKU search supports non-unique SKUs without conflating variants.

### AC-INV-01 — Inventory by location

Inventory shows each representative variant at both Shopify locations with exact quantity and sync time. Stale data is marked. There is no inventory mutation action.

## 9. WhatsApp acceptance cases

### AC-WA-01 — Configuration secrecy and health

Settings shows non-secret account/phone identity, configured template mapping, credential presence/expiry metadata if safe, webhook health, and connection result without returning access token, app secret, or verify token.

### AC-WA-02 — Eligible confirmation message

An eligible new order creates exactly one message intent using the configured template/locale/parameters. Accepted send stores provider message ID and submitted time.

### AC-WA-03 — Duplicate trigger

Replaying order processing, outbox publishing, or queue delivery does not create another CRM message for the same purpose/version. Any provider-level ambiguity is surfaced honestly rather than labelled exactly once.

### AC-WA-04 — Status lifecycle

Valid signed provider events for sent, delivered, read, and failed update the message projection and append status events. Duplicate events do not duplicate effects; a late lower-precedence event does not regress the projection.

### AC-WA-05 — Invalid webhook

Invalid challenge/signature, tampered body, unknown phone/account, and oversized request are rejected with no message mutation.

### AC-WA-06 — Non-blocking ineligible order

Missing/invalid phone, missing consent under approved policy, unsupported locale, or missing/unapproved template causes `SKIPPED`/visible terminal failure while the Shopify order remains imported and operable.

### AC-WA-07 — Provider failures

Rate-limit/transient failures retry with bounded backoff; provider validation/authentication failures stop and expose safe remediation. Manual retry requires permission and creates an audit record without creating a new logical message intent.

### AC-WA-08 — Inbound scope boundary

An inbound customer conversation event does not create an AI response, support inbox, or free-form outbound message. It follows the approved acknowledge/minimize/ignore behavior.

## 10. Courier and shipment acceptance cases

### AC-SHPmt-01 — Confirmation prerequisite

API and UI reject shipment creation for unconfirmed, cancelled, or invalid orders. No outbox/provider operation is emitted.

### AC-SHPmt-02 — Readiness validation

Missing recipient/address/phone, empty shippable lines, invalid COD/currency policy, or unhealthy courier blocks readiness with actionable safe reasons.

### AC-SHPmt-03 — Duplicate prevention

Repeated button clicks, identical API requests, duplicate outbox delivery, worker retry, and concurrent workers result in one active primary shipment operation and one provider shipment for a provider with deterministic idempotency support.

### AC-SHPmt-04 — Successful creation

Successful response stores provider key, shipment ID, tracking number/URL, raw/normalized status, client reference, items, event, operation success, audit/system timeline, and Shopify fulfilment outbox.

### AC-SHPmt-05 — Retryable and terminal failure

Scripted retryable failure schedules bounded retry with attempts/next time visible. Terminal validation/auth failure stops and shows safe remediation. Neither creates a second logical shipment.

### AC-SHPmt-06 — Ambiguous creation

Simulate connection loss after provider acceptance:

- when provider supports client-reference lookup/idempotency, reconciliation finds the existing shipment before resend;
- when capability is absent/unknown, state becomes `RECONCILIATION_REQUIRED`, automatic create retries stop, and operator sees a clear warning.

### AC-SHPmt-07 — Status progression

Approved normalized events move shipment/order through shipped, out for delivery, and delivered as allowed. Raw and normalized statuses are both visible.

### AC-SHPmt-08 — Failure/refusal/return

Representative normalized delivery failed, refused, and returned events produce only approved CRM transitions and remain distinct. Required manual decision points are visible.

### AC-SHPmt-09 — Unknown/regressive status

Unknown or older/regressive event is appended, flagged, and does not silently regress a terminal/current state.

### AC-SHPmt-10 — Fake provider labelling

When fake provider is active, settings, shipment pages, tracking values, logs/evidence, and release banner identify “Simulated courier — not Ozone.” Fake controls are unavailable when production policy forbids them.

## 11. Integration operations, jobs, and audit cases

### AC-OPS-01 — Transactional outbox recovery

Terminate the API immediately after committing a domain change/outbox event but before queue publish. After restart, publisher dispatches the event and one business effect results.

### AC-OPS-02 — Redis loss/recovery

Restart or clear the test Redis queue after intent is committed. PostgreSQL outbox/operation state enables safe republish/reconciliation; committed work does not disappear.

### AC-OPS-03 — Worker lease recovery

Terminate a worker after claiming work. After lease expiry/recovery, another worker completes or safely reconciles it without invalid duplicate effect.

### AC-OPS-04 — Failed jobs screen

Authorized user can filter retryable, terminal, and reconciliation-required operations, inspect safe error/attempt/correlation/provider request metadata, and perform only eligible permitted retries.

### AC-OPS-05 — Integration events screen

Authorized user can inspect provider, event type, received/source time, dedupe identity, processing state, attempts, and sanitized failure. Raw secrets/PII are absent or restricted/encrypted under the approved diagnostic policy.

### AC-OPS-06 — Audit immutability

No application API allows update/delete of audit records. Filters by actor/action/target/date work with bounded pagination.

## 12. Screen acceptance matrix

| Screen | Acceptance |
|---|---|
| Login | AC-AUTH-01 through AC-AUTH-04; keyboard accessible and generic errors |
| Dashboard | Correct linked counts, organization time zone/range, freshness and failure indicators |
| Orders list | AC-ORD-01 |
| Order details/timeline | AC-ORD-02 through AC-ORD-06 and visible message/shipment operations |
| Customers | AC-CUS-01 |
| Products/variants | AC-CAT-01 |
| Inventory by location | AC-INV-01 |
| Shipments/tracking | AC-SHPmt-01 through AC-SHPmt-10 as applicable |
| Shopify settings/health | AC-SHP-01, sync runs, webhook freshness, safe errors |
| WhatsApp settings/status | AC-WA-01 through AC-WA-08 as applicable |
| Ozone settings | Shows blocked readiness with missing official inputs, or passed live connection; never invented values |
| Integration events | AC-OPS-05 |
| Failed jobs | AC-OPS-04 |
| Users/roles | Permission-gated create/update/disable/assignment with audit |
| Audit logs | AC-AUTH-07 and AC-OPS-06 |

Each screen passes responsive small-laptop layout, keyboard navigation, visible focus, labelled controls, contrast, error association, loading, empty, stale, permission-denied, and recoverable-error checks.

## 13. Non-functional acceptance

### AC-NFR-01 — Performance

At the reference dataset of 100,000 orders, 100,000 customers, 50,000 variants, and 1,000,000 events:

- p95 ordinary API reads are below 500 ms;
- p95 ordinary local mutations are below 1 second, excluding external completion;
- list/timeline endpoints remain bounded and paginated;
- critical SQL plans use intended indexes and show no unbounded N+1 queries.

Test hardware and load profile are recorded.

### AC-NFR-02 — Webhook response

Valid webhook acceptance responds within 5 seconds after verification and durable inbox write under expected load. Slow domain/provider processing is deferred.

### AC-NFR-03 — Process recovery

Restart web, API, and worker independently. Readiness changes correctly, in-flight durable work recovers, and no committed order/outbox/inbox record is lost.

### AC-NFR-04 — Backup and restore

Restore encrypted backup to an isolated environment within the target 4-hour RTO. Data meets the 24-hour RPO, migrations/app start, integrity checks pass, and pending operations are reconciled before egress is enabled.

### AC-NFR-05 — Security

The release checklist in `SECURITY.md` passes, including Argon2id benchmark, session/CSRF, permission/isolation, webhook signatures, security headers, private data stores, non-root containers, scans, and log sentinel tests.

### AC-NFR-06 — Secret/PII leakage

Automated sentinel values placed in credentials and customer fields do not appear in:

- browser JavaScript bundles or HTML;
- ordinary API error bodies;
- structured application logs;
- audit metadata;
- BullMQ job inspection available to normal operators.

### AC-NFR-07 — Accessibility/browser

Core golden path completes using keyboard alone with no serious automated accessibility violation and passes smoke tests on the agreed current/previous major browser versions.

### AC-NFR-08 — Observability

An operator can trace the golden-path order across request, webhook, integration event, outbox, queue operation, message, shipment, Shopify fulfilment, and audit records using correlation/entity IDs without inspecting credentials.

### AC-NFR-09 — Time and money

Stored instants are UTC and display correctly in organization time zone around a tested date boundary. All amounts round-trip as integer minor units with correct currency; no floating-point discrepancy appears.

## 14. Live Ozone acceptance addendum

These cases are blocked until official documentation and sandbox credentials are approved.

### AC-OZONE-01 — Contract traceability

Every implemented endpoint, header/auth mechanism, request field, response field, error, capability, and status mapping references the approved official documentation version. No fake fixture is used as evidence.

### AC-OZONE-02 — Authentication and health

Sandbox authentication succeeds; invalid/expired credentials produce safe terminal configuration state. Credentials are server-only and redacted.

### AC-OZONE-03 — Shipment creation

Representative valid order creates one sandbox shipment with correct approved address, parcel, COD, reference, and returned tracking fields.

### AC-OZONE-04 — Duplicate and ambiguous-result safety

Repeated clicks/jobs and simulated timeout follow Ozone's official idempotency/client-reference/lookup semantics and do not create an unintended second shipment. If the official API cannot guarantee reconciliation, documented manual control and business sign-off are required.

### AC-OZONE-05 — Validation/errors/rate limits

Official invalid-address/field, authentication, throttling, transient, and unavailable cases map to correct retry/terminal behavior with safe visible errors.

### AC-OZONE-06 — Status acquisition

Official webhook signature/retry behavior or polling behavior is tested. Duplicate and out-of-order events remain idempotent.

### AC-OZONE-07 — Status mapping

Every official status maps through an approved table to raw, normalized, terminal/review flags, and permitted CRM transition. Failure, refusal, cancellation, and return distinctions are verified with sandbox fixtures or Ozone-approved evidence.

### AC-OZONE-08 — Shopify update

Tracking/status obtained from the real Ozone sandbox drives the approved Shopify fulfilment/tracking flow once without conflating courier and CRM states.

### AC-OZONE-09 — Cutover labelling

Ozone-connected shipments show provider `OZONE`; fake shipments remain `FAKE_COURIER`. Historical fake tracking is never presented as live Ozone tracking.

Only after AC-OZONE-01 through AC-OZONE-09 pass may the release carry the **Core MVP — live Ozone** label.

## 15. Explicit rejection conditions

Release is rejected if any of the following occurs:

- duplicate CRM order for one Shopify source order/connection;
- duplicate courier shipment caused by an ordinary repeated click/job retry;
- CRM inventory deduction in addition to Shopify's adjustment;
- one status field conflates Shopify financial, Shopify fulfilment, CRM operational, or courier state;
- shipment creation before employee confirmation;
- unverified webhook mutates business data;
- credential or sensitive customer data appears in frontend code/logs;
- failed external operations disappear without durable visible state;
- fake courier is represented as live Ozone;
- live Ozone behavior is implemented/tested from guessed endpoints, fields, auth, or statuses;
- role or organization scope can be bypassed through direct API access;
- backup cannot be restored;
- required privacy/retention policy remains unapproved for production handling.

## 16. Traceability summary

| Requirement area | Primary acceptance cases |
|---|---|
| Authentication/RBAC/organizations | AC-AUTH-01–07 |
| Dashboard and operational UI | Screen matrix, AC-ORD-01, AC-NFR-08 |
| Shopify import/webhooks/inventory | AC-SHP-01–08, AC-INV-01 |
| Orders/customers/catalog | AC-ORD-01–06, AC-CUS-01, AC-CAT-01 |
| WhatsApp templates/status | AC-WA-01–08 |
| Shipment/courier | AC-SHPmt-01–10 |
| Shopify fulfilment outbound | AC-SHP-09–10 |
| Outbox/idempotency/jobs/audit | AC-OPS-01–06 |
| Security/privacy/reliability | AC-NFR-02–09 |
| Live Ozone | AC-OZONE-01–09 |

## 17. Sign-off record

Record the following in the release evidence:

```text
Release label:
Release version / image digests:
Environment:
Database migration version:
Shopify shop and API version:
WhatsApp test account/template:
Courier provider (FAKE_COURIER or OZONE):
Acceptance run link/location:
Known accepted defects:
Product owner:
Operations representative:
Technical lead:
Deployment owner:
Date:
```
