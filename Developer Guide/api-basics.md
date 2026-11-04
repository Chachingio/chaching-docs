# API Basics

## Environments

| **Environment** | **Base URL** |
| --- | --- |
| Staging | `https://stage-api.chaching.io` |
| Production | `https://api.chaching.io` |

Paths in this guide and in the [API Reference](/developer-guide/api) go directly after the base URL, with no version prefix: `GET https://stage-api.chaching.io/customers`. Requests and responses are JSON over HTTPS.

Staging and production are separate: an account, its keys, its customers and its data exist in one environment only.

## Authentication

Every request carries your account's secret API key in the `cc-api-key` header:

```
cc-api-key: sk_...
```

- Every endpoint of the [API Reference](/developer-guide/api) requires the **secret key** (`sk_...`). A publishable key (`pk_...`) answers `403` with `{"error":"SECRET_API_KEY_REQUIRED","message":"This route requires the account secret API key (sk_...)"}`.
- The publishable key is used only by the ChaChing hosted checkout pages. Never use it for server-side calls.
- Do not send an `Authorization` header with the API key. Some endpoints answer `409` when a request carries both.
- Keep the secret key on your server. Never put it in a browser, a mobile app or any page your customers open.

You find both keys in the dashboard under **Settings → Developer Settings**. See [Developer Settings](../settings/DeveloperSettings.md).

## Authentication Errors

| **Status** | **When** |
| --- | --- |
| `400` | The request has no `cc-api-key` header (`"message": "Missing apiKey parameter"`). |
| `401` | The key is unknown, or it belongs to a deactivated account (`Invalid apiKey`). |
| `403` | The key is a publishable key (`SECRET_API_KEY_REQUIRED`). |

## Names

Every field, query parameter and path parameter of the [API Reference](/developer-guide/api) is in snake_case. Status values are lowercase: an invoice is `draft`, `open`, `paid` or `void`. The status of a subscription is one of `active`, `trial`, `scheduled`, `paused`, `unpaid`, `cancelled` or `expired`, spelled with two `l`s. The deprecated webhook field `createdAt` is the one exception in a payload; use `created`.

## Timestamps

`created`, `finalized_at` and the other event times are Unix seconds and mark an instant. A transaction's `created` can carry a fraction of a second. `due_date` is a day, not an instant: it is the Unix time of 00:00 UTC on that day, and the day is a calendar date in the account time zone (see [Account time zone and dates](#account-time-zone-and-dates)).

## Account time zone and dates

Every account has a time zone, set by ChaChing. The default is America/New_York; the zones in use lie between UTC−10:00 and UTC+00:00. Instants stay Unix seconds. A day field (`due_date`, the `start_date` of a payment plan, `next_due_date`) keeps its encoding, 00:00 UTC of the day, and the day is a calendar date in the account time zone.

- Payment plan dates are calendar days in the account time zone. The first installment date of a payment plan must be after the current UTC date. The default start date is one interval after today in the account time zone, never earlier than the day after the current UTC date. A down payment is due today in the account time zone.
- Subscription billing dates, invoice dates and the `start_date` and `end_date` of subscriptions are UTC calendar days. An invoice is dated with the UTC day on which it was generated.
- Day filters of the public API, such as `{"created": {"type": "equal", "date": "2026-11-02"}}`, compare UTC calendar days. The date filters of the ChaChing dashboard use the account time zone.
- A payment plan installment is scheduled on its due date at 10:00 UTC for a customer created since ChaChing introduced the fixed billing time, and at the time of day the customer was created for an older customer. In the account time zone that time can fall on the evening before the due date: a customer created at 02:30 UTC whose installment is due on November 3, 2026 is charged on November 2, 2026 at 9:30 p.m. EST.
- An installment is also charged at once, before its scheduled time, when another billing change on the same customer happens between 00:00 UTC and that time on its due date (for example a new subscription, including one the customer starts on a hosted checkout page, a pause, a resume, or a payment of a balance that had blocked billing), which in a US time zone can be as early as the evening before the due date. The "Billing Time of Day" section of the [payment plans guide](/developer-guide/payment-plans) lists every case.

## Pagination

Every list except the three named below takes `page` and `take` (`take` at most `50`) and answers:

```json
{ "data": [], "meta": { "page": 1, "take": 10, "items_total": 0, "pages_total": 0 } }
```

`items_total` and `pages_total` are numbers.

Three routes are exceptions. `GET /customers/search` answers a bare array and takes no `page` or `take`. `GET /dejavoo/terminals` answers `{ "data": [...] }` with no `meta`. `GET /products/taxCodes` answers one object that maps each tax code to its category name, takes only `search`, and has no paging.

## Contract Changes

These changes came before the first integrations went live. The API has no version: a request that sends a removed name answers `400`.

| **Where** | **Before** | **Now** |
| --- | --- | --- |
| Every paged list | `meta.itemsTotal`, `meta.pagesTotal` | `meta.items_total`, `meta.pages_total` |
| `POST /payment-methods/{id}/attach` | `customerId`, `cardNumber`, `cardType`, `cardExp`, `walletType` | `customer_id`, `card_number`, `card_type`, `card_exp`, `wallet_type` |
| `POST /payment-methods/{id}/set-default` | `customerId` | `customer_id` |
| `GET /customers/{customer_id}/payment-methods` and its single read | `card.walletType` | `card.wallet_type` |
| `PATCH /customers/{customer_id}` | `billingAddress`, `shippingAddress` | `phone`, `address`, `shipping`. `address` is shaped as on `POST /customers`; `shipping` takes `phone` and `address`, and `shipping.name` is not accepted |
| `PUT /invoices/{id}` response | the billing engine's payment object in camelCase, amounts in dollars | an `invoice_payment` object in snake_case, amounts in cents |
| `GET /customers` | `sortingField` (never read) | removed |
| Invoice `status`, in the API and in webhooks | `DRAFT`, `OPEN`, `PAID`, `VOID` | `draft`, `open`, `paid`, `void` |
| Invoice `collection_method` | always `charge_automatically` | the method of the invoice; an older `send_invoice` invoice, created before the value was recorded, reads `charge_automatically` |
| Invoice `created`, `status_transitions.finalized_at` | midnight of the invoice date | the instants; `finalized_at` is `null` on a draft |
| `lines.data` of a payment plan invoice | empty description, null amount | the payment page text and the amount; `plan` and `price` are `null` |
| `POST /invoices` with `charge_automatically` | a missing `default_payment_method` answered with a confusing `400` | a `400` that names the field |
| `DELETE /subscriptions/{id}`, already cancelled, cancelling or expired | `500`, or a second `subscription.canceled` | `400 SUBSCRIPTION_CANCELLED`, `400 SUBSCRIPTION_EXPIRED` |
| `POST /subscriptions` with a short `end_date` | a subscription created already expired | a subscription that ends on `end_date`; `400 SUBSCRIPTION_END_DATE_TOO_EARLY` when it is not after the first billed day |
| `POST` and `PATCH /subscriptions` with an unknown customer | `500` | `404 Customer or price not found` |
| A `VOID` transaction | `amount` and `amount_charged` were `null`, undocumented | still `null`: a void carries no amount |
| `GET /customers/{customer_id}` with an unknown customer | `404 Customer details not found` | `404 Customer not found` |
| `PATCH` and `DELETE /customers/{customer_id}` with an unknown customer | `400 Customer not found` | `404 Customer not found` |
| `DELETE /customers/{customer_id}` on an already deleted customer | the delete ran again and, when every step succeeded, sent a second `customer.deleted` | `404 Customer not found`, no webhook |
| `GET /products/{id}`, `GET /prices/{id}`, `GET /taxRates/{id}` with an unknown id | `200` with an empty body | `404 Product not found`, `404 Price not found`, `404 Tax rate not found` |
| `PATCH /taxRates/{id}` with an unknown id | `200` with an empty body | `404 Tax rate not found` |
