# Commerce Operations CRM — Specification

This folder holds the product and architecture baseline for the Commerce Operations CRM.
For the project overview, architecture summary and setup instructions, see the
[root README](../README.md).

## Reading order

| # | Document | Contents |
| --- | --- | --- |
| 1 | [PRD.md](./PRD.md) | Vision, scope, roles, state machine, functional requirements |
| 2 | [ARCHITECTURE.md](./ARCHITECTURE.md) | System context, modules, outbox, webhooks, deployment |
| 3 | [INTEGRATIONS.md](./INTEGRATIONS.md) | Provider contracts, Shopify, WhatsApp, courier workflows |
| 4 | [DATABASE_DESIGN.md](./DATABASE_DESIGN.md) | Target tables, constraints, tenant safety, retention |
| 5 | [SECURITY.md](./SECURITY.md) | Threat model, auth, secrets, privacy, hardening |
| 6 | [IMPLEMENTATION_PLAN.md](./IMPLEMENTATION_PLAN.md) | Milestones, gates, definition of done |
| 7 | [MVP_ACCEPTANCE.md](./MVP_ACCEPTANCE.md) | Acceptance cases and sign-off |

## External inputs still needed

Live integration work is blocked until these are supplied.

| Area | Missing inputs |
| --- | --- |
| **Shopify** | Least-privilege scopes, webhook topic matrix, privacy webhooks, fulfilment ownership policy, inventory behaviour confirmation |
| **WhatsApp** | Meta app and business IDs, token strategy, phone number ID, webhook verify token, approved templates and locales, consent policy, test recipients |
| **Ozone Express** | Official versioned docs, authentication, shipment schema, error model, status catalogue, idempotency behaviour, webhook or polling, sandbox credentials, rate limits |
| **Operations and privacy** | Host, domain and TLS plan, retention and deletion policy, backup policy, monitoring, incident response, password recovery |

> **Ozone rule:** never invent Ozone Express endpoints, authentication, payloads, statuses
> or behaviour. Until official documentation and test credentials exist, the only allowed
> courier implementation is the visibly simulated `FakeCourierProvider`.
