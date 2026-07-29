# Commerce Operations CRM — Integrations

## 1. Integration principles

- Domain modules consume normalized provider ports.
- Shopify, WhatsApp, and courier-specific HTTP/SDK types remain inside adapters.
- Webhooks are verified from raw bytes, durably stored, and processed asynchronously.
- External calls use typed timeouts, bounded retries, rate-limit awareness, correlation, and safe errors.
- Every outbound side effect has a durable operation and stable idempotency key.
- Unknown provider behavior is a blocker, not an invitation to infer endpoints or statuses.
- Raw provider status is retained alongside normalized status.
- Credentials are server-only and redacted from logs, errors, OpenAPI examples, and frontend bundles.

## 2. Shared contract types

The following TypeScript-shaped definitions express semantic contracts. Final code may split them into files, but provider adapters must preserve these meanings.

```ts
type ProviderKey =
  | "SHOPIFY"
  | "WHATSAPP_CLOUD"
  | "FAKE_COURIER"
  | "OZONE";

type ProviderHealthStatus = "HEALTHY" | "DEGRADED" | "UNAVAILABLE" | "BLOCKED";

type ProviderHealth = {
  status: ProviderHealthStatus;
  checkedAt: string;
  safeCode?: string;
  safeMessage?: string;
  capabilities: string[];
};

type WebhookVerificationInput = {
  rawBody: Uint8Array;
  headers: Readonly<Record<string, string | string[] | undefined>>;
  query: Readonly<Record<string, string | string[] | undefined>>;
  receivedAt: string;
};

type VerifiedWebhook = {
  provider: ProviderKey;
  externalAccountId: string;
  externalEventId: string;
  eventType: string;
  occurredAt?: string;
  payload: unknown;
};

type PullPageInput = {
  cursor?: string;
  updatedAfter?: string;
  limit: number;
};

type PullPage<T> = {
  items: T[];
  nextCursor?: string;
  highWaterMark?: string;
};

type ProviderErrorKind =
  | "AUTHENTICATION"
  | "AUTHORIZATION"
  | "VALIDATION"
  | "NOT_FOUND"
  | "CONFLICT"
  | "RATE_LIMIT"
  | "TIMEOUT"
  | "TRANSIENT"
  | "UNAVAILABLE"
  | "UNSUPPORTED"
  | "UNKNOWN_RESULT";

type ProviderError = {
  kind: ProviderErrorKind;
  safeCode: string;
  safeMessage: string;
  retryable: boolean;
  retryAfterMs?: number;
  providerRequestId?: string;
};
```

Provider errors never contain authorization headers, raw credentials, unredacted request bodies, or unnecessary customer data.

## 3. `CommerceProvider`

```ts
interface CommerceProvider {
  readonly providerKey: "SHOPIFY";

  verifyWebhook(input: WebhookVerificationInput): Promise<VerifiedWebhook>;
  healthCheck(): Promise<ProviderHealth>;

  pullProducts(input: PullPageInput): Promise<PullPage<ExternalProduct>>;
  pullLocations(input: PullPageInput): Promise<PullPage<ExternalLocation>>;
  pullInventory(input: PullInventoryInput): Promise<PullPage<ExternalInventoryLevel>>;
  pullCustomers(input: PullPageInput): Promise<PullPage<ExternalCustomer>>;
  pullOrders(input: PullPageInput): Promise<PullPage<ExternalOrder>>;
  getOrder(input: GetExternalResourceInput): Promise<ExternalOrder>;

  upsertFulfilmentTracking(
    input: FulfilmentTrackingInput,
  ): Promise<ExternalFulfilmentResult>;
}

type GetExternalResourceInput = {
  externalId: string;
};

type PullInventoryInput = PullPageInput & {
  externalLocationIds?: string[];
  externalInventoryItemIds?: string[];
};

type FulfilmentTrackingInput = {
  idempotencyKey: string;
  externalOrderId: string;
  externalFulfilmentOrderIds: string[];
  lines: Array<{
    externalOrderLineId: string;
    quantity: number;
  }>;
  tracking: {
    number: string;
    url?: string;
    company?: string;
  };
  notifyCustomer: boolean;
};
```

Normalized external product/customer/order types must contain:

- provider identity and source timestamps;
- typed facts needed by the domain;
- raw status strings where needed for diagnosis;
- no provider SDK class instances;
- no assumption that numeric IDs fit JavaScript safe integers.

## 4. Shopify adapter

### 4.1 Responsibilities

- Authenticate to the Shopify Admin API with Shopify's client-credentials grant for the current same-organization development app/store.
- Treat [Shopify's client-credentials documentation](https://shopify.dev/docs/apps/build/authentication-authorization/access-tokens/client-credentials-grant) and [GraphQL Admin API `2026-07`](https://shopify.dev/docs/api/admin-graphql/2026-07) as the source of truth.
- Exchange `SHOPIFY_CLIENT_ID` and `SHOPIFY_CLIENT_SECRET` server-side at `https://{shop}.myshopify.com/admin/oauth/access_token`, cache the returned token securely, and renew it automatically because it expires after 24 hours.
- Pin and expose `SHOPIFY_API_VERSION=2026-07`.
- Register or validate required webhook subscriptions where supported by the app model.
- Verify webhook HMAC against raw request bytes and bind the event to the configured shop.
- Import and reconcile products, variants, locations, inventory, customers, orders, and required fulfilment references.
- Map Shopify entities into normalized contract models.
- Respect rate-limit/throttle information and cursored pagination.
- Create or update Shopify fulfilment tracking only after shipment creation and policy validation.
- Surface permissions, unsupported scenarios, throttling, and user errors as typed safe failures.

### 4.2 Initial synchronization order

1. Validate shop identity, token, API version, and granted capabilities.
2. Import locations.
3. Import products/variants and inventory-item mappings.
4. Import inventory levels.
5. Import customers.
6. Import orders and line/address snapshots.
7. Import required fulfilment-order/fulfilment references for open operational orders.
8. Record a high-water mark and run reconciliation for changes that occurred during the import.

Each resource sync is resumable, cursor-based, observable, and safe to rerun. Webhook intake can begin before the bulk import completes because external mappings/upserts are idempotent.

### 4.3 Shopify order webhook workflow

1. Shopify calls the public HTTPS endpoint.
2. API applies request size/content-type/rate controls.
3. Adapter verifies HMAC using the exact raw body.
4. API validates the shop domain/account matches a known active connection.
5. Durable inbox insert deduplicates using Shopify's webhook event identifier.
6. API returns success after durable acceptance, not after full order processing.
7. Worker rejects stale overwrites using source update time/version policy.
8. Worker fetches the full authoritative order if the webhook payload does not contain all required facts.
9. Transaction upserts external mapping, customer, addresses, order, lines, independent statuses, transition/timeline metadata, and relevant outbox events.
10. Reconciliation later repairs missed or out-of-order events.

The exact topic list is not frozen in this document because it depends on the pinned Shopify API version and approved scopes. At minimum, the readiness review must cover order create/update/cancel changes, product/variant changes, inventory changes, customer changes, fulfilment changes, app uninstall, and mandatory privacy/compliance events applicable to the app.

### 4.4 Inventory rule

Shopify is the source of truth and normally adjusts tracked inventory as part of Shopify order processing. The CRM:

- mirrors inventory by Shopify location and variant/inventory item;
- never decrements inventory because an order was imported, confirmed, or shipped;
- displays `last_synced_at` and stale state;
- processes inventory events and scheduled reconciliation;
- does not offer inventory-edit controls in MVP.

### 4.5 Shopify outbound fulfilment workflow

1. A created shipment with valid tracking commits an outbox intent.
2. Worker resolves order line mappings and Shopify fulfilment-order references.
3. Adapter validates available assigned quantities and supported ownership/location behavior.
4. Adapter sends the official fulfilment/tracking operation with a stable local operation reference where supported.
5. Provider user errors are terminal and visible; rate limits/transient errors retry.
6. Result stores Shopify fulfilment identity and updates the mirrored fulfilment status.
7. Reconciliation verifies the source state.

The following must be confirmed before implementation: exact Admin API version, required scopes, fulfilment-order ownership, partial fulfilment rules, tracking update semantics, customer notification policy, and whether the installed app may manage the relevant fulfilment orders.

### 4.6 Shopify authentication decision and remaining gates

| Gap | Why it matters | Gate |
|---|---|---|
| Same-organization client-credentials grant | Current development app and store belong to the same Shopify organization | Use for MVP; no OAuth callback |
| Token lifecycle | Token is valid for 24 hours | Cache server-side and renew automatically |
| API version | Schemas, topics, and fulfilment operations change | Pin `2026-07` |
| Access scopes absent | Imports/webhooks/fulfilment may be forbidden | Approve least-privilege scopes |
| Webhook subscriptions absent | Incremental sync cannot be accepted | Approve topic matrix |
| Privacy webhook/policy inputs absent | Compliance risk | Complete before production |
| Fulfilment model untested | Tracking update may fail or duplicate | Test with representative orders |

If the app is later installed on stores belonging to unrelated merchant organizations, use Shopify's authorization-code flow. That onboarding is post-MVP and must not block the current test-store integration. Credentials and tokens are never returned to the frontend or written to logs.

## 5. `MessagingProvider`

```ts
interface MessagingProvider {
  readonly providerKey: "WHATSAPP_CLOUD";

  verifyWebhook(input: WebhookVerificationInput): Promise<VerifiedWebhook>;
  healthCheck(): Promise<ProviderHealth>;

  sendTemplate(input: SendTemplateInput): Promise<SendMessageResult>;
}

type SendTemplateInput = {
  idempotencyKey: string;
  recipientPhoneE164: string;
  template: {
    name: string;
    locale: string;
    parameters: ReadonlyArray<{
      component: "HEADER" | "BODY" | "BUTTON";
      index: number;
      value: string;
    }>;
  };
  businessContext: {
    messageId: string;
    orderId?: string;
  };
};

type SendMessageResult = {
  providerMessageId: string;
  acceptedAt: string;
};
```

The domain passes only pre-approved template parameters. The adapter converts them to the current official Cloud API request format.

## 6. WhatsApp Cloud adapter

### 6.1 Outbound confirmation workflow

1. Order import determines eligibility without blocking order persistence.
2. In the same transaction, the domain inserts a `message` and outbox event with key `order:{orderId}:message:ORDER_CONFIRMATION:v1`.
3. Worker validates active connection, recipient E.164 phone, approved business consent policy, template mapping, locale, and parameter completeness.
4. Message transitions `QUEUED → SUBMITTING`.
5. Adapter calls the official WhatsApp Business Cloud API using a server-held access token.
6. Accepted result stores provider message ID and `SUBMITTED`.
7. Verified webhook status events project `SENT`, `DELIVERED`, `READ`, or `FAILED`.
8. Retryable transport/rate failures use bounded backoff. Validation, missing template, consent/configuration, and provider rejection become visible non-retryable failures until corrected.

The provider may accept a request while the local client times out. Because local idempotency alone cannot prove the provider did not accept it, automatic retry policy must be based on official Meta behavior and a risk review; it must not promise exactly-once delivery.

### 6.2 Webhook workflow

- Handle initial webhook verification/challenge exactly as official Meta documentation requires.
- Verify event authenticity using the official signature mechanism and app secret.
- Deduplicate status events by stable provider identity or a documented conservative fingerprint.
- Resolve by provider message ID.
- Store raw and normalized status plus source event time.
- Retain unknown statuses and errors for analysis.
- Do not implement inbound customer conversation handling in MVP; unsupported inbound message events are acknowledged, classified, and ignored or retained according to approved minimization policy.

### 6.3 Template management boundary

MVP does not create or approve templates in Meta. It stores mappings to templates already approved in WhatsApp Manager:

```text
purpose → template name + locale + ordered parameter mapping
```

At startup/configuration test, the adapter should validate that configured templates are usable if the official API and granted permissions permit. A template mismatch prevents that message purpose from sending but does not stop order synchronization.

### 6.4 WhatsApp readiness gaps

- Meta app ID and app secret.
- WhatsApp Business Account ID and phone number ID.
- Supported permanent system-user access token and rotation owner.
- Webhook verify token and public callback URL.
- Approved template names, categories, locales, exact variable definitions, and example values.
- Customer opt-in/consent evidence and policy for transactional templates.
- Default country/phone normalization rules.
- Test recipient numbers and Meta app/business verification state.
- Rate/quality limits and production alert thresholds.

## 7. `CourierProvider`

```ts
interface CourierProvider {
  readonly providerKey: "FAKE_COURIER" | "OZONE";

  capabilities(): CourierCapabilities;
  healthCheck(): Promise<ProviderHealth>;
  verifyWebhook(input: WebhookVerificationInput): Promise<VerifiedWebhook>;

  createShipment(input: CreateShipmentInput): Promise<CreateShipmentResult>;
  getShipment(input: GetShipmentInput): Promise<ExternalShipment>;
  cancelShipment(input: CancelShipmentInput): Promise<CancelShipmentResult>;
}

type CourierCapabilities = {
  createShipment: boolean;
  lookupByProviderId: boolean;
  lookupByClientReference: boolean;
  cancelShipment: boolean;
  statusWebhooks: boolean;
  statusPolling: boolean;
  providerIdempotencyKey: boolean;
};

type CreateShipmentInput = {
  idempotencyKey: string;
  clientReference: string;
  recipient: {
    name: string;
    phoneE164: string;
    addressLine1: string;
    addressLine2?: string;
    city: string;
    region?: string;
    postalCode?: string;
    countryCode: string;
  };
  parcel: {
    items: ReadonlyArray<{
      clientLineReference: string;
      description: string;
      quantity: number;
    }>;
  };
  collection: {
    amountMinor: number;
    currencyCode: string;
  };
};

type CreateShipmentResult = {
  providerShipmentId: string;
  trackingNumber: string;
  trackingUrl?: string;
  rawStatus?: string;
  createdAt?: string;
};

type GetShipmentInput = {
  providerShipmentId?: string;
  clientReference?: string;
};

type ExternalShipment = {
  providerShipmentId: string;
  clientReference?: string;
  trackingNumber: string;
  trackingUrl?: string;
  rawStatus: string;
  occurredAt?: string;
};

type CancelShipmentInput = {
  providerShipmentId: string;
  clientReference: string;
  reasonCode: string;
};

type CancelShipmentResult = {
  providerShipmentId: string;
  rawStatus?: string;
  cancelledAt?: string;
};
```

The common contract does not claim that Ozone supports every method or field. `capabilities()` controls application behavior; unsupported operations return a typed unsupported result and are disabled in UI.

## 8. Shipment orchestration

### 8.1 Preconditions

Before creating a shipment, the application service verifies:

- order belongs to organization and is `CONFIRMED`;
- caller or workflow is authorized;
- active courier connection is configured;
- one active primary shipment does not already exist;
- shippable lines and positive quantities exist;
- recipient and dispatch address meet normalized domain requirements;
- COD amount/currency policy is defined;
- order has not become source-cancelled or otherwise blocked;
- provider-specific required values are validated by the adapter only after the generic checks.

Successful readiness evaluation transitions the order to `READY_FOR_SHIPMENT` and creates the shipment/outbox intent atomically.

### 8.2 Create operation

1. Worker claims the unique integration operation.
2. Shipment moves `READY → SUBMITTING` using expected version.
3. Adapter is called outside the database transaction.
4. Success transaction stores provider/tracking values, mappings, shipment event, audit/system event, Shopify fulfilment outbox, and appropriate CRM transition.
5. Known retryable failure stores next attempt without creating another logical operation.
6. Known terminal failure remains visible for correction/manual action.
7. Timeout/connection loss after request submission is classified:
   - if lookup by client reference is officially supported, reconcile before any resend;
   - if provider idempotency is officially supported, retry with the identical key under documented rules;
   - if neither is known, mark `RECONCILIATION_REQUIRED` and stop automatic creation.

### 8.3 Status ingestion

The architecture supports verified webhooks, scheduled polling, or both. The chosen Ozone mechanism is unknown. A status mapper returns:

```ts
type CourierStatusMapping = {
  rawStatus: string;
  normalizedStatus:
    | "UNKNOWN"
    | "CREATED"
    | "IN_TRANSIT"
    | "OUT_FOR_DELIVERY"
    | "DELIVERED"
    | "FAILED"
    | "REFUSED"
    | "RETURNED"
    | "CANCELLED";
  proposedOrderTransition?: string;
  isTerminal: boolean;
  requiresReview: boolean;
};
```

Only approved mapping rules may propose an order transition. Unknown, out-of-order, or invalid transitions are recorded and flagged.

## 9. `FakeCourierProvider`

Purpose: unblock local development and acceptance of CRM orchestration before live Ozone inputs exist.

Required behavior:

- Uses no external network.
- Produces deterministic provider shipment and tracking identifiers from a stable client reference.
- Deduplicates repeated `createShipment` calls by client reference.
- Supports scripted success, retryable failure, terminal failure, and ambiguous-result scenarios.
- Supports deterministic manual/test status progression through representative normalized states.
- Emits simulated events through an internal test/admin path unavailable in live mode.
- Stores provider key `FAKE_COURIER` on every connection, shipment, operation, and event.
- Displays a persistent “Simulated courier — not Ozone” warning.
- Refuses startup in production when a configuration policy requires a live courier.

It must not:

- use the Ozone name, branding, endpoint shapes, fields, authentication, or status labels;
- claim that a simulated tracking number is real;
- be presented as completion of the live Ozone integration.

## 10. Ozone Express readiness gate

No live adapter code should be written until all required items are supplied and reviewed:

1. Official versioned API documentation and terms.
2. Sandbox/test base URL and production base URL.
3. Authentication and credential rotation procedure.
4. Shipment creation request/response schema, required fields, validation rules, currency/COD representation, and error schema.
5. Idempotency-key or unique client-reference semantics.
6. Lookup/search endpoints capable of resolving ambiguous creation.
7. Tracking identifiers and tracking URL rules.
8. Complete status catalogue with terminality, order, timestamps, refusal, failed delivery, cancellation, and return meanings.
9. Webhook authentication/delivery/retry contract, or polling endpoints and recommended interval.
10. Cancellation and retry/recreate rules.
11. Rate limits, timeout recommendations, service availability, and support contact.
12. Test credentials and representative sandbox scenarios.

After receipt:

- archive the approved documentation reference/version;
- build an explicit external-to-normalized field mapping;
- build an approved status transition table;
- update capabilities truthfully;
- write adapter contract tests from official examples;
- run duplicate-click, retry, timeout, invalid-address, refusal, return, cancellation, and reconciliation tests;
- obtain product/operations sign-off before enabling `provider_key=OZONE`.

## 11. Retry matrix

| Failure class | Default action |
|---|---|
| Authentication/authorization | No automatic retry; alert/configuration action |
| Provider validation/user error | No automatic retry; expose field-safe diagnostics |
| Not found | Reconcile mapping; terminal unless eventual consistency is documented |
| Conflict/duplicate | Fetch/reconcile using official provider capability |
| Rate limit | Retry after provider hint or exponential backoff with jitter |
| Network failure before send can be established | Retry boundedly when operation is safe |
| Timeout/connection loss after possible create | Reconcile first; never blind-create |
| Provider 5xx/unavailable | Bounded retry unless create result is ambiguous |
| Unknown response/schema | Terminal/degraded alert; preserve sanitized evidence |

Retry count, maximum delay, and total retry window are configurable per provider operation. No infinite retries.

## 12. Integration observability

For each connection show:

- provider and non-secret account identity;
- enabled capabilities;
- configured API version where applicable;
- latest health result;
- last webhook received/processed;
- last successful initial/incremental/reconciliation sync;
- queue/outbox lag;
- recent terminal and retryable error counts;
- credential presence/expiry metadata where safely available.

The UI and logs use safe codes and correlation IDs. Restricted diagnostic views may expose minimized provider request IDs and redacted payload fragments, never tokens or full customer records.

## 13. Required pre-coding integration decisions

- Shopify client-credentials configuration and least-privilege scope matrix for the current store.
- Pinned Shopify GraphQL Admin API `2026-07` and webhook topic matrix.
- Shopify fulfilment ownership/allocation and notify-customer policy.
- WhatsApp confirmation template, locale selection, variable map, and consent policy.
- Automatic versus operator-triggered order confirmation message.
- Ozone documentation availability; otherwise release only the technical demonstration: live Shopify, live WhatsApp, and clearly labelled `FakeCourierProvider`.
- Default country, E.164 normalization, COD amount calculation, and address validation policy.
