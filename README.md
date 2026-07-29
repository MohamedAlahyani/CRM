# Commerce Operations CRM

Commerce Operations CRM is a self-hosted operational CRM for e-commerce teams. It connects Shopify orders, customers, products, and inventory with employee confirmation workflows, WhatsApp Business Cloud API messaging, and courier shipment tracking.

The product is designed first for a single client installation with a one-time license. The architecture keeps organization scoping in the data model so future multi-tenant SaaS work remains possible, but SaaS billing, tenant self-service, and shared-infrastructure tenancy are not part of the MVP.

## Current State

This repository currently contains product and architecture documentation only. No application code, database schema, frontend, backend, worker, or live integration implementation has been created yet.

External baseline:

- A Shopify development app named `CRM Connector` exists outside this repository.
- The app is non-embedded, published, and installed on a Shopify test store.
- Shopify Client ID, Client Secret, and `.myshopify.com` domain are expected later through server-side environment configuration.
- Shopify Admin API authorization, WhatsApp credentials, and Ozone Express official API details are not yet present.

## MVP Scope

The MVP focuses on one end-to-end commerce operations workflow:

1. Shopify sends an order webhook.
2. The CRM ingests the order once, idempotently.
3. Employees review the order, customer, items, and inventory context.
4. The CRM sends an approved WhatsApp confirmation template.
5. An employee manually confirms the order.
6. The CRM creates a courier shipment only after confirmation.
7. Courier status and tracking are shown in the CRM.
8. Valid shipment tracking is pushed back to Shopify fulfilment.

MVP providers:

- Commerce: Shopify only.
- Messaging: official WhatsApp Business Cloud API only.
- Courier: Ozone Express only when official documentation and test credentials are supplied.
- Local/client workflow testing: a clearly labelled `FakeCourierProvider` until live Ozone integration is authorized.

Future providers such as YouCan, WooCommerce, Vitex, and Ameex are represented through provider interfaces, but are not implemented in the MVP.

## Core Product Rules

- Shopify remains the source of truth for storefront, checkout, order facts, financial status, fulfilment status, products, variants, locations, and tracked inventory.
- The CRM owns operational workflow, employee decisions, messaging orchestration, courier coordination, synchronization visibility, and audit history.
- The CRM must not double-deduct stock.
- Courier status is stored separately from CRM operational status.
- Webhooks, jobs, provider calls, and shipment creation must be idempotent.
- Ozone shipment creation is allowed only after an employee confirms the order.
- Provider failures must be visible and retryable.
- Credentials and PII must never be exposed to the frontend or written to logs.

## CRM Order State Machine

The operational order lifecycle is:

```text
NEW
WAITING_CONFIRMATION
CONFIRMED
READY_FOR_SHIPMENT
SHIPPED
OUT_FOR_DELIVERY
DELIVERED
DELIVERY_FAILED
REFUSED
RETURNED
CANCELLED
```

Financial, fulfilment, courier, and WhatsApp message statuses are modeled separately so each external system can remain truthful without forcing incorrect CRM transitions.

## Proposed Architecture

The planned implementation is a TypeScript modular monolith with three process types:

- `web`: Next.js App Router frontend.
- `api`: NestJS REST API, auth, webhooks, domain use cases, and OpenAPI.
- `worker`: NestJS worker for BullMQ jobs, outbox dispatch, sync, reconciliation, and provider calls.

Preferred stack:

- Node.js 22
- pnpm and Turborepo
- Next.js App Router
- NestJS
- PostgreSQL with Prisma
- Redis with BullMQ
- Tailwind CSS and shadcn/ui
- React Hook Form, Zod, TanStack Query
- Argon2id password hashing
- Opaque secure HTTP-only sessions
- Pino logging
- Vitest/Jest and Playwright
- Docker Compose and container images

This is intentionally not a microservice architecture.

## Documentation

The detailed project baseline lives in `docs/`:

- [Product Requirements](docs/PRD.md)
- [Architecture](docs/ARCHITECTURE.md)
- [Database Design](docs/DATABASE_DESIGN.md)
- [Integrations](docs/INTEGRATIONS.md)
- [Security](docs/SECURITY.md)
- [Implementation Plan](docs/IMPLEMENTATION_PLAN.md)
- [MVP Acceptance Criteria](docs/MVP_ACCEPTANCE.md)

Recommended reading order:

1. `docs/PRD.md`
2. `docs/ARCHITECTURE.md`
3. `docs/INTEGRATIONS.md`
4. `docs/DATABASE_DESIGN.md`
5. `docs/SECURITY.md`
6. `docs/IMPLEMENTATION_PLAN.md`
7. `docs/MVP_ACCEPTANCE.md`

## First Implementation Milestone

The first coding milestone should build only the executable secure foundation:

- pnpm/Turborepo workspace.
- `web`, `api`, and `worker` process shells.
- PostgreSQL and Redis development services.
- Validated configuration.
- First Prisma migration for organization, users, roles, sessions, audit, outbox, and idempotency.
- Secure administrator bootstrap.
- Login, logout, session, CSRF, RBAC, and organization isolation.
- Health checks.
- Baseline tests and CI tasks.

Shopify, WhatsApp, Ozone, orders, products, inventory, customers, shipments, and production UI design are explicitly excluded from the first milestone.

## External Requirements Still Needed

Live integration work is blocked until the required external inputs are supplied:

- Shopify: Admin API authorization/token lifecycle, pinned API version, scopes, webhook topic matrix, privacy webhook requirements, fulfilment ownership policy, and inventory behavior confirmation.
- WhatsApp: Meta app/business identifiers, token strategy, phone number ID, webhook verification token, approved templates/locales/variables, consent policy, and test recipients.
- Ozone Express: official versioned documentation, authentication model, shipment schema, error model, status catalogue, idempotency/reference behavior, webhook or polling mechanism, sandbox/test credentials, and rate limits.
- Operations/privacy: host/domain/TLS plan, retention and deletion policy, backup policy, monitoring and incident response expectations, password recovery policy, and deployment environment decisions.

## Important Ozone Rule

Do not invent Ozone Express endpoints, authentication, payloads, statuses, or behavior. Until official Ozone documentation and test credentials are available, the only allowed courier implementation is a visibly simulated fake provider used for workflow testing.

