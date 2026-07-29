# Commerce Operations CRM — Implementation Plan

## 1. Delivery strategy

Build a self-hosted, single-client, single-tenant TypeScript modular monolith in vertical increments. The deployable processes are a Next.js web app, a NestJS API, and a NestJS worker backed by PostgreSQL/Prisma, Redis/BullMQ, and a transactional outbox. This is not a microservice architecture.

The first coding task after this documentation correction is exactly:

```text
Implement Milestone 0 only.
```

No application code is created by this documentation run. YouCan, WooCommerce, Vitex, Ameex, AI-agent functionality, subscriptions, SaaS onboarding, and unrelated product breadth are out of scope.

## 2. Release levels

### Technical demonstration

```text
Live Shopify + Live WhatsApp + FakeCourierProvider
```

This proves the CRM workflow while Ozone access is unavailable. It must be labelled simulated everywhere and is not accepted as live Ozone integration.

### Client MVP

```text
Live Shopify + Live WhatsApp + Live Ozone Express
```

Client-MVP acceptance requires official Ozone documentation, sandbox or test credentials, shipment creation, tracking, and status synchronization to pass. If Ozone access is unavailable, stop at the clearly labelled technical demonstration.

## 3. Milestones

### Milestone 0 — Executable foundation

- Create the monorepo and separate web, API, and worker applications.
- Add PostgreSQL and Redis development services.
- Add the Prisma schema and first migration.
- Add environment validation and safe development/test defaults.
- Add administrator bootstrap.
- Add `ADMIN` and `OPERATOR` authorization only.
- Add secure login, logout, and server-side sessions.
- Add liveness/readiness health endpoints.
- Add structured masked logging and correlation IDs.
- Add base format, lint, typecheck, unit, integration, and build test configuration.
- Add the minimal authenticated dashboard shell.

Milestone 0 does not implement Shopify, WhatsApp, Ozone, orders, products, inventory, customers, shipments, or provider adapters.

### Milestone 1 — Shopify vertical slice

- Implement client-credentials token acquisition for the same-organization development store.
- Cache the token server-side and renew it automatically before or after its 24-hour expiry.
- Use `SHOPIFY_API_VERSION=2026-07` and test connection health.
- Import products, variants, locations, inventory, customers, and orders with idempotent upserts.
- Add Shopify order and inventory webhook ingestion, durable inbox processing, background jobs, and reconciliation.
- Add order-list and order-details APIs.
- Record integration events and sanitized failures.
- Treat Shopify as the authority for Shopify-originated inventory; never subtract an imported order again in CRM.

OAuth authorization-code onboarding for unrelated merchant organizations is post-MVP.

### Milestone 2 — Operational dashboard

- Add orders list, order details, and order timeline.
- Add read-only catalog and inventory views.
- Add manual order confirmation and critical-action audit records.
- Add integration-event visibility and failed-job visibility/retry.
- Add basic user management for the fixed `ADMIN`/`OPERATOR` roles.

Dedicated customer-management screens, editable permissions, additional roles, advanced analytics, and dashboard builders remain deferred.

### Milestone 3 — WhatsApp

- Integrate the official Meta WhatsApp Cloud API.
- Configure an approved order-confirmation template through server-side environment configuration.
- Create one idempotent message intent per order confirmation purpose/version.
- Store provider message IDs and process the delivery-status webhook idempotently.
- Show message status and timeline entries.
- Permit manual authorized resend only for eligible failed messages.

### Milestone 4 — Ozone

- Analyze and approve the official Ozone documentation before implementing provider-specific behavior.
- Implement the `OzoneCourierProvider` skeleton first; do not invent endpoints, headers, fields, responses, signatures, or statuses.
- Once official access exists, implement authentication/health, shipment creation, duplicate prevention, tracking, courier-status synchronization, and Shopify fulfilment/tracking update.
- Keep `FakeCourierProvider` for the technical demonstration and label it permanently as simulated.

If official Ozone access is unavailable, stop at the technical demonstration.

### Milestone 5 — Client MVP verification

- Verify the complete Shopify-to-CRM-to-WhatsApp-to-confirmation-to-Ozone-to-Shopify flow.
- Run duplicate webhook replay, repeated confirmation, failed request/retry, token-expiration/renewal, duplicate WhatsApp send, and duplicate shipment-prevention tests.
- Produce production Docker configuration.
- Document and test the basic backup procedure.
- Complete the client UAT checklist and release sign-off.

## 4. Test strategy

MVP verification is bounded to the development-store dataset:

- initial import and rerun without duplicates;
- multiple test orders with different products and quantities;
- duplicate Shopify webhook replay;
- repeated order-confirmation attempts;
- failed external request with retry;
- token expiration and automatic renewal;
- duplicate WhatsApp-send protection;
- duplicate shipment-prevention and ambiguous-result handling;
- database constraints, transaction/outbox, and idempotency tests.

Enterprise-scale load benchmarks, 100,000-order tests, 1,000,000-event tests, and production-scale disaster simulations are post-MVP.

Every external adapter uses contract tests from official documentation or approved fixtures. Ozone tests remain blocked until official inputs exist.

## 5. Dependencies and gates

### Available or expected for Milestone 0

- Node/TypeScript toolchain, Docker, PostgreSQL, and Redis runtime.
- Bootstrap organization name, time zone, administrator email, and secure password procedure.

### Shopify gate

```env
SHOPIFY_SHOP_DOMAIN=
SHOPIFY_CLIENT_ID=
SHOPIFY_CLIENT_SECRET=
SHOPIFY_API_VERSION=2026-07
```

Authentication and API behavior follow Shopify's [client-credentials documentation](https://shopify.dev/docs/apps/build/authentication-authorization/access-tokens/client-credentials-grant) and [GraphQL Admin API `2026-07`](https://shopify.dev/docs/api/admin-graphql/2026-07).

Credentials and the access token remain server-side. The token is never logged or sent to the frontend. Required least-privilege scopes, webhook topics, fulfilment behavior, and test-store data must be confirmed.

### WhatsApp gate

Official Meta Cloud API credentials, phone/account identifiers, webhook verification settings, approved template and locale, consent policy, and test recipient are required.

### Ozone gate

Official documentation, base URL, authentication, request/response contract, status catalogue, webhook or polling behavior, idempotency/reference semantics, rate limits, sandbox/test credentials, and support contact are required. Missing items block live Ozone.

## 6. Definition of done

Every milestone has:

- passing automated tests appropriate to the milestone;
- validated environment configuration with no committed secrets;
- organization-scoped queries and API authorization;
- masked logs and correlation identifiers;
- durable failure state for external work;
- updated API/UI documentation and runbook notes;
- a recorded verification result and known limitations.

## 7. Explicit change control

Adding a provider, inbound WhatsApp support, AI, billing, multi-tenant onboarding, catalog editing, inventory mutation, partial shipments, advanced privacy workflows, or large-scale performance targets is a scope change. It requires a product decision, security/privacy review, estimate, and post-MVP milestone; it must not be smuggled into the current implementation sequence.
