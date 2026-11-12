# Platforms

A platform is a company that operates ChaChing on behalf of many merchants, for example a practice-management system that serves many dental offices. A platform owns **companies** (one per office) and each company owns **accounts** (one per practice location). Each account is a full ChaChing account with its own customers, plans, terminals and credentials; the platform acts on it with one key.

| **Object** | **Id** | **Your id** |
| --- | --- | --- |
| Company | `com_...` | `company.external_id` |
| Account | `acc_...` | `external_id` |

Store one platform key per environment, one signing secret per webhook endpoint, and one account id per location.

## Get a platform key

Chaching creates your platform and invites your administrator by email. Your administrator registers through the link (it lasts 24 hours and works once) and creates the platform's keys (`psk_...`, 47 characters) in the platform console at `https://stage.chaching.io/platform`, under **API keys**. Creating or rotating a key takes a 6-digit code that the console emails to the signed-in administrator; the code expires after 10 minutes. The console shows a key once, when it is created or rotated, and Chaching never sees it: store it in your secrets manager at that moment. To replace a key, rotate it and choose under **Current key expires** when the old key stops working: **Now**, **In 1 hour**, **In 24 hours** (the default) or **In 7 days**. Both keys work until then, so deploy the new key before that moment; a key that already expires sooner keeps its earlier expiry. Revoking a key stops it from the next request and takes no code. A platform has at most 10 live keys, and a rotated key counts until it expires. See [Get access](./integrating-staging.md#get-access).

## Authenticate

- `/platform` routes take `cc-api-key: psk_...` and no other credential.
- Every other route takes `cc-api-key: psk_...` and `Chaching-Account: acc_...`, the id of the account the request acts on. Paths, bodies and responses are the same as for an account's own key.
- Send no `Authorization` header (see [API Basics](./api-basics.md#authentication)).

```bash
curl https://stage-api.chaching.io/payment-plans \
  -H "cc-api-key: $CHACHING_PLATFORM_KEY" \
  -H "Chaching-Account: $CHACHING_ACCOUNT_ID"
```

| **Status** | **Error code** | **Message** | **When** |
| --- | --- | --- | --- |
| `400` | no code | `Missing apiKey parameter` | No `cc-api-key` header. |
| `400` | no code | `Invalid apiKey format` | The key does not start with `pk_`, `sk_` or `psk_`. |
| `400` | `CHACHING_ACCOUNT_REQUIRED` | `This request uses a platform API key: send the Chaching-Account header with the id of the account it acts on.` | An account route without `Chaching-Account`. |
| `401` | no code | `Invalid apiKey` | The key is unknown, revoked or expired, or your platform is inactive. |
| `403` | `PLATFORM_API_KEY_REQUIRED` | `This route requires a platform API key (psk_...)` | An account secret key on a `/platform` route. |
| `403` | `SECRET_API_KEY_REQUIRED` | `This route requires the account secret API key (sk_...)` | A publishable key on any route. |
| `404` | `ACCOUNT_NOT_FOUND` | `Account not found` | `Chaching-Account` names no account of your platform, whatever the reason. |
| `409` | `ACCOUNT_DEACTIVATED` | `This account is deactivated. It accepts reads, payment plan cancellations and terminal updates only.` | A write to a deactivated account. See [Deactivate an account](#deactivate-an-account). |

When a request has several faults, the answer reports one of them: the account lookup runs before the key-class check, and on the `/platform` lists the query validation runs before it. Fix the fault reported and send the request again.

## Create an account

```bash
curl -X POST https://stage-api.chaching.io/platform/accounts \
  -H "cc-api-key: $CHACHING_PLATFORM_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: create-account-office-1042-location-7" \
  -d '{
    "external_id": "location-7",
    "name": "Rosen Family Dental - Main Street",
    "company": {
      "external_id": "office-1042",
      "name": "Rosen Family Dental"
    },
    "time_zone": "America/New_York"
  }'
```

<!-- captured: POST /platform/accounts on stage, redacted -->
```json
{
  "object": "account",
  "id": "acc_0c194ca27ec457a2f1b50f1f",
  "external_id": "location-7",
  "name": "Rosen Family Dental",
  "company": {
    "id": "com_1779003a63191104815fd879",
    "external_id": "office-1042",
    "name": "Rosen Family Dental"
  },
  "time_zone": "America/New_York",
  "status": "active",
  "deactivated_at": null,
  "created": 1794269584
}
```

Returns the Account object with status `201`. The company is created on its first use and reused afterwards by `company.external_id`. When an active account of that company already has the same `external_id`, the request returns it with status `200`, unchanged: its `name`, `time_zone` and the company `name` are not updated, and a deactivated account stays deactivated. Two concurrent creates of the same account return one account. No route changes an account's `name`, `time_zone` or ids after creation.

- `time_zone` defaults to `America/New_York`. It accepts an IANA zone whose UTC offset stays between -10:00 and +00:00 all year (the US zones, `Pacific/Honolulu`, `UTC`); any other answers `400 ACCOUNT_TIME_ZONE_INVALID`. Payment plan dates are calendar days in this zone; billing runs at 10:00 UTC.
- With an `Idempotency-Key`, the rules of [Idempotency](./integrating-staging.md#idempotency) apply. A `500` stored under an `Idempotency-Key` replays for 24 hours, so repeat the request with the same `external_id` values and a NEW `Idempotency-Key`: the create is idempotent on those ids and answers `200` with the existing account or `201`.

A new account starts with payment plans enabled, customer emails and owner copies off (your platform sends every notice from webhooks), and the revenue-recovery retries 1, 3, 5 and 7 days with the outcome `cancel`. Under `cancel`, a plan whose customer reaches the outcome stops being charged; change it with `PUT /revenue-recovery` (see [Subscription Lifecycle](./subscription-lifecycle.md)).

| **Status** | **Error code** | **When** |
| --- | --- | --- |
| `400` | no code | A field is missing or invalid, or the body has an unknown field. |
| `400` | `ACCOUNT_TIME_ZONE_INVALID` | `time_zone` is not accepted. |
| `400` | `IDEMPOTENCY_KEY_INVALID` | The key is not 1 to 255 printable ASCII characters without spaces. |
| `409` | `IDEMPOTENCY_KEY_IN_PROGRESS` | The first request with this key is still running. |
| `422` | `IDEMPOTENCY_KEY_REUSED` | The key was used with a different body. |
| `500` | no code | The outcome is unknown: the account may or may not exist. Repeat the request with the same external_id values (company and account) and a new Idempotency-Key: it returns the existing account or creates it. A billing-engine record created during a failed attempt can remain behind. |
| `502` | `IDEMPOTENCY_REQUEST_INCOMPLETE` | The first request with this key never completed. Read the account list before you retry with a new key. |

## List and retrieve accounts

```bash
curl -G https://stage-api.chaching.io/platform/accounts \
  -H "cc-api-key: $CHACHING_PLATFORM_KEY" \
  -d company_external_id=office-1042 \
  -d status=active
```

<!-- captured: GET /platform/accounts on stage, redacted -->
```json
{
  "data": [
    {
      "object": "account",
      "id": "acc_0c194ca27ec457a2f1b50f1f",
      "external_id": "location-7",
      "name": "Rosen Family Dental",
      "company": {
        "id": "com_1779003a63191104815fd879",
        "external_id": "office-1042",
        "name": "Rosen Family Dental"
      },
      "time_zone": "America/New_York",
      "status": "active",
      "deactivated_at": null,
      "created": 1794269584
    }
  ],
  "meta": {
    "page": 1,
    "take": 10,
    "items_total": 1,
    "pages_total": 1
  }
}
```

Returns your platform's accounts, newest first, filtered by `external_id`, `company_external_id` (both exact) and `status` (`active` or `deactivated`). `GET /platform/accounts/{id}` returns one account, deactivated ones included; an id that names no account of your platform answers `404 ACCOUNT_NOT_FOUND`.

## Set up a location

Run these in order for each new location:

1. `POST /platform/accounts`, and store the `id`.
2. `PUT /dejavoo/credentials` with `Chaching-Account`: the location's iPOSpays e-commerce TPN, API key, secret key and e-commerce token (all four required; the FEED signing key and Google Pay merchant id are optional). On staging use iPOSpays UAT credentials. See [Processor Credentials](./processor-credentials.md). To receive the location's FEED notifications, include `feed_signing_key` and give Dejavoo the location's FEED webhook URL; see [The Dejavoo FEED Webhook URL](./processor-credentials.md#the-dejavoo-feed-webhook-url).
3. `POST /dejavoo/terminals` for each terminal. Before step 2 it answers `409 DEJAVOO_NOT_CONFIGURED`.
4. `POST /customers`, then the payment plans. Before step 2, `POST /customers` answers `409 PAYMENT_GATEWAY_NOT_CONFIGURED` with the message `This account has no payment gateway credentials. Save the payment gateway credentials before creating customers.`

Read the credential state of a location with `GET /dejavoo/credentials`.

## Platform webhooks

`POST /platform/webhook-endpoints` registers an endpoint that receives the events of every account of your platform, including accounts created later. The nine `/platform/webhook-endpoints` operations take the same bodies and answer the same objects as the account-level [webhook endpoint operations](./webhook.md); your platform holds at most 16 endpoints. An event of an account goes to that account's own endpoints and to your platform's endpoints with the same event `id`; deduplicate on it.

Each delivery's envelope names the account:

```json
"account": {
  "id": "acc_9f3a1c7e5b2d4086a1c3e5f7",
  "external_id": "location-7",
  "company": {
    "id": "com_Hp3sJ6uK9wE2rT5yA8zX1vB4",
    "external_id": "office-1042"
  }
}
```

`GET /platform/webhook-endpoints/{id}/logs` lists an endpoint's deliveries across all your accounts, without the payload and without the account. To read a delivery's payload and attempts, call `GET /webhook-deliveries/{id}` with the `Chaching-Account` of the account whose `id` is in the envelope you received.

## Deactivate an account

`POST /platform/accounts/{id}/deactivate` and `/reactivate` answer the Account object; repeating one changes nothing. While an account is deactivated:

- Every `GET`, `POST /payment-plans/{id}/cancel` and `PATCH /dejavoo/terminals/{id}` work.
- Every other write answers `409 ACCOUNT_DEACTIVATED` and changes nothing, including `POST /payment-plans/preview`, payments, deposits, credential saves, email settings and webhook endpoint writes.
- Scheduled installments, retries and webhooks keep running.
- A plan with a terminal attempt that is still open cannot be cancelled until you reactivate the account.

## What does not apply to a platform account

- No dashboard, no users, no bank account, no ChaChing monthly fee.
- The account's own publishable and secret keys do not exist; a key of that kind answers `401 Invalid apiKey`.
- Hosted checkout and invoice pages: they do not work for a platform account: the `hosted_invoice_url` of its invoices and the checkout address of its prices open on "Invalid or expired token — The requested item could not be found." (observed on staging, 2026-11-10). Collect payments on a platform account through the API and your own pages.
- Not built: platform key scopes, an IP allow-list and a per-key rate limit. Platform keys are created, rotated and revoked in the platform console, not through the public API. One rate limit of 200 requests per minute applies per IP address; see [Rate limits](./integrating-staging.md#rate-limits).

## Environment

The platform layer exists on staging today; every `/platform` operation carries **Staging only** in the [API Reference](./api.json). Production receives it in a later promotion, with its own platform key.
