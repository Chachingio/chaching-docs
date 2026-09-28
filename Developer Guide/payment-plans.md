# Payment Plans

A payment plan collects a fixed total amount from one customer in a fixed number of scheduled installments. ChaChing stores the plan, charges each installment automatically on its due date against the plan's payment method, retries a declined charge on your account's retry schedule, and reports every change over webhooks. There is no interest and there are no fees: the customer pays exactly the total agreed when the plan was created.

Payment Plans are available through the API. They are enabled per merchant account by ChaChing and are off by default. While they are off, every `/payment-plans` endpoint answers `403` with the error code `PAYMENT_PLANS_NOT_ENABLED` and the message `Payment Plans are not enabled for this account. Contact Chaching to enable them.`

Every endpoint on this page authenticates with the account API key in the `cc-api-key` header, like every other public endpoint. Amounts are integers in cents, dates and timestamps are Unix seconds, and field names are `snake_case`. For every request and response schema, see the [API Reference](./api.json). For the webhook payloads, see [Webhooks](./webhook.md).

---

# Concepts

| **Term** | **Meaning** |
| --- | --- |
| Payment plan | An agreement to collect `total_amount` from one customer in a fixed number of installments. Its id starts with `pp_`. |
| Installment | One scheduled payment of a plan. Its id starts with `ppi_`. Each installment has a sequence number, a due date, an amount and a status, and it produces its own invoice when it is billed. |
| Down payment | An amount charged when the plan is created, before any installment. It is recorded as installment sequence `0`, and it is not counted in `installment_count`. |
| Payment | A payment you record on a plan with `POST /payment-plans/{id}/payments`. Its id starts with `ppp_`. |

The amounts always add up: `down_payment_amount` plus the sum of the scheduled installment amounts equals `total_amount`. When the amount left after the down payment does not divide evenly, one installment carries the odd amount: installment 1 with `remainder_placement: "first"` (the default), or the last installment with `"last"`. Every other installment carries exactly `installment_amount`.

For example, $2,100.00 at $200.00 a month with no down payment and the remainder first becomes 11 installments: installment 1 of $100.00, then 10 installments of $200.00. In API terms: `total_amount: 210000`, `installment_amount: 20000`, `remainder_placement: "first"`, and the plan reports `installment_count: 11`.

---

# Plan Statuses

| **Status** | **Meaning** |
| --- | --- |
| `active` | The plan is running. Installments are billed and charged on their due dates. |
| `past_due` | The customer's account is in the late-payment warning. Automatic charging continues. |
| `defaulted` | The customer's account reached your failed-payment outcome. What still runs depends on that outcome; see **Failed Payments and Payment Plans** below. |
| `paid_off` | The plan is paid in full. Nothing more is charged. |
| `canceled` | You canceled the plan. Installments not yet billed are removed; amounts already invoiced stay collectible. |
| `awaiting_deposit` | The plan was created without a payment method and waits for an in-person deposit. This path is not available in this release; see **Create a Plan**. |

`paid_off` and `canceled` are final. The status value is `canceled`, with one `l`.

| **From** | **To** | **When** | **Event** |
| --- | --- | --- | --- |
| — | `active` | You create the plan with `payment_method`. | `payment_plan.created`, then `payment_plan.activated` |
| `active` | `past_due` | The customer's account enters the late-payment warning. | `payment_plan.past_due` |
| `active`, `past_due` | `defaulted` | The customer's account reaches your failed-payment outcome. | `payment_plan.defaulted` |
| `defaulted` | `past_due` | Under the `past-due` outcome, the customer's account falls back from the outcome to the late-payment warning. | `payment_plan.past_due` |
| `past_due`, `defaulted` | `active` | The customer's account clears, or a payment with a principal portion rebuilds the remaining schedule while the account is clear. See **Failed Payments and Payment Plans** for the `cancel` outcome. | `payment_plan.reactivated` |
| `active`, `past_due`, `defaulted` | `paid_off` | The schedule ends with every installment paid, or a payoff payment is recorded. A `past_due` or `defaulted` plan whose schedule ends while an installment is still unpaid keeps its status until that installment is paid and the customer's account clears. | `payment_plan.completed` |
| `active`, `past_due`, `defaulted`, `awaiting_deposit` | `canceled` | You call `POST /payment-plans/{id}/cancel`. | `payment_plan.canceled` |

---

# Create a Plan

`POST /payment-plans` creates a plan and answers `201` with the Payment Plan object, in status `active`.

Request body:

```json
{
  "customer": "cus_739c07713fd4b68565a969cd",
  "total_amount": 210000,
  "down_payment_amount": 30000,
  "installment_amount": 20000,
  "interval": "monthly",
  "start_date": 1760400000,
  "remainder_placement": "first",
  "payment_method": "9f3a1c2e-7b45-4d18-a6c0-2f8e5b7d1a94"
}
```

| **Field** | **Required** | **Rules** |
| --- | --- | --- |
| `customer` | yes | An existing customer id, `cus_…`. |
| `total_amount` | yes | Integer cents, at least `1`. |
| `installment_count` | one of the two | Integer from `1` to `60`. |
| `installment_amount` | one of the two | Integer cents, at least `1`. Send exactly one of `installment_count` and `installment_amount`; ChaChing derives the other. |
| `down_payment_amount` | no | Integer cents, `0` or more and less than `total_amount`. Default `0`. |
| `interval` | no | `weekly`, `biweekly` or `monthly`. Default `monthly`. |
| `start_date` | no | Unix seconds: the due date of installment 1. It must fall on a later calendar day (UTC) than the request; a date on the day of the request is refused. Default: one interval after the day of the request. |
| `remainder_placement` | no | `first` or `last`. Default `first`. |
| `payment_method` | yes | The id of a payment method of this customer, as returned by the payment-methods endpoints. |

Always send `payment_method`. Creating a plan without it — terminal-deposit mode, where the customer pays installment 1 on your card terminal — is not available in this release: such a plan stays `awaiting_deposit`, is never charged, and can only be canceled. The request fields `terminal` and `reference` are rejected with `400`, and the plan's `deposit` object carries no terminal.

What happens when the plan is created:

- Before charging anything, ChaChing checks the schedule it registered against the expected installment dates: every date for a plan of up to 12 installments, and, for a longer plan, the first installment date (plus the second when the first installment carries the odd amount). A mismatch answers `500` `PAYMENT_PLAN_SCHEDULE_MISMATCH`, and no plan is created.
- When `down_payment_amount` is greater than `0`, ChaChing charges it immediately against `payment_method` and records it as installment sequence `0`, already paid. When the charge is declined, the request answers `402` `PAYMENT_PLAN_DOWN_PAYMENT_DECLINED` and no plan is created. When its outcome cannot be confirmed, the request answers `502` `PAYMENT_PLAN_DOWN_PAYMENT_UNCONFIRMED`, no plan is created, and the charge is not reversed because it can still settle: do not retry blindly — check the customer's invoices first.
- `payment_method` becomes the customer's default payment method, and the installments are charged against it.
- ChaChing sends `payment_plan.created` and `payment_plan.activated`, and, when there is a down payment, `payment_plan.installment_paid` for installment `0`.

## The Card on File

A plan with no down payment runs entirely on a card the customer saved earlier. ChaChing does not run a $0.00 card verification. It accepts the saved card only when the card was saved through an approved authorization that established it for automatic payments; otherwise the request answers `409` `PAYMENT_PLAN_CREDENTIAL_NOT_ESTABLISHED` and nothing is charged. Create the plan with a down payment — its charge establishes the card — or ask the customer to enter the card again. On the NMI gateway, a card added to the customer vault without a transaction is refused this way.

Payment Plans run on accounts whose payment gateway is Dejavoo or NMI. On any other gateway the request answers `400` `PAYMENT_PLAN_GATEWAY_NOT_SUPPORTED`.

## When a Plan Cannot Be Created

- Your failed-payment outcome (`subscription_state_on_payment_failure`) is `unpaid`: `409` `PAYMENT_PLAN_UNPAID_OUTCOME_NOT_SUPPORTED`.
- Your retry schedule (`payment_retries`) sums to less than 7 days: `409` `PAYMENT_PLAN_DUNNING_SCHEDULE_TOO_SHORT`.
- The customer's account is in the late-payment warning or in a failed-payment outcome, or it carries an invoice unpaid for one day or more: `409` `PAYMENT_PLAN_CUSTOMER_DELINQUENT`. Collect the balance first.
- The customer's billing state cannot be read: `502` `PAYMENT_PLAN_BILLING_ENGINE_UNAVAILABLE`. Nothing was created; the request is safe to retry.

Change the first two through `PUT /revenue-recovery`; see [Configure Failed-Payment Settings](./subscription-lifecycle.md).

## Duplicate Requests

`POST /payment-plans` and `POST /payment-plans/{id}/payments` accept an `Idempotency-Key` header of 1 to 255 printable ASCII characters with no spaces; any other value answers `400` `IDEMPOTENCY_KEY_INVALID`. A key is scoped to your account and to the endpoint, and it is kept for 24 hours.

- The same key with the same request returns the stored result — the same status code and body — with the response header `Idempotent-Replayed: true`. Nothing is created or charged again, and no webhook is sent again.
- The same key with a different request answers `422` `IDEMPOTENCY_KEY_REUSED`.
- The same key while the first request is still running answers `409` `IDEMPOTENCY_KEY_IN_PROGRESS`.
- When the first request never completed, the key answers `502` `IDEMPOTENCY_REQUEST_INCOMPLETE` after ten minutes. Read the plan before retrying with a new key.
- A request refused with a `4xx` other than `402` is not stored: fix it and retry with the same key.

---

# Read Plans and Installments

- `GET /payment-plans` lists your plans, newest first, with the page parameters `page` and `take` (`take` at most `50`). Filter with `search`, a URL-encoded JSON object: `{"customer": "cus_739c07713fd4b68565a969cd", "status": "past_due,defaulted"}`. `status` takes one status or a comma-separated list. A `search` value that is not valid JSON answers `400`.
- `GET /payment-plans/{id}` returns one plan.
- `GET /payment-plans/{id}/installments` returns the plan's installments in ascending `sequence` order, the down payment first when there is one.

An installment's `status` is `scheduled`, `paid`, `failed` or `canceled`. It reads `failed` after a declined automatic attempt and stays `failed` until it is paid. `attempt_count` counts the automatic charge attempts, retries included, and `next_payment_attempt` is the date of the next retry, or `null` when none is scheduled. An installment not yet billed reads `canceled` when the plan was canceled or paid off, or when a payment that reached the principal shortened the schedule and removed it.

The Invoice object, on the invoice endpoints and in every `invoice.*` webhook, carries `payment_plan` and `installment`. Both are `null` on an invoice that belongs to no payment plan, and an invoice raised for a payment plan reports `subscription: null`.

---

# Record Payments, Extra Payments and a Payoff

`POST /payment-plans/{id}/payments` records every payment made outside the automatic schedule and answers `201` with the Payment object.

```json
{
  "amount": 50000,
  "payment_method": "9f3a1c2e-7b45-4d18-a6c0-2f8e5b7d1a94",
  "external": false
}
```

| **Field** | **Required** | **Rules** |
| --- | --- | --- |
| `amount` | yes | Integer cents, at least `1`, and at most the plan's `remaining_balance`. On a `canceled` plan, at most what is still owed on its billed installments. |
| `payment_method` | no | The card to charge for this payment only; it must belong to the plan's customer. Default: the customer's default payment method. Ignored when `external` is `true`. |
| `external` | no | `true` records money you collected outside ChaChing; nothing is charged. Default `false`. |

The amount is applied first to the plan's billed installments that are not fully paid, oldest first — a smaller amount pays part of an installment — and then to the part of the plan not billed yet, the principal.

- **Extra payment.** An amount below `remaining_balance`. When it reaches the principal, the installment amount and the due dates do not change: the plan ends sooner, and its last installment is reduced to what is left.
- **Payoff.** An amount equal to `remaining_balance`. It pays every open installment, closes the rest of the schedule, and moves the plan to `paid_off`.
- **Several charges.** A payment spanning several billed installments, or billed installments and the principal, is charged as one charge per installment invoice plus one charge for the principal, in that order. When the first charge is declined the request answers `402` `PAYMENT_DECLINED` and nothing is applied. When a later charge is declined, the charges already approved stay applied and the request answers `201` with the `amount` actually applied. When a later charge's outcome cannot be confirmed, the request still answers `201`, but `amount` reads `0` even when earlier charges of the same payment were approved: the payment is recorded only once every one of its charges is resolved. Do not retry with a new `Idempotency-Key`. Wait for `payment_plan.payment_succeeded`, which ChaChing sends with the amount actually applied once it finishes recording the payment, and read the plan's installments when the event does not arrive.
- **The principal has no invoice.** The principal part of a payment belongs to no invoice: it does not appear on the invoice endpoints and sends no `invoice.*` webhook. `payment_plan.payment_succeeded` reports it.
- A payment on a `past_due` or `defaulted` plan is never refused because of its status. While an earlier payment's schedule rebuild is still being completed, a further payment answers `409` `PAYMENT_PLAN_REPLAN_PENDING`; retry it later.

Each recorded payment sends `payment_plan.payment_succeeded`, one `payment_plan.installment_paid` for every installment it paid in full, and `payment_plan.completed` when it is a payoff. A declined or refused payment sends none of them. When a request ends before its payment is fully recorded (for example a `502` `PAYMENT_PLAN_PAYMENT_UNCONFIRMED`, or a `201` whose later charge could not be confirmed), ChaChing finishes recording the payment later and sends these events then.

---

# Failed Payments and Payment Plans

Your retry schedule and your failed-payment outcome apply to the whole account — every subscription and every payment plan — and they are read and changed with `GET /revenue-recovery` and `PUT /revenue-recovery`, described in [Configure Failed-Payment Settings](./subscription-lifecycle.md). The two outcome fields take these exact values: `subscription_state_on_payment_failure` is `cancel`, `unpaid` or `past-due`, and `invoice_state_on_payment_failure` is `past-due` or `uncollectible`.

The late-payment warning and the failed-payment outcome are evaluated per customer, not per plan. An unpaid subscription invoice moves the customer's plans too, and an unpaid installment moves the customer's subscriptions.

- **Late-payment warning.** When the customer's account enters it, each of the customer's `active` plans changes to `past_due` and ChaChing sends `payment_plan.past_due`. Automatic charging continues.
- **Failed-payment outcome.** When the customer's account reaches your outcome, each `active` or `past_due` plan changes to `defaulted` and ChaChing sends `payment_plan.defaulted`. With `cancel`, the plan's schedule is cut at the outcome day: the installment already billed stays owed in full, no further installment is billed, and no automatic charge is attempted again. With `past-due`, the schedule keeps running and every installment is still charged on its due date.
- **Back to `active`.** When the customer's account clears, a `past_due` plan, and a `defaulted` plan under the `past-due` outcome, whose schedule is still running return to `active` and ChaChing sends `payment_plan.reactivated`. Under the `cancel` outcome, a `defaulted` plan whose schedule was cut stays `defaulted` until it is paid off, canceled, or a payment with a principal portion through `POST /payment-plans/{id}/payments` rebuilds its remaining schedule while the customer's account is clear.
- **Settings for accounts with Payment Plans.** On an account with Payment Plans enabled, or holding a plan that has not ended, `payment_retries` must sum to at least 7 days and `subscription_state_on_payment_failure` cannot be `unpaid`; `PUT /revenue-recovery` rejects either with `400`. With these rules the outcome day is day 8 or later.
- **The debt survives.** A `defaulted` plan and a `canceled` plan keep their outstanding balance. ChaChing never forgives a balance on its own: `uncollectible` is a reporting label, and you can still collect through `POST /payment-plans/{id}/payments`.

Do not confuse the plan status `past_due`, which means the late-payment warning, with the outcome value `past-due`. A customer who reaches the `past-due` outcome makes the plan `defaulted`, and ChaChing sends `payment_plan.defaulted`, not `payment_plan.past_due`.

For the subscription side of the same rules, see [Track subscription lifecycle and failed payments](./subscription-lifecycle.md) and [Handle failed subscription payments](../Using%20Chaching/Subscriptions/failed-payments.md).

---

# Cancel a Plan

`POST /payment-plans/{id}/cancel` answers `200` with the plan in status `canceled` and `canceled_at` set, and ChaChing sends `payment_plan.canceled`. Installments not yet billed are removed at the end of the period already billed and are never charged. Cancellation is not a refund and does not forgive debt: amounts already invoiced stay collectible through `POST /payment-plans/{id}/payments`, and `remaining_balance` shows what is still owed.

This endpoint is the only way to end a plan early. A payment plan is not a subscription: the subscription endpoints answer `404` for it.

---

# Error Codes

Errors use the standard body `{ "statusCode", "message", "error", "timestamp", "path" }`. Branch on `statusCode` and `error`; `message` can change. A request body that fails field validation — a value below its minimum such as `total_amount: 0` or `installment_count: 0`, a negative `down_payment_amount`, a value of the wrong type, or an unknown field such as `terminal` — and a `search` value that is not valid JSON are rejected with `400` and a body that carries no `error` field; `message` describes the problem.

| **Status** | **`error`** | **When** |
| --- | --- | --- |
| `400` | `PAYMENT_PLAN_SCHEDULE_INVALID` | Both or neither of `installment_count` and `installment_amount` were sent, `installment_count` is above `60` or larger than the amount to finance allows, `installment_amount` is larger than the amount to finance, or the amounts do not form a valid schedule. |
| `400` | `PAYMENT_PLAN_AMOUNT_INVALID` | `down_payment_amount` is not below `total_amount`, or a payment's `amount` is below `1`. |
| `400` | `PAYMENT_PLAN_INTERVAL_INVALID` | `interval` is not `weekly`, `biweekly` or `monthly`. |
| `400` | `PAYMENT_PLAN_START_DATE_INVALID` | `start_date` is not after the day of the request. |
| `400` | `PAYMENT_PLAN_DOWN_PAYMENT_NOT_ALLOWED` | A down payment was sent without `payment_method`. |
| `400` | `PAYMENT_PLAN_GATEWAY_NOT_SUPPORTED` | The account's payment gateway does not support Payment Plans. |
| `400` | `PAYMENT_PLAN_AMOUNT_EXCEEDS_BALANCE` | A payment is larger than what the plan accepts. Nothing is charged. |
| `400` | `PAYMENT_METHOD_REQUIRED` | A payment was sent with no `payment_method`, it is not external, and the customer has no default payment method. |
| `400` | `IDEMPOTENCY_KEY_INVALID` | The `Idempotency-Key` header is malformed. |
| `402` | `PAYMENT_PLAN_DOWN_PAYMENT_DECLINED` | The down payment was declined. No plan was created. |
| `402` | `PAYMENT_DECLINED` | The first charge of a payment was declined. Nothing was applied. |
| `403` | `PAYMENT_PLANS_NOT_ENABLED` | Payment Plans are not enabled for this account. |
| `404` | `PAYMENT_PLAN_NOT_FOUND` | The plan id does not resolve for this account. |
| `404` | `CUSTOMER_NOT_FOUND` | The customer id does not resolve for this account. |
| `404` | `PAYMENT_METHOD_NOT_FOUND` | The payment method does not resolve, or it does not belong to the customer. |
| `409` | `PAYMENT_PLAN_CREDENTIAL_NOT_ESTABLISHED` | The saved card was not established for automatic payments. Create the plan with a down payment, or ask the customer to enter the card again. |
| `409` | `PAYMENT_PLAN_UNPAID_OUTCOME_NOT_SUPPORTED` | Your failed-payment outcome is `unpaid`. |
| `409` | `PAYMENT_PLAN_DUNNING_SCHEDULE_TOO_SHORT` | Your `payment_retries` sum to less than 7 days. |
| `409` | `PAYMENT_PLAN_CUSTOMER_DELINQUENT` | The customer's account is in the late-payment warning or a failed-payment outcome, or carries an invoice unpaid for one day or more. |
| `409` | `PAYMENT_PLAN_NOT_PAYABLE` | The plan is `awaiting_deposit` or `paid_off`. |
| `409` | `PAYMENT_PLAN_NOT_CANCELABLE` | The plan is already `paid_off` or `canceled`. |
| `409` | `PAYMENT_PLAN_BUSY` | Another payment or a cancellation on this plan is in progress. Retry shortly. |
| `409` | `PAYMENT_PLAN_INSTALLMENT_BILLING_IN_PROGRESS` | The payment reaches the principal and the next installment is due within 24 hours or is being billed. Nothing is charged. |
| `409` | `PAYMENT_PLAN_PAYMENT_PENDING_CONFIRMATION` | An earlier payment on this plan is still being confirmed. Nothing is charged; retry later. |
| `409` | `PAYMENT_PLAN_REPLAN_PENDING` | An earlier payment's schedule rebuild is still being completed. Nothing is charged; retry later. |
| `409` | `PAYMENT_PLAN_PAYMENT_NOT_ATTEMPTED` | The first charge was not attempted. Nothing is charged. |
| `409` | `IDEMPOTENCY_KEY_IN_PROGRESS` | A request with the same `Idempotency-Key` is still running. |
| `422` | `IDEMPOTENCY_KEY_REUSED` | The `Idempotency-Key` was already used with a different request. |
| `500` | `PAYMENT_PLAN_SCHEDULE_MISMATCH` | The registered schedule did not match the expected installment dates. No plan was created. |
| `502` | `PAYMENT_PLAN_DOWN_PAYMENT_UNCONFIRMED` | The down payment's outcome could not be confirmed. No plan was created and the charge is not reversed. Do not retry blindly: check the customer's invoices first. |
| `502` | `PAYMENT_PLAN_PAYMENT_UNCONFIRMED` | The first charge's outcome could not be confirmed. It is not reversed; read the plan before retrying. |
| `502` | `PAYMENT_PLAN_BILLING_ENGINE_UNAVAILABLE` | Billing data could not be read. Nothing was created or charged; safe to retry. |
| `502` | `PAYMENT_PLAN_CANCEL_INCOMPLETE` | The cancellation did not complete. The plan keeps its status and installments, but part of its schedule can already be stopped. Repeat the request: a repeat completes the rest. |
| `502` | `IDEMPOTENCY_REQUEST_INCOMPLETE` | The first request with this `Idempotency-Key` never completed. |

---

# Payment Plan Events

Payment Plans send ten webhook event types on the same endpoints, envelope, signature and retry policy as every other ChaChing event. Their payloads are described in [Webhooks](./webhook.md).

| **Event** | **Sent when** | **`data`** |
| --- | --- | --- |
| `payment_plan.created` | A plan is created. | Payment Plan |
| `payment_plan.activated` | A plan starts running. A plan created with `payment_method` sends it right after `payment_plan.created`. | Payment Plan |
| `payment_plan.installment_paid` | An installment is paid by its automatic charge or a retry, by a payment through `POST /payment-plans/{id}/payments`, or as the down payment at creation (sequence `0`). | Installment |
| `payment_plan.installment_failed` | An automatic charge attempt on an installment is declined, once per attempt, retries included. | Installment |
| `payment_plan.past_due` | The plan changes to `past_due`. | Payment Plan |
| `payment_plan.defaulted` | The plan changes to `defaulted`. | Payment Plan |
| `payment_plan.reactivated` | The plan returns to `active` from `past_due` or `defaulted`. | Payment Plan |
| `payment_plan.completed` | The plan changes to `paid_off`. | Payment Plan |
| `payment_plan.canceled` | The plan is canceled. | Payment Plan |
| `payment_plan.payment_succeeded` | `POST /payment-plans/{id}/payments` records a payment. | Payment |

Read the plan with `GET /payment-plans/{id}` whenever you need its current state: the plan and installment endpoints are the source of truth, and deliveries can arrive in a different order than the changes happened.
