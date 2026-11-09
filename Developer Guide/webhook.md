# Webhooks

ChaChing Webhooks allow your system to receive **real-time notifications** when events occur in your ChaChing account (e.g., customer updates, payments, subscriptions).

Webhooks are designed for clients using the **public API**, which allows them to react to system changes without polling.

---

# How Webhooks Work

```mermaid
graph TD;
  A["Event occurs in ChaChing (e.g. invoice.paid)"] --> B["Generate event notification"];
  B --> C["Find the active webhook endpoints that listen to the event type"];
  C --> D["Send HTTP POST request to webhook endpoint"];
  D --> E["Client processes request"];
  E --> F{"Response is HTTP 2xx?"};

  F -- Yes --> G["Set delivery status to success"];
  F -- No --> K{"Permanent rejection? (any status below 500 except 408 and 429)"};

  K -- Yes --> L["Stop immediately - Set delivery status to failure"];
  K -- No --> H["Retry delivery (according to retry policy)"];
  H --> I{"Max retries reached?"};

  I -- No --> D;
  I -- Yes --> J["Stop retries - Set delivery status to failure"];
```

1. An event occurs in ChaChing (e.g., customer details updated). For details related to the event types, refer to the section [Event Types](#event-types).
2. ChaChing generates an event notification.
3. ChaChing identifies the **active webhook endpoints** of the account whose `enabled_events` contains the event type or `*`.
4. ChaChing sends the event with its payload as an **HTTP POST request** to each webhook endpoint.
    Note: For payload details, refer to the **Event Structure** section.
5. Your system receives and processes the event.
6. Your endpoint must return an HTTP `2xx` response:
    - A `2xx` response sets the delivery status to `success`
    - A `5xx` response is retryable: the delivery status is `retry` while retries remain, and the delivery is retried according to the retry policy below
    - An HTTP `408` (Request Timeout) and an HTTP `429` (Too Many Requests) are also treated as retryable
    - No response at all — network failure, DNS failure, connection timeout, read timeout — is also treated as retryable
    - Any other non-`2xx` response is a **permanent rejection**: every `3xx` redirect and every `4xx` other than `408` and `429` (for example `400`, `401`, `403`, `404`, `410`). The delivery status is set to `failure` immediately and the delivery is never retried

---

# Key Concepts

## Webhooks are account-level

- Webhooks are configured **per account**
- Each account has its own webhook endpoints
- Events of one account are never sent to another account's endpoints. An endpoint registered with a platform key at `/platform/webhook-endpoints` receives the events of every account of that platform, with the same event `id` as any endpoint of that account. See [Platforms](./platforms.md#platform-webhooks).

---

## Multiple webhook endpoints

- You can create **multiple webhook endpoints**
- Events are sent to every **active** endpoint of the account that listens to the event type: its `enabled_events` contains the type or `*`
- Each endpoint has its own list of event types, `enabled_events`. Configure each endpoint to receive only the event types your integration needs: listening to every event puts load on your server that your integration does not need.

---

# Webhook Configuration

Webhooks can be managed via the **UI** (Settings, Developer settings, Webhooks) or programmatically via the **REST API**. Both approaches support the full lifecycle: create, list, choose the events each endpoint receives, update, enable/disable, and delete endpoints.

> In the UI, only users with the **Developer** or **SuperAdmin** role can manage webhooks. Through the REST API, the account's secret API key manages them. For user roles details, refer to the section Manage → Users in the left panel of the ChaChing application.

## Via the UI

1. In ChaChing, go to Settings → Developer settings.
2. In the Developer Settings, select the Webhooks tab to:
    - Create webhook endpoints
    - Choose the events each endpoint receives
    - Open an endpoint's detail page by clicking its row
    - View all endpoints and the key that signs each one
    - Enable / disable endpoints
    - Rotate an endpoint's signing secret
    - Edit/Delete endpoints

{% img src="../images/webhookConfiguration.jpg" alt="webhookConfiguration.jpg" withLightbox=true width="" height="" /%}

### Creating a Webhook

1. Open **Settings, Developer settings, Webhooks.**
2. Click **Create webhook.**

{% img src="../images/creatingWebhook.jpg" alt="creatingWebhook.jpg" withLightbox=true width="" height="" /%}

3. Enter the endpoint URL, choose **All events** or **Select events** and tick the event types the endpoint receives, then select **Create endpoint**. An endpoint receives at least one event type.

After creation:

- Webhook becomes **active**
- Events of the selected types start being delivered immediately
- The endpoint's **signing secret** (`whsec_…`) is shown once, with a **Copy** button. Store it on your server: ChaChing never shows it again. If you lose it, rotate the secret.

### Choosing the Events of an Endpoint

1. Open **Developer Settings → Webhooks** and click the endpoint's row to open its detail page.
2. The **Events** section lists the event types the endpoint is **Listening to** and the ones it is **Not listening to**.
3. Click **Add** next to an event type to start receiving it, or **Remove** to stop receiving it. **Add events** opens an empty list with a search box: tick the event types you want, then select **Save** to add them to the endpoint. **Add all** makes the endpoint listen to every event type, including event types added later.
4. On an endpoint that listens to all events, **Choose specific events** opens the same list. Select **Save** to replace "all events" with the event types you ticked. You need at least one.

An endpoint listens to at least one event type; to stop all deliveries, disable the endpoint. The detail page also lists the endpoint's deliveries; click a row to inspect one.

### Inspecting Deliveries

The **Logs** table of the Webhooks tab lists every delivery of the account, and the **Deliveries** table of an endpoint's detail page lists the deliveries of that endpoint. Filter either table by event ID, event type, status, response code and created date; the **Logs** table also filters by endpoint. Click a row to open the delivery in a side panel: its payload and every attempt with its response code, its response body and any connection error. **Copy payload** and **Copy response body** copy the values to your clipboard.

### Rotating a Signing Secret

1. Open **Developer Settings → Webhooks.**
2. Click the key icon (**Rotate signing secret**) on the endpoint's row and confirm.
3. Copy the new signing secret. The dialog also shows until when the previous secret keeps working (24 hours).

---

## Via the REST API

Manage webhook endpoints with the account **secret** API key (`sk_…`) in the `cc-api-key` header. A publishable key (`pk_…`) is refused with `403` and the error code `SECRET_API_KEY_REQUIRED`. Send no `Authorization` header (see [API Basics](./api-basics.md#authentication)). An account holds at most 16 endpoints.

Two endpoints can have the same URL: each receives its own delivery of every event.

### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/webhook-endpoints` | Create a webhook endpoint |
| `GET` | `/webhook-endpoints` | List the account's webhook endpoints, newest first |
| `GET` | `/webhook-endpoints/{id}` | Retrieve a webhook endpoint |
| `PATCH` | `/webhook-endpoints/{id}` | Change the URL and the event types of a webhook endpoint |
| `DELETE` | `/webhook-endpoints/{id}` | Delete a webhook endpoint |
| `POST` | `/webhook-endpoints/{id}/enable` | Enable a webhook endpoint |
| `POST` | `/webhook-endpoints/{id}/disable` | Disable a webhook endpoint |
| `POST` | `/webhook-endpoints/{id}/rotate-secret` | Rotate the endpoint's signing secret |
| `GET` | `/webhook-endpoints/{id}/logs` | List the deliveries sent to a webhook endpoint |
| `GET` | `/webhook-deliveries` | List the account's webhook deliveries, newest first |
| `GET` | `/webhook-deliveries/{id}` | Retrieve a webhook delivery with its payload and attempts |

### The Webhook Endpoint Object

```json
{
  "id": "0b6f7a52-3c1e-4d8a-9f43-2a7d1c5e8b90",
  "object": "webhook_endpoint",
  "endpoint": "https://your-server.com/webhook",
  "enabled_events": ["customer.created", "invoice.payment_succeeded"],
  "status": "active",
  "signature_key": "endpoint_secret",
  "created": 1761091200,
  "updated": 1761091200
}
```

`enabled_events` lists the event types the endpoint receives, in the order of the [Event Types](#event-types) list; `["*"]` means every event type, including event types added later. `status` is `active` or `inactive`; events are sent only to `active` endpoints. `signature_key` says which key signs the deliveries; it is always `endpoint_secret`, the endpoint's own signing secret. `created` and `updated` are Unix seconds. The signing secret itself is never part of this object: it appears only in the create and rotate responses.

### Create an Endpoint

```
POST /webhook-endpoints
```

**Request body:**

```json
{
  "endpoint": "https://your-server.com/webhook",
  "enabled_events": ["customer.created", "invoice.payment_succeeded"]
}
```

`endpoint` must be an `https` URL in production; an `http` URL is accepted only on non-production environments. A URL whose host is `localhost`, ends in `.local`, or is an IPv4 or IPv6 address in a private, loopback or link-local range is refused with `400`. The endpoint is created `active`, and the response is `201` with the Webhook Endpoint object. Creating a 17th endpoint answers `409` with the error code `WEBHOOK_ENDPOINTS_LIMIT_REACHED`. The create response also carries `secret`, the endpoint's signing secret (`whsec_…`). It is returned only in this response: store it on your server.

Every non-public address is refused: private, loopback, link-local, carrier-grade NAT (100.64.0.0/10), multicast, broadcast and documentation ranges. A hostname that resolves to such an address at delivery time is accepted when you save it, and every delivery to it fails with `error: "blocked"`.

`enabled_events` is required: a non-empty list of event types from [Event Types](#event-types), or `["*"]` for every event type, including event types added later. `*` cannot be combined with other values. Duplicates are removed, and the response lists the types in the order of the Event Types list. `enabled_events` takes at most 64 values. The errors are listed in [Event Type Errors](#event-type-errors).

A create with neither field reports the `endpoint` errors first; once `endpoint` is valid, a missing `enabled_events` answers `400` `WEBHOOK_ENABLED_EVENTS_REQUIRED`.

### List and Retrieve Endpoints

```
GET /webhook-endpoints
GET /webhook-endpoints/{id}
```

The list takes the page parameters `page` and `take` (`take` at most `50`) and answers `{ "data": [...], "meta": { "page": 1, "take": 10, "items_total": 2, "pages_total": 1 } }`. `GET /webhook-endpoints` accepts `search`, a URL-encoded JSON object with `endpoint` (exact match).

### Update an Endpoint

```
PATCH /webhook-endpoints/{id}
```

Send at least one of `endpoint` and `enabled_events`; a field you omit keeps its value. A sent `enabled_events` replaces the whole list: to add one event type, send the current list plus the new type. The endpoint keeps its `id`, its `status` and its signing secret. A body with neither field answers `400` with `WEBHOOK_ENDPOINT_UPDATE_EMPTY`.

### Event Type Errors

| Status | `error` | When |
|--------|---------|------|
| `400` | `WEBHOOK_ENABLED_EVENTS_REQUIRED` | The create body has no `enabled_events` |
| `400` | `WEBHOOK_ENABLED_EVENTS_EMPTY` | `enabled_events` is an empty list |
| `400` | `WEBHOOK_EVENT_TYPE_UNKNOWN` | `enabled_events` holds a value that is not an event type; the message names it |
| `400` | `WEBHOOK_ENABLED_EVENTS_WILDCARD_MIXED` | `enabled_events` combines `*` with other values |
| `400` | `WEBHOOK_ENDPOINT_UPDATE_EMPTY` | The update body has neither `endpoint` nor `enabled_events` |

### Enable / Disable an Endpoint

```
POST /webhook-endpoints/{id}/enable
POST /webhook-endpoints/{id}/disable
```

No request body. Enabling an `active` endpoint answers `400` with `WEBHOOK_ENDPOINT_ALREADY_ACTIVE`, and disabling an `inactive` endpoint answers `400` with `WEBHOOK_ENDPOINT_ALREADY_INACTIVE`.

### Delete an Endpoint

```
DELETE /webhook-endpoints/{id}
```

Deletes the endpoint permanently and answers `200` with the Webhook Endpoint object as it was before the deletion.

An `id` that is not one of your endpoints answers `404` with `WEBHOOK_ENDPOINT_NOT_FOUND`.

### List Deliveries

```
GET /webhook-deliveries
GET /webhook-endpoints/{id}/logs
```

`GET /webhook-deliveries` lists every delivery of the account. `GET /webhook-endpoints/{id}/logs` lists the deliveries of one endpoint, and keeps them after you change its URL. Both answer newest first, with the page parameters `page` and `take`; `take` is at most 50, and a larger value answers `400`.

One event sent to two endpoints is two deliveries, each with its own `id` and the same `event`.

Filter with `search`, a URL-encoded JSON object. A filter value that is not valid answers `400`.

| Key | Value | Example |
| --- | --- | --- |
| `event_id` | the exact event id: `evt_` and 24 hexadecimal characters | `{"event_id": "evt_0123456789abcdef01234567"}` |
| `event_type` | one or more event types, comma-separated | `{"event_type": "invoice.created,invoice.finalized"}` |
| `status` | one or more of `pending`, `retry`, `success`, `failure`, comma-separated | `{"status": "failure,retry"}` |
| `response_status_code` | the exact HTTP status | `{"response_status_code": 500}` |
| `created` | a date filter: `type` `equal`, `after` or `before` with `date`; `between` with `from` and `to`; `inTheLast` with `amount` and `unit` (`days`, `weeks`, `months` or `years`) | `{"created": {"type": "after", "date": "2026-10-01"}}` |
| `endpoint` | text contained in the URL, on `GET /webhook-deliveries` only | `{"endpoint": "example.com"}` |

A search key outside this table answers `400`.

### The Webhook Delivery Object

| Field | Description |
| --- | --- |
| `id` | The delivery id, `whd_…` |
| `object` | `webhook_delivery` |
| `event` | The event id, `evt_…`: the `id` of the delivered payload |
| `event_type` | The event type |
| `webhook_endpoint` | The endpoint id, or `null` when the endpoint was deleted or when an earlier delivery could not be matched to one endpoint |
| `endpoint` | The URL the delivery was sent to, or `null` when an earlier delivery never recorded one |
| `status` | `pending`, `retry` while another attempt is scheduled, `success` or `failure` |
| `response_status_code` | The HTTP status of the latest attempt, or `null` when it got no response |
| `created`, `updated` | Unix seconds |

### Retrieve a Delivery

```
GET /webhook-deliveries/{id}
```

Answers the delivery object plus `payload`, the event's envelope in its current format, and `attempts`, oldest first. A delivery made before the `account` and `created` keys existed went out without them and shows them here. An unknown id, or the id of a delivery of another account, answers `404` with the code `WEBHOOK_DELIVERY_NOT_FOUND`.

| Attempt field | Description |
| --- | --- |
| `attempt` | 1 for the first send |
| `created` | Unix seconds |
| `response_status_code` | The HTTP status, or `null` when no response arrived |
| `response_body` | What your server answered, or `null` when no response arrived |
| `response_body_truncated` | `true` when the stored text is not your server's complete answer: it was cut at 16 KiB, or reading it stopped early (size limit, time limit or a connection error). `false` only when the whole answer was read and nothing was cut. An empty `response_body` with `true` means nothing could be stored |
| `error` | `null` when a response arrived; otherwise `timeout`, `connection_refused`, `dns_failure`, `tls_error`, `blocked`, `connection_error`, or `not_sent` when ChaChing did not send the request |

Attempts are kept for 30 days. A delivery older than that, or one sent before attempts were recorded, answers `attempts: []`, with its payload and its final status. Deliveries and their payloads are kept.

**Do not return sensitive data in webhook responses.** ChaChing stores up to 16 KiB of each response body for 30 days and shows it in the API and the dashboard. It removes API keys, signing secrets, bearer tokens, JWTs and card numbers it recognizes before storing, but that does not cover every kind of sensitive value. Long digit sequences that look like card numbers, such as some numeric ids, are removed too. Answer with a status code and a short message.

Reading a delivery returns your customers' data as it was delivered, so these routes require the secret key, like every route in this section.

### Rotate the Signing Secret

```
POST /webhook-endpoints/{id}/rotate-secret
```

No request body. Creates a new signing secret and answers `200` with the Webhook Endpoint object plus `secret` (the new `whsec_…` secret, returned only in this response) and `previous_secret_expires_at` (Unix seconds, 24 hours after the rotation):

```json
{
  "id": "0b6f7a52-3c1e-4d8a-9f43-2a7d1c5e8b90",
  "object": "webhook_endpoint",
  "endpoint": "https://your-server.com/webhook",
  "enabled_events": ["customer.created", "invoice.payment_succeeded"],
  "status": "active",
  "signature_key": "endpoint_secret",
  "created": 1761091200,
  "updated": 1761177600,
  "secret": "whsec_…",
  "previous_secret_expires_at": 1761264000
}
```

If you lose a secret before you store it, rotate again to get a new one. The lost secret becomes the previous secret and keeps verifying until `previous_secret_expires_at`; if it may have been exposed, rotate once more and it stops verifying at once.

Until `previous_secret_expires_at`, every delivery carries two `v1` signatures: one for the new secret (first) and one for the previous secret. After it, only the new secret signs. At most two secrets are active: rotating again during those 24 hours ends the previous secret immediately, so rotating twice in a row revokes a secret you believe is compromised. Rotating works on `active` and `inactive` endpoints; changing the URL or the event types, enabling and disabling keep the secret.

### Path and Not Found Errors

| Status | `error` | When |
|--------|---------|------|
| `400` | no code | A path id is not `prefix_` plus letters and digits and not a UUID. |
| `404` | `WEBHOOK_ENDPOINT_NOT_FOUND` | The id is not one of your endpoints. |
| `404` | `WEBHOOK_DELIVERY_NOT_FOUND` | The id is not one of your deliveries. |

### Signing

Every delivery is signed as described in [Webhook Signatures](#webhook-signatures), with the endpoint's own signing secret. Endpoints that existed before signing secrets were issued one automatically; rotate the secret once to obtain it (see [Endpoints Created Before Signing Secrets](#endpoints-created-before-signing-secrets)).

---

# Webhook Signatures

ChaChing signs every webhook delivery with an HMAC-SHA256 signature. Verify it to confirm that the request came from ChaChing and that the payload was not changed.

## Signature Header

Each request includes a `Chaching-Signature` header:

```
Chaching-Signature: t=<unix_timestamp>,v1=<hex_hmac_sha256>
```

During the 24 hours after you rotate an endpoint's signing secret, the header carries two `v1` signatures, the one for the new secret first:

```
Chaching-Signature: t=<unix_timestamp>,v1=<signature_with_new_secret>,v1=<signature_with_previous_secret>
```

- `t` — Unix timestamp of when the request was sent. Every delivery attempt, retries included, is signed again with a new timestamp.
- `v1` — HMAC-SHA256 hex digest of the signed payload. There is one `v1` per active secret. Ignore every other scheme name.

## Signing Algorithm

```
HMAC-SHA256(signing_key, "{timestamp}.{raw_json_body}")
```

Where:
- `signing_key` is the endpoint's signing secret, the whole `whsec_…` string including the `whsec_` prefix.
- `timestamp` is the `t` value from the `Chaching-Signature` header
- `raw_json_body` is the **raw, unparsed** request body

> ⚠️ Always compute the signature against the **raw request body**, before any JSON parsing. Re-serializing a parsed object may alter whitespace or key ordering and cause signature verification to fail.

## Verifying a Delivery

1. Read `t` and every `v1` from `Chaching-Signature`.
2. Build `<t>.<raw body>`.
3. Compute HMAC-SHA256 with the whole `whsec_` value as key.
4. Compare in constant time with each `v1` and check `t` is within 5 minutes.

{% admonition type="warning" name="Use the raw body" %}
A JSON-parsing middleware that re-serializes the body breaks verification. Verify the bytes you received.
{% /admonition %}

## Verification Examples

### Node.js

```javascript
const crypto = require('crypto');

const TOLERANCE_SECONDS = 300;

function verifyWebhookSignature(rawBody, signatureHeader, signingSecret, nowSeconds = Math.floor(Date.now() / 1000)) {
  let timestamp = null;
  const signatures = [];
  for (const part of signatureHeader.split(',')) {
    const index = part.indexOf('=');
    if (index === -1) continue;
    const key = part.slice(0, index).trim();
    const value = part.slice(index + 1).trim();
    if (key === 't') timestamp = value;
    if (key === 'v1') signatures.push(value);
  }
  if (timestamp === null || !/^\d+$/.test(timestamp) || signatures.length === 0) return false;
  if (Math.abs(nowSeconds - Number(timestamp)) > TOLERANCE_SECONDS) return false;

  const expected = Buffer.from(
    crypto.createHmac('sha256', signingSecret).update(`${timestamp}.${rawBody}`).digest('hex'),
    'utf8'
  );
  return signatures.some(signature => {
    const received = Buffer.from(signature, 'utf8');
    return received.length === expected.length && crypto.timingSafeEqual(received, expected);
  });
}
```

### Python

```python
import hashlib
import hmac
import re
import time

TOLERANCE_SECONDS = 300

def verify_webhook_signature(raw_body: bytes, signature_header: str, signing_secret: str, now_seconds=None) -> bool:
    timestamp = None
    signatures = []
    for part in signature_header.split(','):
        key, separator, value = part.strip().partition('=')
        if not separator:
            continue
        if key == 't':
            timestamp = value
        elif key == 'v1':
            signatures.append(value)
    if timestamp is None or not re.fullmatch(r'[0-9]+', timestamp) or not signatures:
        return False
    now = int(time.time()) if now_seconds is None else now_seconds
    if abs(now - int(timestamp)) > TOLERANCE_SECONDS:
        return False
    signed_payload = timestamp.encode('utf-8') + b'.' + raw_body
    expected = hmac.new(signing_secret.encode('utf-8'), signed_payload, hashlib.sha256).hexdigest()
    return any(hmac.compare_digest(expected.encode('utf-8'), signature.encode('utf-8')) for signature in signatures)
```

> ℹ️ Check every `v1` value, not only the first or the last: during a rotation the header carries one signature per active secret, and the one that matches yours can be in either position.

## Endpoints Created Before Signing Secrets

Endpoints that existed before signing secrets were introduced were issued a signing secret automatically, and every delivery to them is signed with it. ChaChing never showed that secret, and your account secret API key no longer verifies any delivery. To obtain the secret:

1. Rotate the endpoint's signing secret (`POST /webhook-endpoints/{id}/rotate-secret`, or **Rotate signing secret** in the UI) and store the `whsec_…` secret.
2. Deploy the new secret to the server that verifies the deliveries. For 24 hours each delivery also carries a signature for the secret issued automatically; verify with the new secret, which matches one of the `v1` values.
3. A delivery that failed verification before you deployed the new secret is retried on the usual schedule, and every retry is signed again with the current secrets.

---

# Handling Webhooks (Client Side)

When receiving a webhook:

1. Accept HTTP POST request
2. Parse JSON payload
3. Read `event`
4. Process `data`
5. Return HTTP `200 OK`

---

# Receive on Staging

ChaChing delivers only to public URLs. On staging, expose a local server through a tunnel that gives it a public HTTPS URL, for example `cloudflared tunnel --url http://localhost:4242`, and register that URL. To catch up after downtime, list deliveries with `GET /webhook-deliveries` (attempts are kept 30 days); a platform reads its endpoints' deliveries with `GET /platform/webhook-endpoints/{id}/logs` and each payload with the account-level read. Events can arrive out of order: read the object's current state from the API before you act on an older event.

---

# Retry Behavior

ChaChing automatically retries failed webhook deliveries.

Retries are triggered when:

- Network failure
- Timeout
- HTTP `5xx` response
- HTTP `408` (Request Timeout)
- HTTP `429` (Too Many Requests)

ChaChing waits up to 15 seconds for your response; a slower response counts as a timeout and is retried. Redirects are not followed.

Retries are **not** triggered when the endpoint returns any other non-`2xx` status — every `3xx` redirect and every `4xx` other than `408` and `429`, for example `400`, `401`, `403`, `404` and `410`. Those statuses tell ChaChing the endpoint will never accept this payload, so the delivery status is set to `failure` after the first attempt and the returned status code is recorded in the delivery log.

**Retry Flow:**

- If a delivery fails with a retryable condition, the webhook is retried
- If **max retries NOT reached,** retry continues
- If **max retries reached,** the delivery status is set to `failure`
- If a delivery is **permanently rejected**, no retry is scheduled and the delivery status is set to `failure` right away
- A retry already scheduled is sent even if the endpoint's `enabled_events` changes in the meantime; a disabled or deleted endpoint receives no further retries

**Retry Policy:**

- Attempts: **5 retries** after the first delivery, for a maximum of 6 delivery attempts
- Retry window: **approximately 43 minutes** — the delays between attempts are approximately 15 seconds, 45 seconds, 2 minutes, 10 minutes and 30 minutes, each randomized by up to 20% in both directions

> ⚠️ Webhooks may be delivered multiple times. Ensure your system handles duplicates (use event `id`).
> 

---

# Webhook Delivery

- Method: `POST`
- Content-Type: `application/json`
- Sent to: configured endpoint URL in ChaChing application.

---

# Webhook Logs

ChaChing stores webhook delivery history.

Logs include:

- Event type
- Payload
- Status (`pending` / `retry` / `success` / `failure`)

> Logs reflect actual data sent from the database
> 

---

# Event Status

Webhook delivery has the following internal states:

- `pending` → waiting to be sent
- `retry` → retry in progress
- `success` → delivered successfully
- `failure` → the delivery was abandoned. Two situations produce it: the endpoint returned a **permanent rejection**, in which case no retry was attempted at all; and the endpoint kept failing with retryable conditions until every retry was exhausted. The delivery's attempts tell the two apart

---

# Event Types

ChaChing generates webhook events for key system entities. Each event represents a specific change or action performed in the system.

> Each endpoint receives only the event types in its `enabled_events` (see [Webhook Configuration](#webhook-configuration)). Listen only to the event types your integration needs.
> 

---

## Customer Events

- `customer.created` — A new customer is created
- `customer.updated` — An existing customer is updated
- `customer.deleted` — A customer is deleted

---

## Payment Method Events

- `payment_method.attached` — A payment method is attached to a customer, through the dashboard, the API or the customer portal. When the card is already saved for that customer, the event carries the saved payment method
- `payment_method.detached` — A payment method is detached from a customer

---

## Invoice Events

- `invoice.created` — A new invoice is created
- `invoice.deleted` — An invoice is deleted
- `invoice.finalized` — An invoice is finalized and ready for payment
- `invoice.sent` — An invoice is sent to the customer
- `invoice.payment_succeeded` — Invoice payment completed successfully
- `invoice.payment_failed` — Invoice payment failed

---

## Subscription Events

- `subscription.created` — A subscription is created
- `subscription.updated` — A subscription is updated
- `subscription.canceled` — A subscription is canceled
- `subscription.paused` — A subscription is paused
- `subscription.resumed` — A paused subscription is resumed

---

## Product Events

- `product.created` — A product is created
- `product.updated` — A product is updated
- `product.deleted` — A product is deleted

---

## Price Events

- `price.created` — A price is created
- `price.updated` — A price is updated
- `price.deleted` — A price is deleted

---

## Tax Events

- `tax.created` — A tax configuration is created
- `tax.updated` — A tax configuration is updated

---

## Payment Plan Events

- `payment_plan.created` — A payment plan is created
- `payment_plan.activated` — A payment plan starts running
- `payment_plan.installment_paid` — An installment of a payment plan is paid
- `payment_plan.installment_failed` — An automatic charge attempt on an installment is declined
- `payment_plan.past_due` — A payment plan changes to `past_due`
- `payment_plan.defaulted` — A payment plan changes to `defaulted`
- `payment_plan.reactivated` — A payment plan returns to `active` from `past_due` or `defaulted`
- `payment_plan.completed` — A payment plan is paid off
- `payment_plan.canceled` — A payment plan is canceled
- `payment_plan.payment_succeeded` — A payment is recorded on a payment plan

---

## Summary

ChaChing currently supports **34 webhook event types** across such entities as:

- Customers
- Payment Methods
- Invoices
- Subscriptions
- Products
- Prices
- Taxes
- Payment Plans

---

# Automatic Charges, Retries and Dunning

ChaChing generates webhook events for charges its billing engine initiates on its own — subscription renewals and their scheduled retries — in addition to charges started from the Dashboard, the hosted checkout page, and the REST API.

## Automatic Renewal Events

An automatic subscription renewal produces the following sequence of events:

1. `invoice.created` — the renewal invoice is created
2. `invoice.finalized` — the invoice is committed and ready for payment
3. `invoice.payment_succeeded` — the charge succeeds, **or** `invoice.payment_failed` — the charge is declined

---

## Retries and Failed-Payment Outcomes

A retry of an automatic charge that fails produces its own `invoice.payment_failed` event. The subscription events for the failed-payment outcomes configured in **Invoices & Subscriptions** settings are described in [Track subscription lifecycle and failed payments](./subscription-lifecycle.md). That page also documents the retry timeline, how the outcome day is counted, and when `next_payment_attempt` is present.

---

## Subscription Event Discriminators

`subscription.canceled`, `subscription.paused` and `subscription.resumed` carry two additional fields that distinguish who caused the transition:

- `reason` — `"dunning"` when the transition is driven by the billing engine, or `"requested"` when it is driven by an API call or a Dashboard action
- `initiated_by` — `"system"` for a dunning-driven transition, or `"merchant"` for an API- or Dashboard-driven one

Both fields are `null` on `subscription.created` and `subscription.updated`. `subscription.canceled`, `subscription.paused` and `subscription.resumed` are not sent for a subscription that belongs to a payment plan; follow the plan's `payment_plan.*` events instead.

---

## Deduplication

Every event is delivered at least once. Deduplicate on the envelope's `id` field, per endpoint, as described in [Important Notes for Integrators](#important-notes-for-integrators). Every endpoint that receives an event gets the same `id`.

---

# Event Structure

All ChaChing webhook events share a **common envelope structure**, with the actual entity data contained inside the `data` field.

---

## Common Event Envelope

Every webhook sent by ChaChing follows this structure:

```json
{
  "id": "evt_0123456789abcdef01234567",
  "account": {
    "id": "acc_3c1f0a9e2b7d4c58a1e6f204",
    "external_id": null,
    "company": {
      "id": "com_9d2e7b41c6a0f3e85b17d920",
      "external_id": null
    }
  },
  "created": 1773400500,
  "createdAt": "2026-03-13T11:15:00.000Z",
  "data": {
    // Entity-specific payload
  },
  "event": "customer.updated"
}
```

---

## Fields Description

- `id` — Unique event identifier. The same on every endpoint that receives the event and on every retry. Use it to deduplicate
- `account` — The merchant account the event belongs to: `id` (`acc_…`), `external_id`, and its `company` with `id` (`com_…`) and `external_id`. `external_id` is the value your platform sent as `external_id` when it created the account (`company.external_id` for the company); it is `null` for accounts created in the dashboard.
- `created` — When the event was generated, in Unix seconds
- `createdAt` — The same instant as an ISO 8601 string. Deprecated: use `created`
- `data` — **Entity payload** (varies depending on event type)
- `event` — Event type (see Event Types section)

> ❗ Important: `data` already contains the object. There is **no `data.object` nesting**.
> 

---

## Customer Events Payload

Events:

- `customer.created`
- `customer.updated`
- `customer.deleted`

---

### Schema

```json
{
  "id": "cus_XXXXXXXX",
  "object": "customer",
  "name": "string",
  "email": "string | null",
  "external_id": "string | null",
  "description": "string | null",
  "phone": "string | null",
  "created": 1672531200,
  "currency": "string",
  "address": {
    "line1": "string | null",
    "line2": "string | null",
    "city": "string | null",
    "state": "string | null",
    "postal_code": "string | null",
    "country": "string | null"
  },
  "shipping": {
    "name": "string",
    "phone": "string | null",
    "address": {
      "line1": "string | null",
      "line2": "string | null",
      "city": "string | null",
      "state": "string | null",
      "postal_code": "string | null",
      "country": "string | null"
    }
  }
}
```

`email` is `null` when the customer has no email address.

`external_id` is your own identifier for the customer, `null` when none was sent; see [External IDs](./payment-plans.md#external-ids).

---

### Example

```json
{
  "id": "evt_111",
  "account": {
    "id": "acc_3c1f0a9e2b7d4c58a1e6f204",
    "external_id": null,
    "company": {
      "id": "com_9d2e7b41c6a0f3e85b17d920",
      "external_id": null
    }
  },
  "created": 1773400500,
  "createdAt": "2026-03-13T11:15:00.000Z",
  "data": {
    "id": "cus_NffrFeUfNV2Hib",
    "object": "customer",
    "name": "John Doe",
    "email": "john.doe@example.com",
    "external_id": "PAT-799601",
    "description": "My First Test Customer",
    "phone": null,
    "created": 1680893993,
    "currency": "usd",
    "address": {
      "line1": "123 Market St",
      "line2": "Apt 4B",
      "city": "San Francisco",
      "state": "CA",
      "postal_code": "94103",
      "country": "US"
    },
    "shipping": {
      "name": "John Doe",
      "phone": "+1 (555) 123-4567",
      "address": {
        "line1": "123 Market St",
        "line2": "Apt 4B",
        "city": "San Francisco",
        "state": "CA",
        "postal_code": "94103",
        "country": "US"
      }
    }
  },
  "event": "customer.created"
}
```

---

### Notes

- `address` and `shipping.address` are single objects whose parts are `null` when empty; `phone` and `description` are `null` when empty.
- Timestamps are in Unix format (seconds)

---

## Payment Method Events Payload

Events:

- `payment_method.attached`
- `payment_method.detached`

### Schema

```json
{
  "id": "pm_XXXXXXXX",
  "card": {
    "brand": "string | null",
    "last4": "string | null",
    "exp_month": "number | null",
    "exp_year": "number | null",
    "wallet_type": "apple_pay | google_pay | null"
  },
  "customer": "cus_XXXXXXXX | null"
}
```

### Example

<!-- captured: GET /webhook-deliveries/{id} on stage, redacted -->
```json
CAPTURE PaymentMethodPublicDto
```

### Notes

- `data` is the Payment Method object: `id`, `card` and `customer`. Every key of `card` can be `null`, and so can `customer`.

---

## Invoice Events Payload

Events:

- `invoice.created`
- `invoice.deleted`
- `invoice.finalized`
- `invoice.sent`
- `invoice.payment_succeeded`
- `invoice.payment_failed`

### Schema

```json
{
  "id": "string",
  "object": "invoice",
  "account_country": "string",
  "account_name": "string",
  "amount_due": 0,
  "amount_paid": 0,
  "surcharge_amount": 0,
  "amount_charged": 0,
  "amount_remaining": 0,
  "attempt_count": 0,
  "next_payment_attempt": 1672531200,
  "attempted": true,
  "billing_reason": "string",
  "collection_method": "string",
  "created": 1672531200,
  "currency": "string",
  "customer": "cus_XXXXXXXX",
  "customer_external_id": "string | null",
  "customer_address": {
    "city": "string",
    "country": "string",
    "line1": "string",
    "line2": "string",
    "postal_code": "string",
    "state": "string"
  },
  "customer_email": "string",
  "customer_name": "string",
  "customer_phone": "string",
  "due_date": 1672531200,
  "failure_code": "string | null",
  "failure_message": "string | null",
  "hosted_invoice_url": "string | null",
  "lines": {
    "object": "list",
    "data": [],
    "total_count": 0
  },
  "number": "string",
  "paid": true,
  "period_end": 1672531200,
  "period_start": 1672531200,
  "status": "draft | open | paid | void",
  "status_transitions": {
    "finalized_at": 1672531200,
    "paid_at": 1672531200,
    "voided_at": 1672531200
  },
  "subscription": "sub_XXXXXXXX | null",
  "payment_plan": "pp_XXXXXXXX | null",
  "payment_plan_external_id": "string | null",
  "installment": "ppi_XXXXXXXX | null",
  "payment_plan_charge": "object | null",
  "subtotal": 0,
  "subtotal_excluding_tax": 0,
  "tax": 0,
  "total": 0,
  "total_excluding_tax": 0,
  "total_taxes": [
    {
      "amount": 0,
      "tax_behavior": "inclusive | exclusive",
      "tax_rate": "txr_XXXXXXXX",
      "taxable_amount": 0
    }
  ]
}
```

`surcharge_amount` and `amount_charged` sum the invoice's successful payments, in cents: the credit card surcharge added to them and the total charged to the card. `amount_charged` equals `amount_paid` when no surcharge was added. Every credit card payment of an account that charges through iPOSpays (Dejavoo) can carry a surcharge, and debit card payments carry none; see [Credit Card Surcharges](./card-surcharges.md).

### Example

```json
{
  "id": "evt_456",
  "account": {
    "id": "acc_3c1f0a9e2b7d4c58a1e6f204",
    "external_id": null,
    "company": {
      "id": "com_9d2e7b41c6a0f3e85b17d920",
      "external_id": null
    }
  },
  "created": 1773400500,
  "createdAt": "2026-03-13T11:15:00.000Z",
  "data": {
    "id": "a80baa67-1341-4184-b712-b6f9b85b5a2b",
    "object": "invoice",
    "account_country": "US",
    "account_name": "Acme Corp",
    "amount_due": 1500,
    "amount_paid": 500,
    "surcharge_amount": 0,
    "amount_charged": 500,
    "amount_remaining": 1000,
    "attempt_count": 1,
    "next_payment_attempt": null,
    "attempted": true,
    "billing_reason": "subscription",
    "collection_method": "charge_automatically",
    "created": 1680645568,
    "currency": "USD",
    "customer": "cus_383fda0065594749023e8c8d",
    "customer_external_id": "PAT-799601",
    "customer_address": {
      "city": "San Francisco",
      "country": "US",
      "line1": "123 Market St",
      "line2": "Apt 4B",
      "postal_code": "94103",
      "state": "CA"
    },
    "customer_email": "john.doe@example.com",
    "customer_name": "John Doe",
    "customer_phone": "+1234567890",
    "due_date": 1680645568,
    "failure_code": null,
    "failure_message": null,
    "hosted_invoice_url": "https://acme.pay.example.com/checkout/a80baa67-1341-4184-b712-b6f9b85b5a2b",
    "lines": {
      "object": "list",
      "data": [],
      "total_count": 0
    },
    "number": "INV-1001",
    "paid": true,
    "period_end": 1680645568,
    "period_start": 1680645568,
    "status": "paid",
    "status_transitions": {
      "finalized_at": 1680645568,
      "paid_at": 1680645568,
      "voided_at": null
    },
    "subscription": "sub_Wm45f5PGQvv67azFeH1z7PO3",
    "payment_plan": null,
    "payment_plan_external_id": null,
    "installment": null,
    "payment_plan_charge": null,
    "subtotal": 1500,
    "subtotal_excluding_tax": 1500,
    "tax": 150,
    "total": 1650,
    "total_excluding_tax": 1500,
    "total_taxes": [
      {
        "amount": 150,
        "tax_behavior": "exclusive",
        "tax_rate": "txr_1S2tlWDZYOQH9uHkltYEi1ap",
        "taxable_amount": 1500
      }
    ]
  },
  "event": "invoice.payment_succeeded"
}
```

### Notes

- The `status` field reflects the current lifecycle state of the invoice
- `status_transitions` shows timestamps for key lifecycle changes
- Monetary values are represented in the smallest currency unit (e.g., cents)
- The `lines` field contains invoice line items (structure may vary)
- Some fields may be `null` depending on invoice state and configuration
- `subscription`, `tax` and `total_taxes` are `null` when they do not apply.
- `attempt_count` is present on every invoice event. `next_payment_attempt` is present only on `invoice.payment_succeeded` and `invoice.payment_failed` for an automatic charge; see [Track subscription lifecycle and failed payments](./subscription-lifecycle.md) for when it is present and what `null` means
- `invoice.payment_succeeded` for a payment made through ChaChing (the dashboard, the payment page or an API request that pays the invoice) describes the invoice after that payment: its `amount_paid`, `surcharge_amount`, `amount_charged`, `status` and `attempt_count` include it
- `next_payment_attempt` is a Unix timestamp in seconds, matching every other timestamp in this payload
- `payment_plan`, `installment` and `payment_plan_charge` name the payment plan, the installment and the plan charge an invoice was raised for. All three are `null` on an invoice that belongs to no payment plan, and an invoice raised for a payment plan reports `subscription: null`. `payment_plan_charge` is `{ id, installment, sequence, payment, kind }`, with `kind` one of `down_payment`, `deposit`, `installment`, `extra_payment`, `payoff`. See [Payment Plans](./payment-plans.md). `payment_plan_external_id` is the plan's `external_id`, `null` when the invoice belongs to no plan or the plan has none.
- `due_date` is the invoice due date in Unix seconds. `hosted_invoice_url` is the permanent link to the invoice's payment page, the same URL `GET /invoices/{id}/hosted-link` returns. `failure_code` and `failure_message` describe the decline of the invoice's most recent payment attempt, and are `null` when that attempt was not declined or the invoice has no payment attempt
- `status` is one of `draft`, `open`, `paid` or `void`, in lowercase
- `collection_method` is `send_invoice` for an invoice created that way, and `charge_automatically` for every other invoice, including an invoice created with `send_invoice` before this value was recorded. The first webhook of a `send_invoice` invoice already carries it
- `created` is the instant the invoice was created. `status_transitions.finalized_at` is the instant it was first finalized, and `null` while it is a draft
- On an invoice of a payment plan, each entry of `lines.data` has `plan`, `price` and `subscription` set to `null`, `amount` in cents and `description` set to the text the payment page shows, such as `Knee replacement — Payment 2 of 12`

---

## Subscription Events Payload

Events:

- `subscription.created`
- `subscription.updated`
- `subscription.canceled`
- `subscription.paused`
- `subscription.resumed`

---

### Schema

```json
{
  "id": "sub_XXXXXXXX",
  "object": "subscription",
  "billing_cycle_anchor": 1672531200,
  "cancel_at": 1672531200,
  "cancel_at_period_end": false,
  "canceled_at": 1672531200,
  "collection_method": "string",
  "created": 1672531200,
  "currency": "string",
  "customer": "cus_XXXXXXXX",
  "reason": "dunning | requested | null",
  "initiated_by": "system | merchant | null",
  "ended_at": 1672531200,
  "items": {
    "object": "list",
    "data": [
      []
    ],
    "total_count": 0
  },
  "latest_invoice": "string",
  "start_date": 1672531200,
  "status": "active | trial | scheduled | paused | unpaid | cancelled | expired",
  "trial_end": 1672531200,
  "trial_start": 1672531200
}
```

---

### Example

```json
{
  "id": "evt_789",
  "account": {
    "id": "acc_3c1f0a9e2b7d4c58a1e6f204",
    "external_id": null,
    "company": {
      "id": "com_9d2e7b41c6a0f3e85b17d920",
      "external_id": null
    }
  },
  "created": 1773400500,
  "createdAt": "2026-03-13T11:15:00.000Z",
  "data": {
    "id": "sub_2FdhgypnsJQn9mZlY5qGaElL",
    "object": "subscription",
    "billing_cycle_anchor": 1679609767,
    "cancel_at": null,
    "cancel_at_period_end": false,
    "canceled_at": null,
    "collection_method": "charge_automatically",
    "created": 1679609767,
    "currency": "USD",
    "customer": "cus_739c07713fd4b68565a969cd",
    "reason": null,
    "initiated_by": null,
    "ended_at": null,
    "items": {
      "object": "list",
      "data": [
        []
      ],
      "total_count": 1
    },
    "latest_invoice": "58d0f1c9-ca00-4998-8e89-d74a290b4df6",
    "start_date": 1679609767,
    "status": "active",
    "trial_end": null,
    "trial_start": null
  },
  "event": "subscription.created"
}
```

The following example shows a `subscription.canceled` event driven by the billing engine's dunning process, carrying `reason` and `initiated_by`:

```json
{
  "id": "evt_790",
  "account": {
    "id": "acc_3c1f0a9e2b7d4c58a1e6f204",
    "external_id": null,
    "company": {
      "id": "com_9d2e7b41c6a0f3e85b17d920",
      "external_id": null
    }
  },
  "created": 1773400500,
  "createdAt": "2026-03-13T11:15:00.000Z",
  "data": {
    "id": "sub_2FdhgypnsJQn9mZlY5qGaElL",
    "object": "subscription",
    "billing_cycle_anchor": 1679609767,
    "cancel_at": 1680645568,
    "cancel_at_period_end": false,
    "canceled_at": 1680645568,
    "collection_method": "charge_automatically",
    "created": 1679609767,
    "currency": "USD",
    "customer": "cus_739c07713fd4b68565a969cd",
    "reason": "dunning",
    "initiated_by": "system",
    "ended_at": 1680645568,
    "items": {
      "object": "list",
      "data": [
        []
      ],
      "total_count": 1
    },
    "latest_invoice": "58d0f1c9-ca00-4998-8e89-d74a290b4df6",
    "start_date": 1679609767,
    "status": "cancelled",
    "trial_end": null,
    "trial_start": null
  },
  "event": "subscription.canceled"
}
```

An entry of `items.data`:

<!-- captured: GET /subscriptions/{id} on stage, redacted -->
```json
CAPTURE SubscriptionItem @ /items/data/0
```

---

### Notes

- The `status` field is one of `active`, `trial`, `scheduled`, `paused`, `unpaid`, `cancelled`, or `expired`; see [Track subscription lifecycle and failed payments](./subscription-lifecycle.md). The event name `subscription.canceled` is spelled with one `l`, and the status value `cancelled` with two
- Fields like `cancel_at`, `canceled_at`, and `ended_at` may be `null` depending on lifecycle state
- `items.data` contains subscription items (structure depends on pricing configuration)
- `latest_invoice` links the most recent invoice associated with the subscription
- Timestamps are in Unix format (seconds)
- `initiated_by` and `reason` carry values on `subscription.canceled`, `subscription.paused` and `subscription.resumed`, and are `null` otherwise.
- `collection_method` is `charge_automatically`.
- `latest_invoice` is `''` when the subscription has no invoice.
- `subscription.canceled`, `subscription.paused` and `subscription.resumed` are not sent for a subscription that belongs to a payment plan; follow the plan's `payment_plan.*` events instead.
- The two fields always travel together: `"dunning"` is always paired with `"system"`, and `"requested"` is always paired with `"merchant"`
- `items.data[].price.type` is `one_time`, `recurring` or `null`: it is `null` for an item whose price cannot be resolved. Handle `null` wherever you branch on the price type

---

## Product Events Payload

Events:

- `product.created`
- `product.updated`
- `product.deleted`

### Schema

```json
{
  "id": "prod_XXXXXXXX",
  "object": "product",
  "active": true,
  "created": 1672531200,
  "default_price": "price_XXXXXXXX",
  "description": "string | null",
  "name": "string",
  "tax_code": "string",
  "updated": 1672531200
}
```

### Example

```json
{
  "id": "evt_321",
  "account": {
    "id": "acc_3c1f0a9e2b7d4c58a1e6f204",
    "external_id": null,
    "company": {
      "id": "com_9d2e7b41c6a0f3e85b17d920",
      "external_id": null
    }
  },
  "created": 1773400500,
  "createdAt": "2026-03-13T11:15:00.000Z",
  "data": {
    "id": "prod_xT4LfvxLMQb4s9uFNIbXf9pm",
    "object": "product",
    "active": true,
    "created": 1760018761,
    "default_price": "price_1SGKTmDZYOQH9uHk1vaSxPKa",
    "description": null,
    "name": "Gold Plan",
    "tax_code": "txcd_10000000",
    "updated": 1753181495
  },
  "event": "product.updated"
}
```

### Notes

- The `default_price` field references the default price associated with the product
- The `description` field may be `null` if not provided
- The `active` field indicates whether the product is available for use
- Timestamps are in Unix format (seconds)

---

## Price Events Payload

Events:

- `price.created`
- `price.updated`
- `price.deleted`

### Schema

```json
{
  "id": "price_XXXXXXXX",
  "object": "price",
  "active": true,
  "created": 1672531200,
  "currency": "string",
  "description": "string | null",
  "product": "prod_XXXXXXXX | null",
  "recurring": {
    "interval": "day | week | month | year",
    "interval_count": 1
  } | null,
  "tax_behavior": "inclusive | exclusive",
  "type": "one_time | recurring",
  "unit_amount": 0,
  "unit_amount_decimal": "string"
}
```

### Example

```json
{
  "id": "evt_654",
  "account": {
    "id": "acc_3c1f0a9e2b7d4c58a1e6f204",
    "external_id": null,
    "company": {
      "id": "com_9d2e7b41c6a0f3e85b17d920",
      "external_id": null
    }
  },
  "created": 1773400500,
  "createdAt": "2026-03-13T11:15:00.000Z",
  "data": {
    "id": "price_1SGKTmDZYOQH9uHk1vaSxPKa",
    "object": "price",
    "active": true,
    "created": 1760018761,
    "currency": "USD",
    "description": "Premium Plan - Monthly",
    "product": "prod_xT4LfvxLMQb4s9uFNIbXf9pm",
    "recurring": {
      "interval": "month",
      "interval_count": 1
    },
    "tax_behavior": "exclusive",
    "type": "recurring",
    "unit_amount": 1000,
    "unit_amount_decimal": "1000"
  },
  "event": "price.created"
}
```

---

### Notes

- The `type` field defines whether the price is **one-time** or **recurring**
- The `recurring` object is `null` when `type` is `one_time`, and a `{ interval, interval_count }` object when `type` is `recurring`.
- `unit_amount` is represented in the smallest currency unit (e.g., cents)
- `product` may be `null` if not linked
- `unit_amount_decimal` provides a precise representation of the amount
- `tax_behavior` defines how taxes are applied (inclusive or exclusive)

---

## Tax Events Payload

Events:

- `tax.created`
- `tax.updated`

### Schema

```json
{
  "id": "txr_XXXXXXXX",
  "object": "tax_rate",
  "active": true,
  "country": "string",
  "created": 1672531200,
  "description": "string | null",
  "display_name": "string",
  "inclusive": true,
  "percentage": 0,
  "tax_type": "string"
}
```

### Example

```json
{
  "id": "evt_987",
  "account": {
    "id": "acc_3c1f0a9e2b7d4c58a1e6f204",
    "external_id": null,
    "company": {
      "id": "com_9d2e7b41c6a0f3e85b17d920",
      "external_id": null
    }
  },
  "created": 1773400500,
  "createdAt": "2026-03-13T11:15:00.000Z",
  "data": {
    "id": "txr_inQ4hbLZPyljDXFRCEnjtEqK",
    "object": "tax_rate",
    "active": true,
    "country": "US",
    "created": 1678833149,
    "description": "State sales tax",
    "display_name": "Sales Tax",
    "inclusive": true,
    "percentage": 5,
    "tax_type": "Custom"
  },
  "event": "tax.created"
}
```

---

### Notes

- `percentage` is a number, such as `7.25` for 7.25 %
- The `inclusive` field indicates whether the tax is included in the price
- `tax_type` defines the type of tax configuration (e.g., custom)
- The `description` field may be `null` depending on configuration

---

## Payment Plan Events Payload

Events:

- `payment_plan.created`
- `payment_plan.activated`
- `payment_plan.past_due`
- `payment_plan.defaulted`
- `payment_plan.reactivated`
- `payment_plan.completed`
- `payment_plan.canceled`

carry the Payment Plan object.

- `payment_plan.installment_paid`
- `payment_plan.installment_failed`

carry the Installment object.

- `payment_plan.payment_succeeded`

carries the Payment object.

---

### Schema — Payment Plan

| Field | Type | Nullable |
| --- | --- | --- |
| `id` | string | no |
| `object` | string | no |
| `external_id` | string | yes |
| `customer` | string | no |
| `customer_external_id` | string | yes |
| `description` | string | yes |
| `status` | string, one of `awaiting_deposit`, `active`, `past_due`, `defaulted`, `paid_off`, `canceled` | no |
| `currency` | string | no |
| `total_amount` | integer | no |
| `down_payment_amount` | integer | no |
| `installment_amount` | integer | no |
| `installment_count` | integer | no |
| `interval` | string, one of `daily`, `weekly`, `biweekly`, `monthly`, `annual` | no |
| `start_date` | integer | no |
| `remainder_placement` | string, one of `first`, `last` | no |
| `remaining_balance` | integer | no |
| `installments_paid` | integer | no |
| `installments_remaining` | integer | no |
| `next_due_date` | integer | yes |
| `payment_method` | string | yes |
| `gateway` | string | yes |
| `reference` | string | yes |
| `deposit` | object | yes |
| `deposit.reference` | string | no |
| `deposit.kind` | string, one of `down_payment`, `installment_1` | no |
| `deposit.amount` | integer | no |
| `deposit.status` | string, one of `pending`, `collected` | no |
| `deposit.transaction_reference` | string | yes |
| `deposit.collected_at` | integer | yes |
| `deposit.attempt` | object | yes |
| `deposit.attempt.id` | string | no |
| `deposit.attempt.object` | string | no |
| `deposit.attempt.terminal` | string | no |
| `deposit.attempt.reference_id` | string | no |
| `deposit.attempt.state` | string, one of `in_flight`, `resolving`, `approved`, `approved_no_credential`, `declined`, `cancelled`, `not_charged`, `failed`, `partial_approval`, `force_closed` | no |
| `deposit.attempt.started` | integer | no |
| `deposit.attempt.resolved` | integer | yes |
| `deposit.attempt.decline_code` | string | yes |
| `deposit.attempt.decline_message` | string | yes |
| `deposit.attempt.surcharge_amount` | integer | no |
| `deposit.attempt.amount_charged` | integer | no |
| `created` | integer | no |
| `ended_at` | integer | yes |
| `canceled_at` | integer | yes |

### Schema — Installment

| Field | Type | Nullable |
| --- | --- | --- |
| `id` | string | no |
| `object` | string | no |
| `payment_plan` | string | no |
| `payment_plan_external_id` | string | yes |
| `customer` | string | no |
| `customer_external_id` | string | yes |
| `sequence` | integer | no |
| `due_date` | integer | no |
| `amount` | integer | no |
| `amount_paid` | integer | no |
| `surcharge_amount` | integer | no |
| `amount_charged` | integer | no |
| `surcharge_status` | string, one of `final`, `unavailable` | no |
| `status` | string, one of `scheduled`, `paid`, `failed`, `canceled` | no |
| `attempt_count` | integer | no |
| `next_payment_attempt` | integer | yes |
| `paid_at` | integer | yes |
| `first_failed_at` | integer | yes |
| `failure_code` | string | yes |
| `failure_message` | string | yes |
| `card` | object | yes |
| `invoice` | string | yes |
| `hosted_invoice_url` | string | yes |
| `payment_plan_summary` | object | no |

### Schema — Payment

| Field | Type | Nullable |
| --- | --- | --- |
| `id` | string | no |
| `object` | string | no |
| `payment_plan` | string | no |
| `payment_plan_external_id` | string | yes |
| `customer` | string | no |
| `customer_external_id` | string | yes |
| `status` | string | no |
| `amount` | integer | no |
| `surcharge_amount` | integer | no |
| `amount_charged` | integer | no |
| `surcharge_status` | string, one of `final`, `unavailable` | no |
| `external` | boolean | no |
| `payment_method` | string | yes |
| `card` | object | yes |
| `invoice` | string | yes |
| `hosted_invoice_url` | string | yes |
| `payoff` | boolean | no |
| `created` | integer | no |
| `payment_plan_status` | string, one of `awaiting_deposit`, `active`, `past_due`, `defaulted`, `paid_off`, `canceled` | no |
| `remaining_balance` | integer | no |
| `installments_remaining` | integer | no |
| `allocations` | array | no |
| `payment_plan_summary` | object | no |
| `terminal` | object | yes |

`status` is one of the six values listed under [Record Payments](./payment-plans.md#record-payments-extra-payments-and-a-payoff).

The `terminal` object the event emits carries all 17 keys of the terminal object described in [Take a Payment on a Terminal](./payment-plans-terminal.md#take-a-payment-on-a-terminal), and its `state` is `approved` or `partial_approval` at emission.

`card` is `{ "brand": "string | null", "last4": "string | null" }`, or `null` when the card cannot be read. `card.brand` is the gateway's card network label as stored; it is not guaranteed to be lowercase, so compare it case-insensitively.

`first_failed_at` is the time ChaChing recorded the first confirmed decline of the installment, in Unix seconds. It is `null` for an installment whose only declines predate this field, and it does not change on later retries.

### Schema — Payment Allocation

`allocations` lists the installment and principal amounts a payment paid, in the order they were applied. It is `[]` for a payment whose `status` is not `applied` or `partially_applied`.

| Field | Type | Nullable |
| --- | --- | --- |
| `installment` | string | yes |
| `sequence` | integer | yes |
| `kind` | string, one of `installment`, `principal` | no |
| `amount` | integer | no |
| `surcharge_amount` | integer | no |
| `amount_charged` | integer | no |
| `surcharge_status` | string, one of `final`, `unavailable` | no |
| `invoice` | string | yes |
| `hosted_invoice_url` | string | yes |

### Schema — Plan Summary

`payment_plan_summary` carries the same values as the same-named keys of the Payment Plan object.

| Field | Type | Nullable |
| --- | --- | --- |
| `installment_count` | integer | no |
| `currency` | string | no |
| `description` | string | yes |
| `remaining_balance` | integer | no |
| `installments_paid` | integer | no |
| `installments_remaining` | integer | no |
| `next_due_date` | integer | yes |
| `status` | string, one of `awaiting_deposit`, `active`, `past_due`, `defaulted`, `paid_off`, `canceled` | no |

### Example

`payment_plan.installment_failed`, with a retry scheduled:

```json
{
  "id": "evt_a52d1c2eb30833c6e7b3ed21",
  "account": {
    "id": "acc_3c1f0a9e2b7d4c58a1e6f204",
    "external_id": null,
    "company": {
      "id": "com_9d2e7b41c6a0f3e85b17d920",
      "external_id": null
    }
  },
  "created": 1763110812,
  "createdAt": "2025-11-14T09:00:12.000Z",
  "data": {
    "id": "ppi_9d078490b235b24e80eb92cc",
    "object": "payment_plan_installment",
    "payment_plan": "pp_5c1bb9d8d74f27c18e4a3629",
    "payment_plan_external_id": "PLAN-2026-0117",
    "customer": "cus_739c07713fd4b68565a969cd",
    "customer_external_id": "PAT-799601",
    "sequence": 2,
    "due_date": 1763078400,
    "amount": 20000,
    "amount_paid": 0,
    "surcharge_amount": 0,
    "amount_charged": 0,
    "surcharge_status": "final",
    "status": "failed",
    "attempt_count": 1,
    "next_payment_attempt": 1763164800,
    "paid_at": null,
    "first_failed_at": 1763078412,
    "failure_code": "05",
    "failure_message": "Do Not Honor",
    "card": {
      "brand": "visa",
      "last4": "4242"
    },
    "invoice": "c1f0d9aa-2b44-4d19-9d02-77b2aa4e1c30",
    "hosted_invoice_url": "https://acme.pay.example.com/checkout/c1f0d9aa-2b44-4d19-9d02-77b2aa4e1c30",
    "payment_plan_summary": {
      "installment_count": 9,
      "currency": "USD",
      "description": "Orthodontic treatment, upper and lower braces",
      "remaining_balance": 160000,
      "installments_paid": 1,
      "installments_remaining": 8,
      "next_due_date": 1763078400,
      "status": "past_due"
    }
  },
  "event": "payment_plan.installment_failed"
}
```

The values shown are illustrative.

`payment_plan.completed`:

```json
{
  "id": "evt_3646c8c63a087e5b5770b2f2",
  "account": {
    "id": "acc_3c1f0a9e2b7d4c58a1e6f204",
    "external_id": null,
    "company": {
      "id": "com_9d2e7b41c6a0f3e85b17d920",
      "external_id": null
    }
  },
  "created": 1781395651,
  "createdAt": "2026-06-14T00:07:31.000Z",
  "data": {
    "id": "pp_5c1bb9d8d74f27c18e4a3629",
    "object": "payment_plan",
    "external_id": "PLAN-2026-0117",
    "customer": "cus_739c07713fd4b68565a969cd",
    "customer_external_id": "PAT-799601",
    "description": "Orthodontic treatment, upper and lower braces",
    "status": "paid_off",
    "currency": "USD",
    "total_amount": 210000,
    "down_payment_amount": 30000,
    "installment_amount": 20000,
    "installment_count": 9,
    "interval": "monthly",
    "start_date": 1760400000,
    "remainder_placement": "first",
    "remaining_balance": 0,
    "installments_paid": 9,
    "installments_remaining": 0,
    "next_due_date": null,
    "payment_method": "9f3a1c2e-7b45-4d18-a6c0-2f8e5b7d1a94",
    "gateway": "dejavoo",
    "reference": null,
    "deposit": null,
    "created": 1757894400,
    "ended_at": 1781395651,
    "canceled_at": null
  },
  "event": "payment_plan.completed"
}
```

`payment_plan.payment_succeeded` for a payment that reached the principal:

```json
{
  "id": "evt_fb2db40ae8a86ce10424edfe",
  "account": {
    "id": "acc_3c1f0a9e2b7d4c58a1e6f204",
    "external_id": null,
    "company": {
      "id": "com_9d2e7b41c6a0f3e85b17d920",
      "external_id": null
    }
  },
  "created": 1763483200,
  "createdAt": "2025-11-18T16:26:40.000Z",
  "data": {
    "id": "ppp_a6766968bdcb187da402e727",
    "object": "payment_plan_payment",
    "payment_plan": "pp_5c1bb9d8d74f27c18e4a3629",
    "payment_plan_external_id": "PLAN-2026-0117",
    "customer": "cus_739c07713fd4b68565a969cd",
    "customer_external_id": "PAT-799601",
    "status": "applied",
    "amount": 50000,
    "surcharge_amount": 0,
    "amount_charged": 50000,
    "surcharge_status": "final",
    "external": false,
    "payment_method": "9f3a1c2e-7b45-4d18-a6c0-2f8e5b7d1a94",
    "card": {
      "brand": "visa",
      "last4": "4242"
    },
    "invoice": "d5b9e2c1-8a41-4f36-9c17-3ad0b1e5f882",
    "hosted_invoice_url": "https://acme.pay.example.com/checkout/d5b9e2c1-8a41-4f36-9c17-3ad0b1e5f882",
    "payoff": false,
    "created": 1763483200,
    "payment_plan_status": "active",
    "remaining_balance": 130000,
    "installments_remaining": 7,
    "allocations": [
      {
        "installment": "ppi_9d078490b235b24e80eb92cc",
        "sequence": 2,
        "kind": "installment",
        "amount": 20000,
        "surcharge_amount": 0,
        "amount_charged": 20000,
        "surcharge_status": "final",
        "invoice": "c1f0d9aa-2b44-4d19-9d02-77b2aa4e1c30",
        "hosted_invoice_url": "https://acme.pay.example.com/checkout/c1f0d9aa-2b44-4d19-9d02-77b2aa4e1c30"
      },
      {
        "installment": null,
        "sequence": null,
        "kind": "principal",
        "amount": 30000,
        "surcharge_amount": 0,
        "amount_charged": 30000,
        "surcharge_status": "final",
        "invoice": "d5b9e2c1-8a41-4f36-9c17-3ad0b1e5f882",
        "hosted_invoice_url": "https://acme.pay.example.com/checkout/d5b9e2c1-8a41-4f36-9c17-3ad0b1e5f882"
      }
    ],
    "payment_plan_summary": {
      "installment_count": 9,
      "currency": "USD",
      "description": "Orthodontic treatment, upper and lower braces",
      "remaining_balance": 130000,
      "installments_paid": 2,
      "installments_remaining": 7,
      "next_due_date": 1765670400,
      "status": "active"
    },
    "terminal": null
  },
  "event": "payment_plan.payment_succeeded"
}
```

---

### Notes

- The Payment Plan object in `data` is the plan after the change, and its `status` is the plan's status when the event is sent. A plan created with a payment method sends `payment_plan.created` and `payment_plan.activated` with the same object, `status: "active"`
- `payment_plan.installment_failed` is sent once per declined automatic charge attempt, retries included. A declined payment through `POST /payment-plans/{id}/payments` answers the request with an error instead and sends no `payment_plan.installment_failed`. `attempt_count` counts the attempts made on the installment, and `next_payment_attempt` is the date of the next retry, or `null` when no retry is scheduled
- `payment_plan.installment_paid` is sent once per installment. The down payment sends it with `sequence: 0`, both when it is charged at creation and when it is collected on a terminal
- `deposit` is `null` for a plan created with a payment method. For a plan created without one, `deposit.kind` is `down_payment` when the plan has a down payment and `installment_1` otherwise, `deposit.amount` is the amount the terminal collects, and `deposit.attempt` is `null` until the first terminal attempt and then holds the latest attempt; see [Collect the First Installment on a Terminal](./payment-plans-terminal.md#collect-the-first-installment-on-a-terminal)
- `payment_plan.completed` is the single paid-in-full event. It is sent when the schedule ends with every installment paid, when a payoff payment is recorded, and when the last unpaid installment is paid through the hosted invoice page, the dashboard or `PUT /invoices/{id}`, settled by credit, or collected by the automatic charge or a retry after the schedule ended; `status` is `paid_off` in every case. It is never sent while an installment is still unpaid
- `payment_plan.payment_succeeded` is sent for every payment `POST /payment-plans/{id}/payments` records — installment payments, principal payments, payoffs and external payments alike — and never for a declined or refused one. Its `data` is the Payment object at the moment the payment is recorded: `id`, `amount`, `external`, `payment_method`, `invoice` and `payoff` match the endpoint's response (except for a payment taken on a terminal, whose `201` is `processing` with `amount` `0`: the event carries the final values), while for a payment that reached the principal, the event is sent after the plan's remaining schedule is rebuilt, and `remaining_balance`, `installments_remaining`, `payment_plan_status` and `payment_plan_summary` describe the plan after it. A repeated request with the same `Idempotency-Key` sends nothing again. When a request ends before its payment is fully recorded, ChaChing finishes recording it later and sends the payment's events then. For one payment they are sent in this order: `payment_plan.installment_paid`, then `payment_plan.payment_succeeded`, then `payment_plan.completed`; a delivery that is retried can arrive after a later event. This event is sent only with `status` `applied` or `partially_applied`.
- `surcharge_amount` and `amount_charged` on the Installment and Payment objects report the credit card surcharge and the total charged to the card, in cents. While ChaChing cannot read the surcharge from the gateway yet, they read `0` and the amount paid, and `payment_plan.installment_paid` and `payment_plan.payment_succeeded` wait until the surcharge is read, for up to an hour; after an hour they are sent with `surcharge_status` `unavailable`. When ChaChing cannot read the surcharge of a charge (after retrying for up to 7 days, or at once when the charge cannot be matched), the objects keep `surcharge_amount` at `0` and `amount_charged` equal to the amount paid, and that does not change later; the exact amount charged to the card is then the one shown for that transaction in your Dejavoo (iPOSpays) account. See [Credit Card Surcharges](./payment-plans.md#credit-card-surcharges)
- The principal part of a payment has its own invoice: when the principal is recorded, ChaChing sends `invoice.payment_succeeded` for it (and `invoice.payment_failed` when its card charge is declined), with `payment_plan_charge.kind` `extra_payment` or `payoff`. `payment_plan.payment_succeeded` reports the same money, so count each payment once. A principal recorded without an invoice (account credit, automatic or manual payment switched off, invoicing unavailable) raises no `invoice.*` event.
- `terminal` on the Payment object is `null` for a payment that was not taken on a terminal. For a payment taken on a terminal it holds the sale, and `payment_plan.payment_succeeded` is sent when ChaChing records it after the terminal approved it; see [Take a Payment on a Terminal](./payment-plans-terminal.md#take-a-payment-on-a-terminal)
- The plan status `past_due` means the customer's account is in the late-payment warning. It is not the `past-due` failed-payment outcome: reaching that outcome sends `payment_plan.defaulted`
- The `invoice.*` events keep firing for every installment invoice; the invoice payload's `payment_plan` and `installment` fields link it back to the plan
- The spelling is `canceled`, with one `l`, for both the status and the event
- One change can send several events, and deliveries can arrive in any order. Read the plan with `GET /payment-plans/{id}` for its current state; see [Payment Plans](./payment-plans.md)
- To send your own receipts and notices from these events, see [Send Your Own Customer Notifications](./customer-notifications.md)

---

# Payload Behavior

- The payload reflects the **entity after change**
- Example:
    - Customer updated → full updated customer data is sent
    - Subscription created → full subscription object is sent

---

# Important Notes

The data object is contained inside the `data` field.

- The `data` field contains the **full entity object**

---

Deleted events carry the full object as it was just before the deletion.

---

Always rely on `event`

- Use `event` field to determine how to parse `data`

---

# Important Notes for Integrators

**Choose the events of each endpoint**

An endpoint receives only the event types in its `enabled_events`

➡ Listen only to the event types your integration needs, and keep switching on `event` in your handler: an endpoint with `["*"]` also receives event types added later

---

**Payload varies per event**

`data` structure depends on event type

➡ You must handle multiple schemas

---

**Retries may cause duplicates**

➡ Use `id` to deduplicate events

---

**Endpoint must be stable**

➡ If your endpoint fails → events may be lost after retries

---