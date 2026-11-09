# Integrating Against Staging

Staging is where you build and test your integration. It carries the newest release of the API; production receives each release in a later promotion.

## Environments

| | **Staging** | **Production** |
| --- | --- | --- |
| API | `https://stage-api.chaching.io` | `https://api.chaching.io` |
| Merchant dashboard | `https://stage.chaching.io` | `https://dashboard.chaching.io` |
| Hosted checkout and invoice pages | `https://<slug>.stage.chaching.io`, or `https://stage.chaching.io` for an account without a slug | `https://<slug>.chaching.io`, or `https://dashboard.chaching.io` for an account without a slug |
| Money moves | No. Charges go to the gateways' test hosts; use test credentials and test cards only. | Yes. |

Paths go directly after the base URL, with no version prefix. The environments are separate: an account, its keys, its customers and its webhooks exist in one environment only, so you get a separate key for each. The keys carry no environment marker: the base URL decides where a request goes.

## Get access

- **Platform.** Chaching creates your platform and issues one platform API key (`psk_...`) per environment. Ask your Chaching contact for a staging platform key. Chaching issues it and sends it through a secure channel within one business day. It is shown to Chaching once, so store it in your secrets manager on receipt. Ask your Chaching contact for a production platform key after the production promotion. Chaching issues it and sends it through a secure channel within one business day. It is shown to Chaching once, so store it in your secrets manager on receipt. To replace a key, Chaching issues a second key, you deploy it, and Chaching revokes the first. No API route issues, rotates or revokes a platform key.
- **Merchant account.** The account owner creates secret keys in the dashboard under **Settings, Developer settings** and can rotate and revoke them there. A secret key is shown once. See [Developer Settings](../settings/DeveloperSettings.md).

| **Key** | **Prefix** | **Used for** | **Safe to expose** |
| --- | --- | --- | --- |
| Secret key | `sk_` | Every API request of your account. | No. Keep it on your server. |
| Publishable key | `pk_` | The ChaChing hosted checkout pages only; the API answers `403`. | Yes, in a hosted checkout link. |
| Platform key | `psk_` | `/platform` routes, and every account route together with `Chaching-Account`. | No. Keep it on your server. |
| Webhook signing secret | `whsec_` | Verifying webhook deliveries. It is not an API key. | No. |

## Test cards and amounts

On staging, Dejavoo accounts charge through the iPOSpays UAT host. The amount decides the answer:

| **Amount** | **Answer** |
| --- | --- |
| `100` ($1.00) | Approved. |
| `5` ($0.05) | Declined. |

| **Card** | **Number** | **Expiry** | **CVV** | **Type** |
| --- | --- | --- | --- | --- |
| Visa | `4012 0000 9876 5439` | `12/26` | `999` | Credit (a credit card surcharge applies when your account sets one) |
| Visa | `4111 1111 1111 1111` | `12/26` | `999` | Debit (no surcharge) |

These values were measured on the staging UAT merchant (2026-10-05 and 2026-10-23); iPOSpays publishes its UAT test data at https://uatdocs.ipospays.tech. On a terminal, present the card on the UAT terminal itself.

Accounts that charge through NMI use NMI's test mode cards, and accounts that charge through Stripe use Stripe's test cards (https://docs.stripe.com/testing).

Billing runs once a day at 10:00 UTC; it cannot be triggered early. A plan becomes `past_due` one day after a failed installment stays unpaid, and `defaulted` on the outcome day, day 8 or later with the default retries of 1, 3, 5 and 7 days.

Events a request causes, which you can provoke on staging:

| **Request** | **Events** |
| --- | --- |
| Customer, payment method, product, price and tax rate writes | `customer.*`, `payment_method.*`, `product.*`, `price.*`, `tax.*` |
| `POST /invoices` | `invoice.created`, `invoice.finalized` |
| `POST /invoices/{id}/send` | `invoice.sent` |
| `PUT /invoices/{id}` | `invoice.payment_succeeded` or `invoice.payment_failed` |
| `DELETE /invoices/{id}` | `invoice.deleted` |
| `POST /subscriptions`, `PATCH /subscriptions/{id}` | `subscription.created`, `subscription.updated` |
| `POST /subscriptions/{id}/pause`, `/resume`, `DELETE /subscriptions/{id}` | `subscription.paused`, `subscription.resumed`, `subscription.canceled` |
| `POST /payment-plans`, the terminal deposit, `POST /payment-plans/{id}/payments`, `POST /payment-plans/{id}/cancel` | `payment_plan.created`, `payment_plan.activated`, `payment_plan.installment_paid`, `payment_plan.payment_succeeded`, `payment_plan.completed`, `payment_plan.canceled` |

Events of the billing run, which need the daily run and the passing days: renewal invoices and their payments, `payment_plan.installment_failed`, `payment_plan.past_due`, `payment_plan.defaulted`, `payment_plan.reactivated` and `subscription.expired`.

## Idempotency

`POST /payment-plans`, `POST /payment-plans/{id}/payments` and `POST /platform/accounts` accept an `Idempotency-Key` header: 1 to 255 printable ASCII characters with no spaces (`400 IDEMPOTENCY_KEY_INVALID` otherwise). A key is kept for 24 hours.

- The same key with the same request returns the stored status and body with the header `Idempotent-Replayed: true`; nothing runs again.
- The same key with a different request answers `422 IDEMPOTENCY_KEY_REUSED`.
- The same key while the first request runs answers `409 IDEMPOTENCY_KEY_IN_PROGRESS`; after ten minutes without an outcome it answers `502 IDEMPOTENCY_REQUEST_INCOMPLETE`.
- Successes and `402`, `500` and `502` answers are stored and replayed, except `502 PAYMENT_PLAN_DOWN_PAYMENT_UNCONFIRMED` and `502 PAYMENT_PLAN_EXTERNAL_ID_CHECK_FAILED`, which release the key like every other answer (every other `4xx` and `503`): fix the request and retry with the same key. Retry a stored `500` with a new key.

Every other `POST` takes no key. Before you retry one of them, look the object up by your `external_id` (`GET /customers?external_id=…`, `GET /payment-plans?external_id=…`).

## Rate limits

- 200 requests per 60 seconds from one IP address, over all keys and all accounts. A platform backend that serves many locations from one IP shares this budget: polling one terminal deposit every 2 to 3 seconds uses 20 to 30 requests a minute.
- `POST /invoices/{id}/send` and `GET /invoices/{id}/pdf`: 60 requests per 60 seconds each, from one IP address.
- `PUT /dejavoo/credentials`, `/nmi/credentials` and `/stripe/credentials`: 10 per 60 seconds per key, and with a platform key per key and location.

Every response carries `RateLimit-Policy`, `RateLimit-Limit`, `RateLimit-Remaining` and `RateLimit-Reset`. Beyond a limit the API answers `429`; wait the seconds in `RateLimit-Reset` and back off on repeats. There is no per-key or per-platform limit beyond these.

## Receive webhooks on staging

ChaChing delivers events to a public URL; it refuses `localhost`, private, loopback and other non-public addresses when you register the endpoint. To receive events on a laptop, expose your local server through a tunnel that gives it a public HTTPS URL, for example `cloudflared tunnel --url http://localhost:4242`, and register that URL. Answer `2xx` within 15 seconds, before slow work; verify the signature on the raw body; deduplicate on the event `id`. See [Webhooks](./webhook.md).

## How a card is saved

On a Dejavoo account a card is saved by the terminal deposit of a payment plan (see [Payment Plans on a Terminal](./payment-plans-terminal.md)). `POST /payment-methods/{id}/attach` is the NMI browser flow (Collect.js) and is not a server-to-server card save. On a Dejavoo account, a card saved while the customer pays a hosted invoice cannot be charged again: create the plan with that card and the down payment is declined. Save the card through the subscribe checkout or collect the deposit on a terminal.

## What production does not have yet

These operation groups exist only on staging today; the [API Reference](./api.json) marks each operation with **Staging only**: Payment Plans (14), Platform (14), Webhook Endpoints (11), Processor Credentials (6), Dejavoo Terminals (5), Invoices (3), Email Settings (2), Revenue Recovery (2), Payment Methods (1). Production also answers some shared objects with older field names; build against staging and move to production after Chaching promotes the release. On production, `daily` plan intervals are refused and webhook URLs must be `https`.

## The API Reference

Each operation of the [API Reference](./api.json) shows a curl sample, the staging and production servers, and its request and response schemas.
