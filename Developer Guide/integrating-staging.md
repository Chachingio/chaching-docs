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

- **Platform.** Chaching creates your platform and invites your administrator by email. The invite link lasts 24 hours and works once. Give Chaching an administrator address that is not a user of a Chaching merchant dashboard: an address belongs to one platform at most, and an address that is already a dashboard user is refused. Your administrator registers through the link, then creates, rotates and revokes platform keys (`psk_...`) in the platform console at `https://stage.chaching.io/platform`, under **API keys**. Creating or rotating a key takes a 6-digit code that the console emails to the signed-in administrator; revoking a key takes no code. The console shows a key once, when it is created or rotated, and Chaching never sees it: store it in your secrets manager at that moment. A platform has at most 10 live keys. Production keys come from the production console after the production promotion. The public API has no route that creates, rotates or revokes a platform key: these are console actions.
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

These values were measured on the staging UAT merchant (2026-10-05 and 2026-10-23). iPOSpays does not publish UAT test cards or amount-triggered answers on https://uatdocs.ipospays.tech; its documentation directs test-card requests to devsupport@dejavoo.io (https://uatdocs.ipospays.tech/hosted-payment-page/apidocs, read 2026-11-09). On a terminal, present the card on the UAT terminal itself.

Accounts that charge through NMI use NMI's test mode cards, and accounts that charge through Stripe use Stripe's test cards (https://docs.stripe.com/testing).

Billing runs once a day at 10:00 UTC; it cannot be triggered early. A plan becomes `past_due` one day after a failed installment stays unpaid, and `defaulted` on the outcome day, day 8 or later with the default retries of 1, 3, 5 and 7 days.

Events a request causes, which you can provoke on staging:

| **Request** | **Events** |
| --- | --- |
| Customer, payment method, product, price and tax rate writes | `customer.*`, `payment_method.*`, `product.*`, `price.*`, `tax.*` |
| `POST /invoices` | `invoice.created`, `invoice.finalized` |
| `POST /invoices/{id}/send` | `invoice.sent` |
| `PUT /invoices/{id}` | `invoice.payment_succeeded` or `invoice.payment_failed` |
| `DELETE /invoices/{id}` | `invoice.deleted` for a voided invoice; `invoice.written_off` when an open invoice that bills a subscription or payment plan period is written off |
| `POST /subscriptions`, `PATCH /subscriptions/{id}` | `subscription.created`, `subscription.updated` |
| `POST /subscriptions/{id}/pause`, `/resume`, `DELETE /subscriptions/{id}` | `subscription.paused`, `subscription.resumed`, `subscription.canceled` |
| `POST /payment-plans`, the terminal deposit, `POST /payment-plans/{id}/payments`, `POST /payment-plans/{id}/cancel` | `payment_plan.created`, `payment_plan.activated`, `payment_plan.installment_paid`, `payment_plan.payment_succeeded`, `payment_plan.completed`, `payment_plan.canceled` |

Events of the billing run, which need the daily run and the passing days: renewal invoices and their payments, and `subscription.expired`. The payment plan failure path has test helpers, described below.

## Payment plan test helpers

Test helpers move one payment plan through the failure path on staging in seconds, so that your endpoint receives every `payment_plan.*` event in one session. They exist only on staging: production answers each of them `403 TEST_HELPERS_NOT_AVAILABLE`, also after a release reaches production.

| **Request** | **What it does** | **Events** |
| --- | --- | --- |
| `POST /test-helpers/payment-plans/{id}/fail-installment` | Fails the plan's earliest scheduled or failed installment once more, as a declined charge would: `status` `failed`, `attempt_count` plus one, `next_payment_attempt` from your retry schedule (`null` once the retries are spent), `first_failed_at` on the first failure, and `failure_code` and `failure_message` from the body's `failure_code`: `04` (Pick up card (no fraud)), `05` (Do not honor, the default), `14` (Invalid card number), `51` (Insufficient funds) or `54` (Expired card). Call it again for each retry you want to see. The plan must be `active`, `past_due` or `defaulted`. | `payment_plan.installment_failed` and `payment_plan.payment_failed` |
| `POST /test-helpers/payment-plans/{id}/mark-past-due` | `active` to `past_due`. The plan needs a failed installment. | `payment_plan.past_due` |
| `POST /test-helpers/payment-plans/{id}/mark-defaulted` | `active` or `past_due` to `defaulted`. The plan needs a failed installment. | `payment_plan.defaulted` |
| `POST /test-helpers/payment-plans/{id}/reactivate` | A `past_due` or `defaulted` status that a test helper set, to `active`. A plan whose installment runs have all ended with nothing open becomes `paid_off` instead. | `payment_plan.reactivated`, or `payment_plan.completed` |

Each request answers `200` with the Payment Plan object. Send it with your secret key, or with a platform key holding `*` or `operate` together with `Chaching-Account`. The events, their envelopes, signatures and deliveries are exactly those of real events: nothing in them marks them as produced by a test helper. `fail-installment` also records the failed payment, listed by `GET /payment-plans/{id}/payments`, as a real decline does.

A request that cannot apply answers `409`: `TEST_HELPER_TRANSITION_NOT_ALLOWED` (the plan's status does not allow it), `TEST_HELPER_NO_OPEN_INSTALLMENT`, `TEST_HELPER_NO_FAILED_INSTALLMENT` (call `fail-installment` first), `TEST_HELPER_PLAN_IN_DUNNING` (the customer is in dunning in the billing engine, which owns the plan's status), `TEST_HELPER_STATUS_NOT_SIMULATED` (`reactivate` on a status that real dunning set) or `PAYMENT_PLAN_BUSY` (retry shortly).

What a test helper does not do: it charges nothing and does not call the billing engine, so the side effects of a real failure are absent. The customer gets no decline e-mail, no `invoice.payment_failed` is sent, and a `defaulted` plan keeps its schedule: the installments keep billing at 10:00 UTC on their due dates. As on a plan that dunning defaulted, a `defaulted` plan skips the fallback charge on the plan's card that follows a refused automatic payment. A failed installment that was not invoiced yet carries `invoice: null` and `hosted_invoice_url: null`; a real failure always carries its invoice.

How a simulated status ends:

- `reactivate`.
- By itself: 24 hours after the last `mark-past-due` or `mark-defaulted`, a status that a test helper set becomes eligible for a periodic expiry sweep. When the customer is not in dunning, the sweep applies the billing engine's clear decision: `active` with `payment_plan.reactivated`, `paid_off` with `payment_plan.completed` when every installment run has ended with nothing open, or the status kept when an installment is still open or dunning ended the schedule. Do not rely on an `active` status or a `payment_plan.reactivated` webhook at the 24-hour mark.
- A real billing event wins. When a failed installment is invoiced and charged on its due date, the charge result replaces the simulated failure (`payment_plan.installment_paid`, or a real `payment_plan.installment_failed`). A real overdue change of the customer takes the plan's status over, and the next real recovery sends the real `payment_plan.reactivated`.
- Payments and cancellation work as on any plan: a payment that rebuilds the schedule reactivates the plan, a payoff completes it, and canceling it cancels the installments not invoiced yet, a failed one included.

Every `payment_plan.*` event on staging:

| **Event** | **How to produce it** |
| --- | --- |
| `payment_plan.created` | `POST /payment-plans` |
| `payment_plan.activated` | `POST /payment-plans` with a card; the terminal deposit; `POST /payment-plans/{id}/payments` with `external: true` on a plan awaiting its deposit |
| `payment_plan.installment_paid` | `POST /payment-plans/{id}/payments` paying an invoiced installment or the deposit; the daily billing run |
| `payment_plan.payment_succeeded` | `POST /payment-plans/{id}/payments` |
| `payment_plan.completed` | A payoff through `POST /payment-plans/{id}/payments`; `reactivate` on a plan with nothing left |
| `payment_plan.canceled` | `POST /payment-plans/{id}/cancel` |
| `payment_plan.payment_method_updated` | `PATCH /payment-plans/{id}` |
| `payment_plan.installment_failed` | `fail-installment` |
| `payment_plan.past_due` | `mark-past-due` |
| `payment_plan.defaulted` | `mark-defaulted` |
| `payment_plan.reactivated` | `reactivate`; a payment that rebuilds the schedule of a `past_due` or `defaulted` plan |
| `payment_plan.payment_failed` | `fail-installment`; a declined `POST /payment-plans/{id}/payments` |
| `payment_plan.payment_refunded` | `POST /payment-plans/{id}/payments/{payment_id}/refunds` |
| `payment_plan.refund_forgiven` | `POST /payment-plans/{id}/payments/{payment_id}/refunds/{refund_id}/forgive` |

### The real failure path

To watch the billing engine itself, create a `daily` plan whose installment is `5` ($0.05) on the debit card `4111 1111 1111 1111` (no surcharge) and leave it running: the first installment bills at the next 10:00 UTC and is declined (`payment_plan.installment_failed`), and the plan becomes `past_due` and then `defaulted` on the days given above. Record the balance as collected outside (`POST /payment-plans/{id}/payments` with `external: true`) to receive the real `payment_plan.reactivated`. A declined $0.05 sale can make the iPOSpays UAT host refuse later sales on the same card number for hours to days (measured 2026-11-25), so do not use that card for other tests during this one.

## Idempotency

`POST /payment-plans`, `POST /payment-plans/{id}/payments` and `POST /platform/accounts` accept an `Idempotency-Key` header: 1 to 255 printable ASCII characters with no spaces (`400 IDEMPOTENCY_KEY_INVALID` otherwise). A key is kept for 24 hours.

- The same key with the same request returns the stored status and body with the header `Idempotent-Replayed: true`; nothing runs again.
- The same key with a different request answers `422 IDEMPOTENCY_KEY_REUSED`.
- The same key while the first request runs answers `409 IDEMPOTENCY_KEY_IN_PROGRESS`; after ten minutes without an outcome it answers `502 IDEMPOTENCY_REQUEST_INCOMPLETE`.
- Successes and `402`, `500` and `502` answers are stored and replayed, except `502 PAYMENT_PLAN_DOWN_PAYMENT_UNCONFIRMED` and `502 PAYMENT_PLAN_EXTERNAL_ID_CHECK_FAILED`, which release the key like every other answer (every other `4xx` and `503`): fix the request and retry with the same key. Retry a stored `500` with a new key.

Every other `POST` takes no key. Before you retry one of them, look the object up by your `external_id` (`GET /customers?external_id=…`, `GET /payment-plans?external_id=…`).

## Rate limits

Two regimes apply, by key.

**Account keys (`sk_`):**

- 200 requests per 60 seconds from one IP address, over all account keys. Polling one terminal deposit every 2 to 3 seconds uses 20 to 30 requests a minute.
- `POST /invoices/{id}/send` and `GET /invoices/{id}/pdf`: 60 requests per 60 seconds each, from one IP address.
- `PUT /dejavoo/credentials`, `/nmi/credentials` and `/stripe/credentials`: 10 per 60 seconds per key.

**Platform keys (`psk_`):**

- 6,000 requests per 60 seconds per platform and 600 requests per 60 seconds per account (`Chaching-Account`). A request past either quota answers `429 PLATFORM_RATE_LIMITED` with `Retry-After`, and keeps doing so when the client retries without honouring `Retry-After`. The `RateLimit-*` headers describe the tighter quota.
- Platform-key requests are not counted by the 200-per-IP limit. Their per-IP ceiling is 12,000 requests per 60 seconds from one IP address, whatever their outcome; it is answered like the general per-IP limit: `429` with the body `Too many requests, please try again later.`, `Retry-After` and no error code. Requests with different unrecognised keys fall under it.
- Refusals limit: after 200 refused requests with the same unrecognised platform key from one IP address in a minute (an unknown, revoked or expired key, a key of an inactive platform, a key used from outside its IP allow-list), that key from that IP answers `429 PLATFORM_API_KEY_REFUSALS_LIMITED` with `Retry-After` until the minute ends. It refuses only that key from that IP; a burst of legitimate concurrent requests is never refused by it.
- `PUT /dejavoo/credentials`, `/nmi/credentials` and `/stripe/credentials`: 10 per 60 seconds per key and location.
- Invoice documents keep their lower per-route limits; see [Invoices](./invoices.md#rate-limits).

Every response to an account key, and every platform-key response that reached the quota check, carries `RateLimit-Policy`, `RateLimit-Limit`, `RateLimit-Remaining` and `RateLimit-Reset`; a platform-key request refused earlier (`401`, `403`) carries none. Beyond a limit the API answers `429`; wait the seconds in `Retry-After` (or `RateLimit-Reset`) and back off on repeats. See [API Basics](./api-basics.md#rate-limits).

## Receive webhooks on staging

ChaChing delivers events to a public URL; it refuses `localhost`, private, loopback and other non-public addresses when you register the endpoint. To receive events on a laptop, expose your local server through a tunnel that gives it a public HTTPS URL, for example `cloudflared tunnel --url http://localhost:4242`, and register that URL. Answer `2xx` first, before slow work (ChaChing waits up to 30 seconds for the whole response and 10 seconds to connect); verify the signature on the raw body; deduplicate on the event `id`. See [Webhooks](./webhook.md).

## How a card is saved

On a Dejavoo account a card is saved by the terminal deposit of a payment plan (see [Payment Plans on a Terminal](./payment-plans-terminal.md)). `POST /payment-methods/{id}/attach` is the NMI browser flow (Collect.js) and is not a server-to-server card save. On a Dejavoo account, a card saved while the customer pays a hosted invoice cannot be charged again: create the plan with that card and the down payment is declined. Save the card through the subscribe checkout or collect the deposit on a terminal.

## What production does not have yet

These operation groups exist only on staging today; the [API Reference](./api.json) marks each operation with **Staging only**: Payment Plans (21), Platform (14), Webhook Endpoints (11), Processor Credentials (6), Dejavoo Terminals (5), Invoices (3), Email Settings (2), Revenue Recovery (2), Payment Methods (1). Production also answers some shared objects with older field names; build against staging and move to production after Chaching promotes the release. On production, `daily` plan intervals are refused and webhook URLs must be `https`. The four payment plan test helpers are staging-only by design: production answers them `403 TEST_HELPERS_NOT_AVAILABLE`.

## The API Reference

Each operation of the [API Reference](./api.json) shows a curl sample, the staging and production servers, and its request and response schemas.
