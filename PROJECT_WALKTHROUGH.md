# How this CRM works

This guide explains the implemented code before an architecture redesign.
The source baseline is commit `ca842ee4bc8397d454958cb9e53059eb6f7fecd4`.
The review covers source code and configuration. It does not verify a running installation or configured credentials.

## 1. Start with the correct project

The runnable application lives in **`crm/`**. Run its package commands from that directory.

The outer `README.md` and `docs/ARCHITECTURE.md` describe a planned commerce system.
That plan includes Shopify orders, WhatsApp confirmation, courier shipping, and a BullMQ worker.
The implemented application is an AI-assisted sales CRM with different runtime components.

| Location | Responsibility |
| --- | --- |
| `crm/apps/app` | Next.js screens, server rendering, browser queries, authentication redirects, API and agent proxies |
| `crm/apps/api` | NestJS authentication hosting, tRPC procedures, business rules, database writes, mailbox sync |
| `crm/apps/agent` | eve runtime, research tools, task dispatch, record chat, agent builder and custom runs |
| `crm/packages/db` | Prisma schema, client, migrations, database helpers and settings |
| `crm/packages/auth` | Better Auth configuration, provider scopes, workspace membership and permission helpers |
| `crm/packages/validation` | Shared Zod schemas for data crossing component boundaries |
| `crm/packages/ui` | Shared UI components |
| `crm/packages/env` | Workspace environment-file discovery and loading |
| `crm/packages/telemetry` | Installation and product telemetry support |

**Architecture:** three applications share one database and several source packages.
They run separately, but their shared schema couples their behavior.

Sources: [workspace manifest](crm/package.json), [application modules](crm/apps/api/src/app.module.ts), [root project status](README.md).

## 2. What starts during development

The documented local ports are Next.js `3000`, NestJS `3001`, eve `2000`, and PostgreSQL `5432`.
Docker Compose starts PostgreSQL. It does not start the three applications.
Bun workspaces resolve local packages. Turborepo coordinates their development and build tasks.

`bun run dev` runs preparation tasks before application startup.
The database package prepares migrations and the generated Prisma client.
The API creates its NestJS modules and listens for requests.
Next.js serves pages and proxy routes. The agent starts its separate eve runtime.

The workspace root for environment loading is **`crm/`**, because its manifest declares workspaces.
The shared loader reads `.env`, then `.env.local`.
Within that loader, existing process variables take priority over file values.
Next.js and Bun also have runtime environment behavior. Inspect resolved targets before database operations.

Sources: [Turbo tasks](crm/turbo.json), [database scripts](crm/packages/db/package.json), [API entry](crm/apps/api/src/main.ts), [environment loader](crm/packages/env/src/index.ts), [local setup](crm/docs/setup.md).

## 3. Opening the application

1. The browser requests a Next.js page.
2. `proxy.ts` examines the route and session-cookie presence.
3. A visitor without a cookie goes to `/sign-in`, except on permitted public routes.
4. A visitor with a cookie passes through onboarding checks.
5. Server-side session helpers resolve the actual session where required.

Cookie presence and session validation serve different purposes.
The proxy uses cookie presence for routing. API middleware and server helpers check the actual session.

The proxy also normalizes workspace URLs, such as `/acme/contacts`.
The workspace slug is cosmetic. It does not isolate another customer's data.

Sources: [request proxy](crm/apps/app/proxy.ts), [session helpers](crm/apps/app/lib/session.ts).

## 4. Sign-in, step by step

### Available methods

Password sign-in is disabled in Better Auth.
Configured Google and Microsoft providers supply social sign-in.
An administrator also configures OIDC SSO through the CRM.
Slack supplies an integration connection; it does not supply normal CRM sign-in.

The sign-in page reads `sso.signInOptions` from the API.
It displays configured SSO providers or social sign-in buttons.
The page displays setup instructions when no sign-in method exists.

### The Google or Microsoft path

1. The user clicks a provider button.
2. The client calls `signIn.social()` and supplies return URLs.
3. Browser authentication calls use the application's origin through the `/api` proxy.
4. The API hosts Better Auth under `/api/auth/*`.
5. The provider handles its login and consent screens.
6. Better Auth handles the callback and persists identity information through Prisma.
7. Before creating a new user, the hook checks `ALLOWED_SIGN_IN`.
8. Before creating a session, the hook ensures shared-workspace membership.
9. Better Auth creates the session and sets its cookie.
10. The browser returns to `/`, where setup routing continues.

`ALLOWED_SIGN_IN` supports comma-separated domains and individual addresses.
The inspected allow-list hook runs before **user creation**.
Removing an address from that list does not itself delete an existing user or revoke existing sessions.

### What authentication stores

| Model | Meaning |
| --- | --- |
| `User` | The CRM employee's identity |
| `Account` | Linked provider identity, granted scopes and token fields |
| `Session` | An authenticated session and its expiry |
| `Organization` | The singleton shared workspace |
| `Member` | A user's workspace role |

An employee `User` differs from a customer `Contact`.
Creating a customer contact does not create a login account.

The configured session lifetime is seven days, with an update age of one day.
Better Auth also enables a five-minute cookie cache.
Session creation notifies registered sign-in handlers. One handler starts a best-effort backfill sweep.

Sources: [sign-in page](crm/apps/app/app/(landing)/sign-in/page.tsx), [provider button](crm/apps/app/app/(landing)/sign-in/social-sign-in.tsx), [browser auth client](crm/packages/auth/src/client.ts), [auth configuration](crm/packages/auth/src/auth.ts), [allow-list](crm/packages/auth/src/workspace.ts), [backfill hook](crm/apps/api/src/backfill/backfill.service.ts).

## 5. Workspace membership and permissions

The application uses the literal singleton workspace ID `workspace`.
The first enrollment pass makes the oldest user the owner and enrolls other users as members.
Later sign-ins upsert membership. Roles include `owner`, `admin`, and `member`.

There is no normal invitation-based workspace selection flow.
CRM records do not carry tenant-scoping `organizationId` fields.
The organization plugin supplies workspace identity and membership, rather than multi-tenant separation.

Authentication proves who the user is. Service-level checks decide which actions the user controls.
Examples include workspace changes, connection management, currency configuration and agent management.
The workspace service prevents demotion of the last owner.

Sources: [membership and permissions](crm/packages/auth/src/organization.ts), [workspace service](crm/apps/api/src/workspace/workspace.service.ts), [schema](crm/packages/db/prisma/schema.prisma).

## 6. What happens after sign-in

The Next.js proxy reads two API answers concurrently:

| Check | Procedure | Result |
| --- | --- | --- |
| Workspace setup | `workspace.get` | Workspace name, website, onboarding state, permissions and slug |
| Research setup | `settings.researchKey` | Whether a Context key exists |

The routing order prioritizes workspace setup, then research setup.

**Workspace setup:** an eligible owner or administrator supplies the workspace name and website.
`workspace.update` normalizes the domain, stores the name and slug, and records onboarding metadata.
A website change also queues a `workspace-profile` agent task.

**Research setup:** `/onboarding/research` asks for a Context API key.
`settings.setResearchKey` asks the agent to verify it and stores it through database settings helpers.
An explicitly invalid key is rejected. An inconclusive verification does not prevent storage.
The save also starts a best-effort company backfill.

**Mailbox consent:** pages using `requireMailboxAccess()` also inspect linked-account scopes.
Accounts with only mailbox-provider sign-ins and no complete mailbox grant go to `/grant-access`.
An SSO identity does not require mailbox consent under that rule.

The setup gate reads time out after two seconds. An unknown response lets routing continue.
That routing behavior does not replace API authentication checks.

**Redesign dependency:** optional research-provider handling exists in the backend, but normal UI routing requires the Context key.
Removing the agent therefore also requires changes to onboarding and settings.

Sources: [proxy](crm/apps/app/proxy.ts), [gate readers](crm/apps/app/lib/onboarding.ts), [workspace update](crm/apps/api/src/workspace/workspace.service.ts), [settings service](crm/apps/api/src/settings/settings.service.ts), [mailbox grant rules](crm/packages/auth/src/scopes.ts).

## 7. Loading a screen

A contacts page has two cooperating parts.
The server page checks the session and prepares data through the server tRPC client.
The browser component renders the interactive table and uses TanStack Query for browser data state.

The server tRPC client calls `${API_URL}/api/trpc` and forwards request cookies.
The browser tRPC client calls `/api/trpc` on the Next.js origin.
The Next.js catch-all proxy forwards that request to the API.

Inside NestJS:

1. tRPC builds a context with the request and Better Auth session.
2. `AuthMiddleware` rejects a request without a session user.
3. The router validates input with its Zod contract.
4. The service executes business rules and Prisma queries.
5. The response returns to TanStack Query and the screen.

Routers use aliases such as `contacts`, `companies`, `deals`, `workspace`, and `settings`.
List queries filter, sort and paginate in the database.

The API also exposes a generated REST bridge under `/rest` and runtime OpenAPI documentation.
The main application uses tRPC. A redesign must account for the REST bridge as another API surface.

Sources: [contacts page](crm/apps/app/app/(app)/[slug]/contacts/page.tsx), [browser client](crm/apps/app/lib/trpc/client.tsx), [server client](crm/apps/app/lib/trpc/server.ts), [API proxy](crm/apps/app/app/api/[...path]/route.ts), [context](crm/apps/api/src/trpc/trpc.context.ts), [authentication middleware](crm/apps/api/src/trpc/middlewares/auth.middleware.ts), [contacts router](crm/apps/api/src/contacts/contacts.router.ts), [API rules](crm/docs/api.md).

## 8. Example: creating a contact

1. The contact form calls `contacts.create`.
2. The router validates the request and calls `ContactsService`.
3. The service normalizes values and resolves the company association.
4. A transaction creates the contact and records its CRM event task.
5. The service queues identification work through `AgentTriggerService`.
6. Dynamic-field backfill work is also requested.
7. The browser invalidates affected cached queries and reloads their data.
8. The agent processes queued work separately from the form request.

These writes have different purposes:

- `Contact` stores the customer record.
- `AgentTask` stores work that remains due.
- Fact, brief and event tables store research evidence and execution history.

The event and contact transaction does not include every later background operation.
Avoid treating the complete feature as one atomic database change.

Browser mutations use `useCrmCache()` to invalidate related lists, details and dashboard queries.
Background enrichment uses polling while records have pending or running statuses.

Sources: [create contact form](crm/apps/app/app/(app)/[slug]/contacts/create-contact-sheet.tsx), [contact service](crm/apps/api/src/contacts/contacts.service.ts), [task writer](crm/apps/api/src/agent/agent-trigger.service.ts), [cache invalidation](crm/apps/app/lib/trpc/cache.ts).

## 9. The main business data

| Area | Models and meaning |
| --- | --- |
| Companies | `Company`, company profile and enrichment data |
| People | `Contact`, `ContactFact`, `ContactBrief` |
| Sales | `Deal`, `DealContact`, `ExchangeRate` |
| History | `Activity`, `EmailThread`, `EmailMessage`, calendar events and attendees |
| Customization | `FieldDefinition`, `FieldOption`, `FieldValue`, `SavedView` |
| Background research | `AgentTask`, `AgentEvent` |
| Chat and automation | Conversations, submissions, agent definitions, versions, triggers, runs, actions and audit events |
| Website acquisition | Tracked domains, visitors, events and form submissions |
| Workspace | Organization, members, workspace profile and application settings |

A company groups contacts and deals. `DealContact` attaches people to a deal.
The service checks that attached contacts belong to the deal's company.
Activities connect records to notes, calls, emails, meetings, tasks and stage changes.

Deals use sales stages such as `DEMO_BOOKED`, `CONTRACT_SENT`, `CLOSED_WON`, and `CLOSED_LOST`.
These stages describe sales opportunities. The planned commerce order lifecycle needs separate modeling.

Money has both an original amount/currency and converted reporting fields.
Reports sum `baseAmount` with matching `baseCurrency`.
Changing money fields also requires review of conversion services and dashboard queries.

Source: [complete schema](crm/packages/db/prisma/schema.prisma), [currency rules](crm/docs/currency.md).

## 10. Email and calendar synchronization

The implemented provider scopes cover Gmail, Google Calendar and Microsoft Outlook mail.
Microsoft calendar sync is not part of the inspected provider scopes.

1. OAuth stores linked-account credentials and granted scopes.
2. Connection reconciliation creates per-user, per-source `MailboxSync` rows.
3. The scheduled mailbox endpoint finds due rows and claims work.
4. Provider services obtain tokens, fetch changes and parse provider responses.
5. Gmail and Outlook produce a shared `IncomingMessage` format.
6. `ThreadWriterService` matches records and stores messages, threads and email activities.
7. Cursor and status fields record progress for later sync runs.

The Vercel configuration schedules `/internal/sync/mailboxes` every five minutes.
The endpoint accepts a `CRON_SECRET` bearer token. It refuses execution without the configured secret.
Manual provider sync procedures also exist.

Gmail uses a history cursor. Outlook uses a timestamp cursor with overlap.
RFC message IDs support deduplication and cross-provider threading.
Outlook also has a provider-specific conversation fallback.

Sync does not turn every address into a customer.
Matching excludes internal identities, suppressed addresses and domains, and machine addresses.
Email auto-creation also requires the source setting and an outbound message or known outbound reply.
New Gmail and Outlook email sync rows default to auto-creation disabled.
New Google Calendar rows default to auto-creation enabled, with additional attendee checks.

Sources: [scopes](crm/packages/auth/src/scopes.ts), [sync coordinator](crm/apps/api/src/sync/mailbox-sync.service.ts), [cron authorization](crm/apps/api/src/sync/sync.controller.ts), [schedules](crm/apps/api/vercel.json), [thread writer](crm/apps/api/src/mailbox/thread-writer.service.ts), [Google connection](crm/apps/api/src/google/google-connection.service.ts), [Microsoft connection](crm/apps/api/src/microsoft/microsoft-connection.service.ts).

## 11. Background research and the agent

The API records events and requests work. The agent owns research decisions and provider calls.
The durable handoff is a PostgreSQL task row.
An optional HTTP dispatch signal accelerates processing after task creation.
An API startup heartbeat also requests queue draining while that process remains active.

The agent claims tasks using `FOR UPDATE SKIP LOCKED` and a lease.
Claims consider due time, priority, current lease and attempt limits.
An expired lease permits another claim. Exhausted work has a retirement path.

Dispatch separates two lanes:

- Logos and portraits use direct processing without a model conversation.
- Research tasks use eve sessions and tools.

The minute schedule also reconciles stale tasks, applies eligible blank-field suggestions, and dispatches custom agent work.
Under `eve dev`, cron schedules do not fire automatically.
The configured bridge signal, API heartbeat, or manual dispatch supplies local execution paths.
Production scheduling still requires the runtime and deployment schedule setup.

Contact facts pass through explicit evidence rules.
Human-entered values take priority. Eligible blank fields and verified facts have application paths.
Other retained conflicts become proposals. Facts below the evidence floor are discarded.

The agent accesses Prisma directly. Agent writes do not all pass through NestJS services.

Sources: [API task creation](crm/apps/api/src/agent/agent-trigger.service.ts), [heartbeat](crm/apps/api/src/agent/dispatch-heartbeat.service.ts), [leases](crm/apps/agent/agent/lib/tasks.ts), [dispatch schedule](crm/apps/agent/agent/schedules/dispatch.ts), [fact writes](crm/apps/agent/agent/lib/facts.ts), [agent guide](crm/docs/agent.md), [local schedule behavior](crm/docs/setup.md).

## 12. Record chat and custom automations

**Record chat:** the browser uses Next.js `/eve/v1/*` routes.
The route checks sign-in and relevant conversation ownership.
It removes browser cookies from the forwarded request and creates a signed bridge token.
The agent verifies that token through its eve channel.
Responses stream through the proxy. CRM conversation and event records support application history.

**Builder chat:** the API persists builder messages as `AgentConversationSubmission` rows.
The client request ID supports replay detection.
The agent consumes pending submissions and authors draft artifacts.

**Custom automations:** definitions, versions, instructions, manifests and triggers live in database records.
The deploy operation checks permissions and requires a validated version.
It changes the current version, enables that version's triggers and writes an audit event.
Here, deploying an automation activates database-backed configuration for the existing runtime.

Scheduled and event-driven work produces queued runs.
The agent runner executes those runs and records outcomes.
Connection capabilities, run approval handling and action audit records affect this path.
Review those controls before replacing automation execution.

Sources: [chat proxy](crm/apps/app/app/eve/v1/[...path]/route.ts), [token verification](crm/apps/agent/agent/channels/eve.ts), [builder submissions](crm/apps/api/src/conversations/conversations.service.ts), [definition deployment](crm/apps/api/src/agent/agent-definitions.service.ts), [custom dispatch](crm/apps/agent/agent/lib/custom-agent-dispatch.ts), [runner](crm/apps/agent/agent/subagents/agent_runner/agent.ts).

## 13. Other behavior that affects a redesign

- **Archive:** setting `archivedAt` hides a record without deleting its history.
- **Purge:** permanent deletion performs cleanup and suppression. Contact suppression prevents sync recreation.
- **Retention:** scheduled pruning removes expired archived records through the purge path.
- **Tracking:** the app serves tracking scripts; the API accepts events and files eligible form submissions as contacts.
- **Slack:** shared workspace connectivity supports automation actions and identity matching.
- **Images:** optional Blob storage supports copied logos and portraits.
- **Redis:** optional API caching falls back to process-local memory.
- **Telemetry:** shared package code reports installation and product events, with disable controls.

Sources: [API archive rules](crm/docs/api.md), [tracking rules](crm/docs/tracking.md), [connection rules](crm/docs/connections.md), [cache implementation](crm/apps/api/src/cache/cache.module.ts), [capability detection](crm/apps/agent/agent/lib/capabilities.ts), [telemetry disable logic](crm/packages/telemetry/src/disabled.ts).

## 14. Dependencies to address during major changes

| Proposed change | Areas that need coordinated changes |
| --- | --- |
| Replace authentication | Better Auth config, provider callbacks, cookies, Next.js gates, API context, membership, API keys and agent identity bridge |
| Support multiple customer workspaces | Record schema, query scoping, uniqueness, permissions, tasks, sync state, settings, tracking, secrets and agent context |
| Route all data through the API | Next.js server auth/database reads and agent Prisma reads/writes |
| Replace tRPC | Routers, contracts, browser/server clients, generated router types, cache keys and the REST/OpenAPI bridge |
| Remove or replace AI | Task creation, enrichment status, onboarding research gate, record panels, chat, builder, custom runs and dispatch |
| Introduce a queue worker | Task durability, leases, retries, duplicate protection, reconciliation, schedules and deployment lifecycle |
| Rename or split packages | Workspace imports, package exports, Turbo dependency order and app build configuration |
| Change the database schema | Prisma migrations, generated client, API services, agent tools, validation contracts, UI types and fixtures |
| Change deployment platform | API serverless entry, cron triggers, agent schedules, background work, environment loading and proxy URLs |
| Add commerce operations | Orders, order lines, statuses, inventory ownership, provider adapters, webhook durability and messaging/shipping workflows |

These are dependency findings, not a requirement to preserve the current architecture.
The source offers useful modules, but the shared database currently supplies much of their integration contract.

## 15. Suggested reading order

1. [Schema](crm/packages/db/prisma/schema.prisma): understand identities, CRM records and automation state.
2. [Auth configuration](crm/packages/auth/src/auth.ts) and [membership](crm/packages/auth/src/organization.ts): understand access and the singleton workspace.
3. [Next.js proxy](crm/apps/app/proxy.ts): understand navigation and setup restrictions.
4. [Contacts service](crm/apps/api/src/contacts/contacts.service.ts): follow one business operation completely.
5. [Task writer](crm/apps/api/src/agent/agent-trigger.service.ts) and [task leases](crm/apps/agent/agent/lib/tasks.ts): follow the background handoff.
6. [Mailbox coordinator](crm/apps/api/src/sync/mailbox-sync.service.ts) and [thread writer](crm/apps/api/src/mailbox/thread-writer.service.ts): understand imported data.
7. [Agent definitions](crm/apps/api/src/agent/agent-definitions.service.ts): understand the separate automation lifecycle.
8. [Setup](crm/docs/setup.md), [Turbo configuration](crm/turbo.json), and [deployment schedules](crm/apps/api/vercel.json): understand runtime assumptions.

## 16. Redesign preparation

First, choose the target domain: sales CRM, commerce operations, or both.
Next, choose the tenancy model and the owner of each database write.
Then define authentication, API contracts and background execution boundaries.
Use those decisions to plan migrations and replace one complete user flow at a time.

A useful first flow is: sign in, complete setup, create a contact, read it back, and observe queued work.
For each replacement, check its access rules, saved data, duplicate behavior, and visible failure state.
This preserves observable behavior while the internal architecture changes.

## Issues and review limits

1. **NOT DONE** — Shopify, WhatsApp and Ozone workflows remain planned or blocked in the root project status.
   Fix: implement the commerce domain and provider workflows separately.
2. **RISK** — The singleton workspace provides no tenant separation. Multiple customer organizations share unscoped CRM data under this design.
   Fix: redesign record ownership and query authorization before introducing multiple tenants.
3. **RISK** — The allow-list check runs during user creation. Allow-list removal alone does not revoke existing access.
   Fix: define account disablement and session revocation before changing access policy.
4. **RISK** — Sign-in backfill starts detached asynchronous work. Serverless request completion does not guarantee that work finishes.
   Fix: move required work into a durable queue or runtime-managed background task.
5. **UNKNOWN** — Runtime credentials, live provider grants and complete user journeys are not verified in this source review.
   Fix: verify the chosen flows against an isolated development database before restructuring them.
