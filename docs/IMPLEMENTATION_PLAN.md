# Commerce Operations CRM — Implementation Plan

## 1. Delivery strategy

Build one modular monolith in vertical increments. Establish correctness, security, and operational visibility before live provider side effects. Each milestone must leave the repository runnable and tested; provider work advances only after its readiness gate.

No YouCan, WooCommerce, Vitex, Ameex, AI agent, SaaS billing, or multi-tenant product workflow is scheduled in this MVP plan.

## 2. Workstreams

- Product decisions and external readiness.
- Repository/tooling and development environment.
- Database/domain platform.
- Authentication, authorization, and audit.
- Shopify inbound synchronization.
- CRM operational UI and workflow.
- WhatsApp outbound messaging.
- Courier abstraction and fake workflow.
- Shopify outbound fulfilment.
- Ozone live adapter, conditionally.
- Reliability, security, performance, and deployment acceptance.

## 3. Pre-coding decision gate

Coding can begin on platform foundations while external integration inputs are pending, but these decisions must be recorded:

- Node.js 22 and pnpm versions/pinning mechanism.
- Target operating system/container runtime for production.
- Organization name, time zone, default locale, currency expectations, and default phone country.
- Initial administrator bootstrap procedure.
- MVP policy: one primary shipment per order and no partial/split shipment.
- Whether the new-order WhatsApp confirmation template is automatically triggered.
- Fake courier allowed for first client workflow demonstration.

Integration-specific gates in `INTEGRATIONS.md` apply before each live adapter stage.

## 4. Exact first coding milestone

### Milestone 1 — Executable secure foundation

**Objective:** create a reproducible repository that proves the three-process modular-monolith shape, database migration path, authenticated organization-scoped API, durable outbox plumbing, and automated test baseline without implementing commerce features.

**Deliverables:**

1. Initialize pnpm workspace and Turborepo with pinned Node.js 22 and package manager versions.
2. Create:
   - `apps/web` with Next.js App Router, Tailwind, shadcn/ui foundation, TanStack Query provider, and a minimal authenticated shell;
   - `apps/api` with NestJS, versioned REST prefix, OpenAPI, validation, cookie/CSRF middleware, Pino, and liveness/readiness endpoints;
   - `apps/worker` with NestJS application context, BullMQ connection, graceful shutdown, and one internal smoke-test queue;
   - `packages/core`, `database`, `contracts`, `integrations`, `config`, `eslint-config`, and `typescript-config`.
3. Add Docker Compose services for PostgreSQL and Redis plus development commands for web/API/worker.
4. Add validated configuration with startup failure for missing required values and explicit dev/test defaults that cannot be used in production.
5. Implement the first Prisma migration for:
   - organization;
   - user, membership, role, permission, join tables;
   - session;
   - audit log;
   - outbox event;
   - idempotency record.
6. Add secure bootstrap command for one organization and administrator plus default roles/permissions.
7. Implement login, logout, session resolution/revocation, `GET /api/v1/auth/me`, Argon2id hashing, secure cookie policy, CSRF protection, and sign-in throttling.
8. Implement mandatory organization context and permission guard with a second-organization isolation test fixture.
9. Implement transactional audit/outbox repository primitives and an outbox publisher that creates a stable BullMQ smoke job.
10. Implement health checks:
    - liveness: process event loop/responding;
    - readiness: validated configuration, PostgreSQL, and Redis/queue connectivity.
11. Add CI tasks for format, lint, typecheck, unit tests, integration tests, build, Prisma migration validation, dependency audit, secret scan, and container build when Docker CI is available.
12. Add baseline Vitest unit/integration tests and Playwright browser tests:
    - bootstrap and login/logout;
    - invalid/disabled credentials;
    - secure session revocation/expiry;
    - CSRF rejection;
    - permission denial;
    - cross-organization access denial;
    - atomic domain/audit/outbox transaction;
    - outbox duplicate publish/consume safety;
    - process health and authenticated shell.

**Explicit exclusions:** Shopify, WhatsApp, Ozone, orders, products, inventory, customers, shipments, and production visual design are not implemented in this milestone.

**Exit criteria:**

- A new developer can start PostgreSQL/Redis and all three processes using documented commands.
- Clean database migration plus bootstrap succeeds.
- Administrator can log in and load the authenticated shell.
- A protected test endpoint enforces permission and organization scope.
- Committed test outbox event is consumed once in business effect despite duplicate delivery.
- Restarting API/worker does not lose the committed outbox event.
- CI is green and no secret appears in client bundles/log sentinel test.

This is the first coding milestone. Work must not start by building dashboard screens or provider adapters before this foundation passes.

## 5. Subsequent milestones

### Milestone 2 — Commerce data model and operator read shell

**Objective:** establish domain schema, read APIs, status separation, and production screen structure using fixtures.

Deliverables:

- Prisma entities/constraints/indexes for connections, external maps, sync runs, integration events, customers/addresses, catalog, inventory, orders/lines/address snapshots/transitions, operations.
- Order operational state machine with table-driven transition tests.
- Organization-scoped REST modules and OpenAPI for orders, customers, products, inventory, integration events, sync runs, audit.
- Dashboard/order/customer/catalog/inventory/settings page shells with loading/empty/error/permission states.
- Cursor pagination for timelines/events and bounded list endpoints.
- Seeded fixture data and performance-oriented query tests.

Exit:

- Independent CRM/financial/fulfilment/courier statuses are visible and cannot overwrite one another.
- Duplicate external mapping/order constraints pass concurrency tests.
- Lists do not make unbounded/N+1 queries at reference volume.

### Milestone 3 — Shopify connection and inbound sync

**Entry gate:** approved Shopify auth/install method, token handling, pinned API version, scope matrix, shop domain, test-store access, and webhook topic matrix.

Deliverables:

- `CommerceProvider` contract tests.
- Shopify adapter health/capability check.
- Secure connection configuration and credential presence/rotation UI.
- Resumable initial sync in dependency order.
- Verified durable webhook inbox and idempotent processors.
- Scheduled reconciliation and stale-event protection.
- Sync health, progress, failures, retry, and audit visibility.
- Shopify privacy/uninstall event handling required by approved app model.

Exit:

- Products, variants, locations, inventory, customers, and orders import from test store.
- Repeated/out-of-order webhook tests do not duplicate or regress data.
- CRM never decrements Shopify inventory as an order side effect.
- Dropped-webhook scenario is repaired by reconciliation.

### Milestone 4 — Operational order workflow

Deliverables:

- Complete orders list/detail/timeline and customer details.
- Search/filter/sort/pagination.
- Authorized, idempotent order confirmation with optimistic concurrency.
- Readiness validation and visible blocking reasons.
- Dashboard operational counts and freshness.
- Audit for all user transitions.

Exit:

- Two simultaneous confirmation attempts yield one transition.
- Invalid transitions and stale versions are rejected.
- Order source updates remain separate from CRM state.

### Milestone 5 — WhatsApp outbound templates

**Entry gate:** Meta credentials, public callback, approved templates/locales/variables, phone normalization, consent policy, and test recipients.

Deliverables:

- `MessagingProvider` contract tests and WhatsApp Cloud adapter.
- Template mapping settings with secret-safe health.
- Idempotent confirmation message intent and sending.
- Verified webhook status ingestion.
- Message panel/list, retries, failure reasons, and audit/operations.
- Rate-limit and timeout tests.

Exit:

- Eligible order creates one message intent and accepted message gets provider ID.
- Duplicate trigger/webhook does not duplicate business effect.
- Sent/delivered/read/failed lifecycle is shown.
- Invalid/missing consent/configuration does not block order import.

### Milestone 6 — Courier orchestration with fake provider

Deliverables:

- Shipment schema/module/UI and `CourierProvider` contract tests.
- Readiness checks after confirmation.
- `FakeCourierProvider` with deterministic dedupe and scripted scenarios.
- Shipment creation operation, tracking, status timeline, normalized transition mapping.
- Ambiguous-result/reconciliation-required handling.
- Persistent simulated-provider labelling and production policy guard.

Exit:

- Repeated click/job retry creates one fake provider shipment.
- Status progress updates shipment/order correctly.
- Unknown/regressive statuses are retained and flagged.
- Failure/retry/reconciliation scenarios are operable from CRM.

### Milestone 7 — Shopify fulfilment/tracking outbound

**Entry gate:** fulfilment scopes, ownership/allocation, notification policy, and representative test orders approved.

Deliverables:

- Fulfilment-order reference synchronization.
- Outbox-driven tracking/fulfilment adapter operation.
- Idempotency/reconciliation and safe error handling.
- UI status and retry visibility.
- Shopify confirmation tests on COD, cancelled, already fulfilled, and representative fulfilment-order cases.

Exit:

- Created shipment tracking appears in Shopify once.
- Retry does not produce an unintended second fulfilment.
- Permission/user errors are visible and do not corrupt CRM shipment state.

### Milestone 8A — Client-testable MVP using fake courier

Use this path if Ozone remains blocked.

Deliverables:

- End-to-end acceptance suite across Shopify, WhatsApp, fake courier, and Shopify fulfilment.
- All requested MVP screens complete.
- Deployment/runbooks, backup/restore, monitoring, security/performance validation.
- Explicit release label: “Courier is simulated; live Ozone is not included.”

Exit is defined in `MVP_ACCEPTANCE.md`. Passing this milestone validates CRM workflow, not live Ozone integration.

### Milestone 8B — Live Ozone adapter

**Entry gate:** every item in the Ozone readiness gate is received and approved.

Deliverables:

- Official field/auth/error/status mapping document.
- Ozone adapter and contract/integration tests against sandbox.
- Webhook verification or polling implementation exactly per official contract.
- Provider idempotency/reconciliation strategy.
- Sandbox scenarios for creation, duplicate prevention, tracking, failure, refusal, return, cancellation, rate limiting, and ambiguous timeouts.
- Operations/product sign-off and configuration switch from fake to Ozone.

Exit:

- Live-Ozone acceptance addendum in `MVP_ACCEPTANCE.md` passes.
- Fake provider data cannot be mistaken for Ozone data.

## 6. Test strategy

### Unit tests

- State transitions and invariants.
- Normalization/mapping.
- Permission policy.
- Retry/error classification.
- Idempotency key derivation.
- PII/log redaction.

### Integration tests

- Prisma constraints and transaction rollback.
- Outbox/inbox concurrency and lease recovery.
- Redis/BullMQ duplicate delivery.
- API auth/CSRF/organization scope.
- Provider adapters against official mocked fixtures; no invented Ozone fixtures.

### Contract tests

Each provider adapter runs a shared suite for advertised capabilities. Capability-specific cases are required only when advertised. Fake provider has its own explicit simulated scenarios.

### End-to-end tests

Playwright covers role-aware web workflows. Provider sandbox tests are separate, tagged, and never run with production credentials. Webhook fixtures are signed in test using test-only secrets.

### Resilience tests

- Process termination after domain commit but before queue publish.
- Duplicate delivery and concurrent workers.
- Redis restart and outbox replay.
- Provider timeouts/rate limits/5xx.
- Out-of-order webhooks.
- Database connection loss and recovery.
- Ambiguous shipment creation.

### Performance tests

Generate the reference dataset from `PRD.md`; measure p95 reads/mutations, sync throughput, webhook acceptance, queue lag, and critical SQL plans.

## 7. Definition of done for every milestone

- Requirements and decision records updated.
- Migrations reviewed with rollback/roll-forward notes.
- Unit/integration/e2e tests appropriate to the change pass.
- OpenAPI and frontend client remain synchronized.
- Organization scope and permission checks are present.
- Audit/outbox effects are atomic where required.
- Logs/errors pass secret and PII review.
- Loading/empty/error/permission/accessibility states are implemented for affected UI.
- Monitoring and operational recovery path exist.
- No provider behavior is asserted without official documentation or an explicitly fake adapter.
- User-visible terminology distinguishes source, CRM, and courier statuses.

## 8. Dependencies

### Internal/technical

- Node.js 22 LTS, pnpm, Turborepo.
- Next.js App Router, React, Tailwind CSS, shadcn/ui.
- NestJS, OpenAPI tooling, Pino.
- PostgreSQL, Prisma.
- Redis, BullMQ.
- React Hook Form, Zod, TanStack Query.
- Argon2id implementation.
- Vitest for unit/integration tests across packages and backend; Playwright for browser E2E.
- Docker Compose and production image build tooling.

Versions are pinned at coding time after compatibility/security review; this documentation does not claim current package versions.

### External/business

- Shopify app ownership/configuration, store access, approved scopes/API version/auth lifecycle.
- Meta app/business verification, WhatsApp credentials, templates, consent, webhook URL.
- Ozone official documentation, sandbox, credentials, support contact.
- Hosting, DNS/TLS, backups, monitoring, incident ownership.
- Product decisions and representative order/fulfilment/address/COD cases.

## 9. Risk register

| ID | Risk | Probability / impact | Mitigation | Owner/gate |
|---|---|---|---|---|
| R-01 | Shopify app lacks token/scopes | High / High | Resolve OAuth/token and scope matrix before M3 | Client + integration lead |
| R-02 | Shopify fulfilment ownership unsupported | Medium / High | Test representative fulfilment orders before M7 | Integration lead |
| R-03 | Ozone contract unavailable | High / High for live courier | Use clearly fake adapter; do not claim live completion | Client |
| R-04 | Ozone lacks idempotency/reference lookup | Unknown / Critical | Reconcile or stop on ambiguous result; operational approval | Architecture + Ozone |
| R-05 | WhatsApp templates/consent delayed | High / Medium | Gate M5, preserve order workflow, expose skipped/blocked reason | Client |
| R-06 | Webhook loss/order | Medium / High | Durable inbox, source timestamps, scheduled reconciliation | Backend |
| R-07 | Duplicate side effects | Medium / Critical | Constraints, operation keys, outbox, contract tests | Backend |
| R-08 | PII/secret leakage in logs | Medium / High | Minimize, redaction middleware, sentinel tests | Security |
| R-09 | Initial sync rate/volume | Medium / Medium | Cursor, throttle, checkpoint, high-water reconciliation | Backend |
| R-10 | Tenant scope omissions | Medium / High future | Mandatory context/repository convention/cross-org tests | Architecture |
| R-11 | Single-host loss | Medium / High | Off-host encrypted backup and restore drill | Deployment owner |
| R-12 | Self-hosted upgrades break schema | Medium / High | Versioned images, controlled migration, compatibility/runbook | Release owner |
| R-13 | Status mapping conflates domains | Medium / High | Separate columns/enums, explicit mapping, UI labels/tests | Product + backend |
| R-14 | Scope expands into support inbox/AI/other providers | Medium / Medium | Enforce PRD exclusions and change control | Product owner |

## 10. Release and deployment plan

1. Build and scan immutable images.
2. Back up database and verify release prerequisites.
3. Run migration preflight and controlled migration task.
4. Deploy API/worker/web with versioned configuration.
5. Verify liveness/readiness and release version.
6. Run smoke tests without unintended live messages/shipments.
7. Enable inbound webhooks and workers in controlled order.
8. Observe queue/outbox/provider error metrics.
9. Run reconciliation and acceptance sample.
10. Record release, migration, image digests, configuration versions, and operator.

Rollback is allowed only if the previous application is schema-compatible. Otherwise execute the prepared roll-forward remediation.

## 11. Documentation/runbooks required before production

- Local development and test fixture guide.
- Environment configuration and secret rotation.
- Organization/admin bootstrap and recovery.
- Shopify install/reinstall/uninstall and resync.
- WhatsApp credential/template/webhook verification.
- Courier connection/fake-to-live cutover.
- Failed job and ambiguous operation reconciliation.
- Pause/resume each outbound integration independently.
- Backup, restore, and disaster recovery.
- Release, migration, rollback/roll-forward.
- Incident response and provider credential compromise.

## 12. Change control

Any request to add a provider, inbound conversation, AI, billing, multi-tenant product flow, catalog editing, inventory mutation, partial shipments, or advanced returns is a scope change. It requires:

- product requirement and acceptance update;
- architecture/data/security impact review;
- migration and provider-contract analysis;
- estimate and milestone approval.

Provider documentation changes also trigger adapter contract review before upgrading a pinned API version.
