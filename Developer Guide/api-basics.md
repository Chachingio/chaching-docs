# API Basics

## Environments

| **Environment** | **Base URL** | **Money moves** |
| --- | --- | --- |
| Staging | `https://stage-api.chaching.io` | No |
| Production | `https://api.chaching.io` | Yes |

Paths in this guide and in the [API Reference](/developer-guide/api) go directly after the base URL, with no version prefix: `GET https://stage-api.chaching.io/customers`. Requests and responses are JSON over HTTPS. Staging and production are separate: an account, its keys, its customers and its data exist in one environment only.

Staging carries the newest release; production receives it in the next promotion. Operations that exist only on staging today carry a **Staging only** badge in the API Reference; see [What production does not have yet](./integrating-staging.md#what-production-does-not-have-yet).

## Authentication

Every request carries an API key in the `cc-api-key` header: your account's secret key, or, for a platform, the platform key together with the `Chaching-Account` header.

```bash
curl https://stage-api.chaching.io/customers \
  -H "cc-api-key: $CHACHING_SECRET_KEY"
```

| **Key** | **Prefix** | **Used for** | **Safe to expose** |
| --- | --- | --- | --- |
| Secret key | `sk_` | Every API request of your account. | No |
| Publishable key | `pk_` | The ChaChing hosted checkout pages only. The API answers `403 SECRET_API_KEY_REQUIRED`. | Yes, in a hosted checkout link |
| Platform key | `psk_` | `/platform` routes, and every account route together with `Chaching-Account`. See [Platforms](./platforms.md). | No |
| Webhook signing secret | `whsec_` | Verifying webhook deliveries; not an API key. | No |

- `Chaching-Account: acc_...` names the account a platform request acts on. It is required with a platform key on every route outside `/platform`, and ignored on `/platform`. With an account key it is optional and must name that key's own account.
- Send the key in `cc-api-key` and send no `Authorization` header. An `Authorization` value that is not exactly `Bearer`, one space and a token answers `401 MALFORMED_AUTHORIZATION_HEADER` on every endpoint. A well-formed `Bearer` header answers `409` with the message `This route authenticates with cc-api-key only; remove the Authorization header` and no `error` code. `PUT /invoices/{id}` and `POST /payment-methods/{id}/attach` are the two exceptions: each accepts a request that carries an `Authorization: Bearer` header the API cannot verify (for example an expired token) together with a valid `cc-api-key`, and the key wins.
- Keep secret and platform keys on your server. Never put them in a browser, a mobile app or any page your customers open.

You manage account keys in the dashboard under **Settings, Developer settings**, where you create, rotate and revoke them. A secret key is shown only once. See [Developer Settings](../settings/DeveloperSettings.md).

## Authentication Errors

| **Status** | **Error code** | **Message** | **When** |
| --- | --- | --- | --- |
| `400` | no code | `Missing apiKey parameter` | The request has no `cc-api-key` header. |
| `400` | no code | `Invalid apiKey format` | The key does not start with `pk_`, `sk_` or `psk_`. |
| `400` | `CHACHING_ACCOUNT_REQUIRED` | `This request uses a platform API key: send the Chaching-Account header with the id of the account it acts on.` | A platform key without `Chaching-Account`. |
| `401` | no code | `Invalid apiKey` | The key is unknown, revoked or expired, or it belongs to a deactivated account. |
| `401` | `MALFORMED_AUTHORIZATION_HEADER` | `Malformed Authorization header` | An `Authorization` header that is not `Bearer`, one space and a token. |
| `401` | no code | `Conflicting account credentials` | An `Authorization` bearer of another account. |
| `403` | `SECRET_API_KEY_REQUIRED` | `This route requires the account secret API key (sk_...)` | A publishable key. |
| `403` | `PLATFORM_API_KEY_REQUIRED` | `This route requires a platform API key (psk_...)` | A key other than a platform key on `/platform`. |
| `404` | `ACCOUNT_NOT_FOUND` | `Account not found` | `Chaching-Account` names no account the key can act on. |
| `409` | no code | `This route authenticates with cc-api-key only; remove the Authorization header` | A well-formed `Bearer` header. |
| `409` | `ACCOUNT_DEACTIVATED` | `This account is deactivated. It accepts reads, payment plan cancellations and terminal updates only.` | A write to a deactivated platform account. |
| `429` | no code | | Too many requests; see [Rate limits](#rate-limits). |

A `403` body reads, in full:

```json
{
  "statusCode": 403,
  "message": "This route requires the account secret API key (sk_...)",
  "error": "SECRET_API_KEY_REQUIRED",
  "timestamp": "2026-11-09T14:00:00.000Z",
  "path": "/customers"
}
```

When a request has several faults, the answer reports one of them; fix it and send the request again.

## Errors

Every error body has `statusCode`, `message`, `timestamp` and `path`, and `error` when the failure has a stable code. Branch on `statusCode` and `error`, never on `message`.

- A validation failure (`400` for a body, a query or a path value) has `message` as an array of strings and no `error`.
- A failure with a stable code has `error` in upper case and `message` as a string.
- A search parameter that is not a JSON object, a search key outside the documented keys and an invalid filter value answer `400` with `message` as a string and no `error`.
- A missing object answers `404` with `message` such as `Customer not found`; some routes add an `error` code, as each guide's error table states.

| **Status** | **Meaning** |
| --- | --- |
| `400` | The request is invalid. |
| `401` | The key is not valid. |
| `402` | A charge was declined. |
| `403` | The key cannot use this route. |
| `404` | The object does not exist for this account. |
| `409` | The request conflicts with the object's state. |
| `422` | An idempotency key was reused, or a credential was refused. |
| `429` | Too many requests. |
| `500` | ChaChing failed; the route's guide says whether a retry is safe. |
| `502`, `503` | A gateway or the billing engine did not answer; the route's guide says whether a retry is safe. |

## Rate limits

200 requests per 60 seconds from one IP address, over all keys. Every response carries `RateLimit-Policy`, `RateLimit-Limit`, `RateLimit-Remaining` and `RateLimit-Reset`; beyond the limit the API answers `429`. Some routes have lower limits; see [Rate limits](./integrating-staging.md#rate-limits).

## Names

Every field, query parameter and path parameter of the [API Reference](/developer-guide/api) is in snake_case. Status values are lowercase: an invoice is `draft`, `open`, `paid` or `void`; the `status` of a Transaction object and of each `transactions[]` entry of the `PUT /invoices/{id}` answer is `succeeded`, `failed`, `refunded`, `disputed`, `voided`, `pending` or `unknown`, and the `status` filter of `GET /transactions` takes `succeeded`, `failed`, `refunded` and `disputed`. The status of a subscription is one of `active`, `trial`, `scheduled`, `paused`, `unpaid`, `cancelled` or `expired`, spelled with two `l`s. The deprecated webhook field `createdAt` is the one exception in a payload; use `created`. The `status` search filter of `GET /transactions` matches the payment's outcome (`succeeded`, `failed`, `refunded`, `disputed`), while each transaction's own `status` field reports that transaction (a refunded payment lists its original `succeeded` purchase and its `refunded` refund), so a filtered list can contain rows whose own `status` differs from the filter value.

## Identifiers

| **Object** | **Id** |
| --- | --- |
| Customer | `cus_` and 24 letters and digits |
| Payment plan, installment, payment | `pp_`, `ppi_`, `ppp_` and 24 letters and digits |
| Terminal attempt | `ppda_` and 24 letters and digits |
| Account, company | `acc_`, `com_` and 24 letters and digits |
| Event | `evt_` and 24 hexadecimal digits |
| Webhook delivery | `whd_` and 24 hexadecimal digits |
| Invoice, terminal, webhook endpoint | a UUID |
| Payment method | `pm_` and letters and digits, or a UUID, depending on the gateway |

A path id that is neither `prefix_` followed by letters and digits nor a UUID answers `400`. `POST /payment-methods/{id}/attach` is the one operation outside this rule: its `{id}` is the gateway card token (a Stripe `pm_` id, a hyphenated NMI Collect.js token or a Dejavoo card token), not an id of the table above. A well-formed id that names nothing of your account answers `404`. Two routes differ: `/platform/accounts/{id}` answers `404 ACCOUNT_NOT_FOUND` for every id it cannot resolve. Products, prices, subscriptions, payment plans and terminal attempts take only `prefix_` ids, and terminals only UUIDs; the other shape answers `400` there.

## Nullable Fields

A field listed as required in the API Reference is always present in the response. A required field can hold `null`: its schema is marked nullable, and the field is `null` when it has no value. A field that is not required can be absent.

## Timestamps

`created`, `finalized_at` and the other event times are Unix seconds and mark an instant. A transaction's `created` can carry a fraction of a second. `due_date` and `start_date` are days, not instants: each is the Unix time of 00:00 UTC on that day. Payment plan days follow the account's time zone (`time_zone`, default `America/New_York`); billing runs at 10:00 UTC.

## Pagination

Every list except the three named below takes `page` (an integer from `1` to `1000000`) and `take` (an integer from `1` to `50`, default `10`) and answers:

```json
{ "data": [], "meta": { "page": 1, "take": 10, "items_total": 0, "pages_total": 0 } }
```

A `page` past the last page answers `200` with an empty `data`; a `page` above `1000000` answers `400`. `items_total` and `pages_total` are numbers. `order` takes `ASC` or `DESC`, in upper case.

Three routes are exceptions. `GET /customers/search` answers a bare array of at most 50 customers, newest first, and takes only `search`: any other query parameter, `page` and `take` included, answers `400`. `GET /dejavoo/terminals` answers `{ "data": [...] }` with no `meta`. `GET /products/taxCodes` answers one object that maps each tax code to its category name, takes only `search`, and has no paging.

## Search Filters

A list's `search` parameter is a URL-encoded JSON object whose keys the API Reference lists for that list, for example `GET /invoices?search={"status":"open"}`. A key outside that list, a value outside its allowed values, and a `search` that is not a JSON object answer `400`. A status filter takes a comma-separated list, such as `{"status":"open,paid"}`. The shape of a `created` filter is the one the API Reference shows for that list: on transactions, webhook deliveries and terminal attempts it is an object such as `{"type":"inTheLast","amount":7,"unit":"days"}` (units `days`, `weeks`, `months`, `years`; also `equal`, `before` and `after` with `date`, and `between` with `from` and `to`).

## Contract Changes

These changes came before the first integrations went live. The API has no version: a request that sends a removed name answers `400`.

| **Where** | **Before** | **Now** |
| --- | --- | --- |
| Every paged list | `meta.itemsTotal`, `meta.pagesTotal` | `meta.items_total`, `meta.pages_total` |
| `POST /payment-methods/{id}/attach` | `customerId`, `cardNumber`, `cardType`, `cardExp`, `walletType` | `customer_id`, `card_number`, `card_type`, `card_exp`, `wallet_type` and the optional `zipcode` |
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
| `GET /balance` on an account that does not charge through Stripe | `200` with an empty body | `409 BALANCE_NOT_SUPPORTED` |
| Transaction `status` | `SUCCESS` and other upper-case values | one of `succeeded`, `failed`, `refunded`, `disputed`, `voided`, `pending`, `unknown`; the `status` filter of `GET /transactions` takes `succeeded`, `failed`, `refunded` and `disputed` |
| `status` of each `transactions[]` entry of the `PUT /invoices/{id}` answer | `SUCCESS` and other upper-case values | the same seven values as the Transaction object |
| `POST /subscriptions/{id}/pause` and `/resume` | `201` with a text body, whatever the state | `200` with the Subscription object; `400 SUBSCRIPTION_ALREADY_PAUSED`, `SUBSCRIPTION_NOT_PAUSED`, `SUBSCRIPTION_CANCELLED`, `SUBSCRIPTION_EXPIRED` |
| `DELETE /subscriptions/{id}` | `200` with a text body | `200` with the Subscription object |
| `GET /subscriptions/{id}` with an unknown id | `200` with `[]` | `404 Subscription not found` |
| `GET /customers/search` | every match, deleted customers included | at most 50 active customers, newest first |
| A malformed path id on customers, tax rates, transactions, payment methods, invoices, webhook endpoints and deliveries | `404` | `400` |
| A `search` key a list does not document | ignored | `400` |
| `GET /customers` `search` with `name` or `email` | ignored | filters the list |
| `GET /webhook-endpoints` `search` with `endpoint` | also returned other URLs | exact match |
| `page` above `1000000` | `500`, or `200` | `400` |
| `POST /subscriptions` with `items: []` | `201` with an empty body | `400` |
| `POST /invoices` with an unknown customer or price | `400` | `404 Customer not found`, `404 Price not found` |
| `GET /invoices/{id}/transactions` with an unknown invoice | `200` with an empty page | `404 Invoice not found` |
| `DELETE /prices/{id}` | `200` with an empty body | `200` with `{ id, object, deleted }` |
| `DELETE /customers/{customer_id}` | `200` with `1` | `200` with `{ id, object, deleted }` |
| A terminal payment on a plan canceled before its deposit | `409 PAYMENT_PLAN_TERMINAL_NOT_SUPPORTED` | `409 PAYMENT_PLAN_NOT_PAYABLE` |
| `POST /customers` with an invalid `email` | `201` | `400` |
| A well-formed `Authorization` header on `GET /customers`, `/products`, `/invoices`, `/subscriptions`, `/transactions` and the payment plan reads | accepted | `409` |
| Customers in `GET /customers` | `address` was `{}`; `description`, `phone` and `shipping.address` missing | the same keys as `GET /customers/{customer_id}` |
| The Payment Method object | `card` keys differed by route | `id`, `card` (`brand`, `last4`, `exp_month`, `exp_year`, `wallet_type`) and `customer` on every route |
| `billing_reason` of a payment plan invoice | `manual`, once `subscription` | always `manual` |
| Tax rate `percentage` | a string such as `"5.00"` | a number such as `5` |
| Invoice `subscription`, `tax`, `total_taxes`, `period_start`, `period_end`, `next_payment_attempt`, `lines.data[].price.type` | absent when empty | present, `null` when empty |
| Subscription `initiated_by`, `reason` in the API | absent | present, `null` outside a lifecycle event |
| `POST /platform/accounts` when ChaChing cannot register the account's billing notifications | `201`, notifications lost | `500` means the outcome is unknown and the account may exist; repeating the request with the same `external_id` values and a new `Idempotency-Key` returns the existing account or creates it; a billing-engine record can remain behind |
