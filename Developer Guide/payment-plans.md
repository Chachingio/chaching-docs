# Payment Plans

A payment plan collects a fixed total amount from one customer in a fixed number of scheduled installments. ChaChing stores the plan, charges each installment automatically on its due date against the plan's payment method, retries a declined charge on your account's retry schedule, and reports every change over webhooks. There is no interest. The installments always add up to the total agreed when the plan was created; on an account that charges through iPOSpays (Dejavoo), each credit card charge of a plan also carries the credit card surcharge configured for your account in the iPOSpays S.T.E.A.M portal, recorded separately from the plan amounts; debit card charges carry none (see [Credit Card Surcharges](#credit-card-surcharges)).

Payment Plans are available through the API. They are enabled per merchant account by ChaChing and are off by default. While they are off, every `/payment-plans` endpoint answers `403` with the error code `PAYMENT_PLANS_NOT_ENABLED` and the message `Payment Plans are not enabled for this account. Contact Chaching to enable them.` Accounts created through the platform API have payment plans enabled.

Merchants also work with payment plans in the ChaChing dashboard. On an account that charges through iPOSpays (Dejavoo), the dashboard collects the deposit of a plan — its down payment, or its first installment when it has none — on one of the account's registered terminals and takes payments on a terminal, through the same deposit and payment flows this page describes. Nothing changes for an integration: every endpoint, object and webhook on this page works as documented.

Every endpoint on this page authenticates with the account API key in the `cc-api-key` header, like every other public endpoint. Amounts are integers in cents, dates and timestamps are Unix seconds, and field names are `snake_case`. For every request and response schema, see the [API Reference](./api.json). For the webhook payloads, see [Webhooks](./webhook.md). Every endpoint on this page requires the account secret API key (`sk_`), or a platform key with `Chaching-Account` (see [Platforms](./platforms.md)), in the `cc-api-key` header. A publishable key (`pk_`) answers `403` `SECRET_API_KEY_REQUIRED` on every endpoint.

---

# Concepts

| **Term** | **Meaning** |
| --- | --- |
| Payment plan | An agreement to collect `total_amount` from one customer in a fixed number of installments. Its id starts with `pp_`. |
| Installment | One scheduled payment of a plan. Its id starts with `ppi_`. Each installment has a sequence number, a due date, an amount and a status, and it produces its own invoice when it is billed. |
| Down payment | An amount paid before any installment: charged when a plan is created with `payment_method`, or collected on one of your Dejavoo terminals for a plan created without one. It is recorded as installment sequence `0`, and it is not counted in `installment_count`. |
| Payment | One attempt to collect money on a plan, successful or not: a payment you record with `POST /payment-plans/{id}/payments`, an automatic installment charge or retry, a down payment, a terminal deposit, an invoice payment or a charge of a new card. Its id starts with `ppp_`, its `origin` says where it came from and its `status` whether it collected money. |

The amounts always add up: `down_payment_amount` plus the sum of the scheduled installment amounts equals `total_amount`. When the amount left after the down payment does not divide evenly, one installment carries the odd amount: installment 1 with `remainder_placement: "first"` (the default), or the last installment with `"last"`. Every other installment carries exactly `installment_amount`.

For example, $2,100.00 at $200.00 a month with no down payment and the remainder first becomes 11 installments: installment 1 of $100.00, then 10 installments of $200.00. In API terms: `total_amount: 210000`, `installment_amount: 20000`, `remainder_placement: "first"`, and the plan reports `installment_count: 11`.

---

# Plan Statuses

| **Status** | **Meaning** |
| --- | --- |
| `active` | The plan is running. Installments are billed and charged on their due dates. |
| `past_due` | The customer's account is in the late-payment warning. Automatic charging continues. |
| `defaulted` | The customer's account reached your failed-payment outcome. What still runs depends on that outcome; see **Failed Payments and Payment Plans** below. |
| `paid_off` | The plan is paid in full. Nothing more is charged. A refund made after that leaves it `paid_off`, with the refunded amount still owed in `amount_due`. |
| `canceled` | You canceled the plan. Installments not yet billed are removed; amounts already invoiced stay collectible. |
| `awaiting_deposit` | The plan was created without a payment method and waits for its deposit — the down payment when the plan has one, else its first installment. The customer pays it on one of your Dejavoo terminals (see **Collect the First Installment on a Terminal**), or you record it as collected outside ChaChing (see **Record a Deposit Collected Outside Chaching**). |

`paid_off` and `canceled` are final. A refund does not change a plan's status: a `paid_off` or `canceled` plan stays in its status and shows the refunded amount still owed in `amount_due`, which `POST /payment-plans/{id}/payments` collects. The status value is `canceled`, with one `l`. A plan in `awaiting_deposit` accepts only a payment with `external: true` of at least the deposit through `POST /payment-plans/{id}/payments`: it records the deposit collected outside ChaChing and activates the plan (see **Record a Deposit Collected Outside Chaching**). A payment without `external: true` answers `409` `PAYMENT_PLAN_NOT_PAYABLE`, and an external payment below the deposit answers `400` `PAYMENT_PLAN_AMOUNT_BELOW_DEPOSIT`.

| **From** | **To** | **When** | **Event** |
| --- | --- | --- | --- |
| — | `active` | You create the plan with `payment_method`. | `payment_plan.created`, then `payment_plan.activated` |
| — | `awaiting_deposit` | You create the plan without `payment_method`. | `payment_plan.created` |
| `awaiting_deposit` | `active` | The deposit collected on your terminal is confirmed and its card is saved. A plan with one installment and no down payment goes to `paid_off` instead. | `payment_plan.activated`, `payment_plan.installment_paid` and `payment_plan.payment_succeeded`, plus `payment_plan.completed` for `paid_off` |
| `awaiting_deposit` | `active` | You record the deposit collected outside with `external: true`. A payment of the whole remaining balance goes to `paid_off` instead. | `payment_plan.activated`, `payment_plan.installment_paid` and `payment_plan.payment_succeeded`, plus `payment_plan.completed` for `paid_off` |
| `active` | `past_due` | The customer's account enters the late-payment warning. | `payment_plan.past_due` |
| `active`, `past_due` | `defaulted` | The customer's account reaches your failed-payment outcome. | `payment_plan.defaulted` |
| `defaulted` | `past_due` | Under the `past-due` outcome, the customer's account falls back from the outcome to the late-payment warning. | `payment_plan.past_due` |
| `past_due`, `defaulted` | `active` | The customer's account clears, or a payment with a principal portion rebuilds the remaining schedule while the account is clear. See **Failed Payments and Payment Plans** for the `cancel` outcome. | `payment_plan.reactivated` |
| `active`, `past_due`, `defaulted` | `paid_off` | The schedule ends with every installment paid, a payoff payment is recorded, or the last unpaid installment is paid through the hosted invoice page, the dashboard or `PUT /invoices/{id}`, or settled by credit on the customer's account. An `active`, `past_due` or `defaulted` plan whose schedule ends while an installment is still unpaid keeps its status until that installment is paid, and is never `paid_off` before then: it changes as soon as the payment goes through ChaChing, credit settles it, or the customer's automatic charge or a retry collects it. A plan that owes a refunded amount (`amount_due` above `0`) is not `paid_off` either: it changes when that amount is collected with `POST /payment-plans/{id}/payments` or forgiven. A written-off installment is not owed, so a plan whose remaining installments are paid or written off completes as `paid_off`. | `payment_plan.completed` |
| `active`, `past_due`, `defaulted`, `awaiting_deposit` | `canceled` | You call `POST /payment-plans/{id}/cancel`. | `payment_plan.canceled` |

---

# Create a Plan

`POST /payment-plans` creates a plan and answers `201` with the Payment Plan object, in status `active`. A plan created without `payment_method` answers in status `awaiting_deposit` instead; see **Collect the First Installment on a Terminal**.

Request body:

```json
{
  "customer": "cus_739c07713fd4b68565a969cd",
  "external_id": "PLAN-2026-0117",
  "description": "Orthodontic treatment, upper and lower braces",
  "total_amount": 210000,
  "down_payment_amount": 30000,
  "installment_amount": 20000,
  "interval": "monthly",
  "start_date": 1811808000,
  "remainder_placement": "first",
  "payment_method": "9f3a1c2e-7b45-4d18-a6c0-2f8e5b7d1a94"
}
```

Replace start_date with a day after today.

| **Field** | **Required** | **Rules** |
| --- | --- | --- |
| `customer` | yes | An existing customer id, `cus_…`. |
| `description` | no | A short text describing what the plan pays for. ChaChing shows it to the customer on the plan's receipts and failed-payment emails. Up to 255 characters, with no line breaks or other control characters. Leading and trailing spaces are removed, and a value that is empty or only spaces is stored as no description. With an `Idempotency-Key`, sending such a value and leaving `description` out are the same request; adding a non-empty `description` to a request you retry with the same key, or changing it, answers `422`. |
| `external_id` | no | Your own identifier for this plan, for example the plan id in your system. Up to 255 Unicode characters (code points; an emoji counts as one), with no control characters. Leading and trailing spaces, tabs and line breaks are removed, and a blank value is stored as none. Unique among all payment plans of your account, whatever their status: a second create with the same value answers `409` `PAYMENT_PLAN_EXTERNAL_ID_EXISTS`. It cannot be changed after the plan is created. See **External IDs**. |
| `total_amount` | yes | Integer cents, at least `1`. |
| `installment_count` | one of the two | Integer from `1` to `60`. |
| `installment_amount` | one of the two | Integer cents, at least `1`. Send exactly one of `installment_count` and `installment_amount`; ChaChing derives the other. |
| `down_payment_amount` | no | Integer cents, `0` or more and less than `total_amount`. Default `0`. |
| `interval` | no | `weekly`, `biweekly`, `monthly` or `annual`. Default `monthly`. On test environments, such as stage, `daily` is also accepted; production refuses `daily` with `400` `PAYMENT_PLAN_INTERVAL_INVALID`. An `annual` plan cannot start on 29 February. |
| `start_date` | no | Unix seconds: the due date of installment 1. Only the calendar date is used and the plan returns 00:00:00 UTC of that date. It must be later than today's date both in UTC and in the account time zone, and, for a start within the next two UTC days, late enough that installment 1 does not bill within an hour of the request; otherwise the request answers `400` `PAYMENT_PLAN_START_DATE_INVALID` and the message names the earliest valid date. Default: one interval after today in the account time zone, never earlier than the day after the current UTC date. |
| `remainder_placement` | no | `first` or `last`. Default `first`. |
| `payment_method` | no | The id of a payment method of this customer, as returned by the payment-methods endpoints. Leave it out to collect the deposit on a terminal: the down payment when `down_payment_amount` is above `0`, else installment 1. |

Without `payment_method`, the plan opens in `awaiting_deposit`, and its deposit is collected on one of your Dejavoo terminals: the down payment when `down_payment_amount` is above `0`, else installment 1. Nothing is charged and nothing is billed until the deposit is collected. With a down payment, the terminal collects that amount only, and installment 1 is charged on `start_date` to the card the terminal saved. Collect the deposit before `start_date`; installment 1 is due on `start_date` whenever the deposit is collected. A plan in this mode runs only on an account whose payment gateway is Dejavoo. The request fields `terminal` and `reference` are rejected with `400`: you choose the terminal when you collect the deposit, and ChaChing creates the plan's `reference`.

What happens when a plan is created with `payment_method`:

- Before charging anything, ChaChing checks the schedule it registered against the expected installment dates: every date for a plan of up to 12 installments, and, for a longer plan, the first installment dates (the exact set depends on the plan). A date that cannot be projected because the customer has another subscription that bills often, such as a daily one, is not checked, and the plan is still created. A date more than 36 months after the day the check runs (UTC) is not checked either, and the plan is still created. A mismatch answers `500` `PAYMENT_PLAN_SCHEDULE_MISMATCH`, and no plan is created.
- When `down_payment_amount` is greater than `0`, ChaChing charges it immediately against `payment_method` and records it as installment sequence `0`, already paid. When the charge is declined, the request answers `402` `PAYMENT_PLAN_DOWN_PAYMENT_DECLINED` and no plan is created. When its outcome cannot be confirmed, the request answers `502` `PAYMENT_PLAN_DOWN_PAYMENT_UNCONFIRMED`, no plan is created, and the charge is not reversed because it can still settle. The response's `down_payment_invoice_id` is the id of that down-payment invoice; once its charge has settled, send the same request again, with the same `Idempotency-Key` or a new one, and ChaChing handles the invoice as the next item describes.
- Before it charges a down payment, ChaChing looks for an earlier down-payment invoice of the same customer and the same amount that no plan uses, which is what a create that did not finish leaves behind. When that invoice was paid with the same `payment_method` and was not refunded, the new plan uses that payment: nothing is charged again, and installment `0` carries that earlier invoice in `invoice` and the time of that payment in `paid_at`. When its charge failed, ChaChing voids it and charges the down payment again. When that invoice is still open without a confirmed outcome, when it was settled with a different payment method or without a card payment, when more than one such invoice was paid, or when a failed invoice cannot be voided, the request answers `409` `PAYMENT_PLAN_DOWN_PAYMENT_PENDING`, no plan is created and nothing is charged. From the next UTC day, an earlier down-payment invoice that is still unpaid answers `409` `PAYMENT_PLAN_CUSTOMER_DELINQUENT` instead, until it is paid or voided. An earlier down-payment invoice of a different amount is not matched, so check the customer's invoices before you change `down_payment_amount` after a `502`.
- `payment_method` is the plan's card: every installment is charged to it, whatever the customer's default payment method is. It becomes the customer's default payment method only when the customer has none.
- ChaChing sends `payment_plan.created` and `payment_plan.activated`, and, when there is a down payment, `payment_plan.installment_paid` for installment `0` and `payment_plan.payment_succeeded` for its payment.
- When there is a down payment, ChaChing emails the customer a receipt for it. See **Emails Your Customer Receives**.

## The Card on File

A plan with no down payment runs entirely on a card the customer saved earlier. ChaChing does not run a $0.00 card verification. It accepts the saved card only when the card was saved through an approved authorization that established it for automatic payments; otherwise the request answers `409` `PAYMENT_PLAN_CREDENTIAL_NOT_ESTABLISHED` and nothing is charged. Create the plan with a down payment — its charge establishes the card — or ask the customer to enter the card again. On the NMI gateway, a card added to the customer vault without a transaction is refused this way.

On a Dejavoo account, a card saved while the customer pays a hosted invoice cannot be charged again: create the plan with that card and the down payment is declined. Save the card through the subscribe checkout or collect the deposit on a terminal.

Payment Plans run on accounts whose payment gateway is Dejavoo or NMI. On any other gateway the request answers `400` `PAYMENT_PLAN_GATEWAY_NOT_SUPPORTED`.

### Change a plan's card

`PATCH /payment-plans/{id}` replaces the card of a plan, or removes it, with the `payment_method` field.

```bash
curl -X PATCH https://stage-api.chaching.io/payment-plans/$PLAN_ID \
  -H "cc-api-key: $CHACHING_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "payment_method": "628423f5-980e-45f0-97e0-f11b4f732d38"
  }'
```

The response is the plan with the new card:

```json
{
  "id": "pp_sZoJQteM01TAGaOSFN1z35re",
  "object": "payment_plan",
  "external_id": null,
  "customer": "cus_0377afcde4bc99362f542b4a",
  "customer_external_id": null,
  "description": "Orthodontic treatment, upper and lower braces",
  "status": "active",
  "currency": "USD",
  "total_amount": 660,
  "down_payment_amount": 0,
  "installment_amount": 110,
  "installment_count": 6,
  "interval": "monthly",
  "start_date": 1796169600,
  "remainder_placement": "first",
  "remaining_balance": 660,
  "amount_refunded": 0,
  "amount_due": 0,
  "amount_written_off": 0,
  "installments_paid": 0,
  "installments_remaining": 6,
  "next_due_date": 1796169600,
  "payment_method": "628423f5-980e-45f0-97e0-f11b4f732d38",
  "gateway": "dejavoo",
  "reference": null,
  "deposit": null,
  "created": 1795643587,
  "ended_at": null,
  "canceled_at": null
}
```

Returns the Payment Plan object.

- `payment_method` is the id of a saved card of the plan's customer, saved with the payment gateway of the plan and set up for automatic payments, or `null` to remove the plan's card. It is the only field the request accepts; any other field answers `400`.
- The plan must be `active`, `past_due` or `defaulted`. A plan in `awaiting_deposit` takes its card when its deposit is collected; a plan in `canceled` or `paid_off` no longer takes a card.
- A request without `payment_method`, or with the card the plan already has, answers `200` with the plan unchanged: nothing is changed, charged or sent, so repeating a request is safe.
- The customer's default payment method does not change, except that the new card becomes the default when the customer has none.
- With `payment_method` `null`, the plan has no card: each installment is invoiced on its due date and nothing is charged automatically, as described in **A plan without a card** under **Failed Payments and Payment Plans**. On a deactivated account, `null` is the only value the request accepts.
- ChaChing emits one `payment_plan.payment_method_updated` event per change.

| **Status** | **Error code** | **When** | **What to do** |
| --- | --- | --- | --- |
| `400` | none | A field other than `payment_method` was sent, or `payment_method` is neither a payment method id nor `null`. The body has no `error` field; `message` describes the problem. | Send only `payment_method`. |
| `403` | `PAYMENT_PLANS_NOT_ENABLED` | Payment Plans are not enabled for this account. | Contact Chaching to enable them. |
| `404` | `PAYMENT_PLAN_NOT_FOUND` | The plan id does not resolve for this account. | Check the id. |
| `404` | `PAYMENT_METHOD_NOT_FOUND` | The payment method does not resolve, or it does not belong to the plan's customer. | Send a payment method returned by the payment-methods endpoints for this customer. |
| `409` | `PAYMENT_PLAN_PAYMENT_METHOD_NOT_CHANGEABLE` | The plan is `awaiting_deposit`, `canceled` or `paid_off`. | For a plan in `awaiting_deposit`, collect its deposit first. A closed plan keeps its card. |
| `409` | `ACCOUNT_DEACTIVATED` | The account is deactivated and the request names a card. | Send `null`, or reactivate the account first. |
| `409` | `PAYMENT_PLAN_PAYMENT_METHOD_NOT_SUPPORTED` | The card is not saved with the payment gateway of this plan. | Choose a card saved with the plan's gateway. |
| `409` | `PAYMENT_PLAN_CREDENTIAL_NOT_ESTABLISHED` | The saved card was not set up for automatic payments. This includes a card that was used for a one-time payment and has no stored credential. | Save the card again through a terminal payment with `save_payment_method`, or keep the plan's current card. |
| `409` | `PAYMENT_PLAN_BUSY` | Another operation on this plan is in progress. | Retry shortly. |
| `409` | `PAYMENT_METHOD_CHANGE_IN_PROGRESS` | Another change to this customer's payment methods is in progress. | Try again. |
| `500` | none | The change could not be completed. The plan's card is unchanged. | Read the plan, then send the request again. |
| `502` | `PAYMENT_PLAN_BILLING_ENGINE_UNAVAILABLE` | The change could not be written to the billing engine, or the write could not be confirmed. The plan keeps the card that `GET /payment-plans/{id}` shows. | Read the plan to see its card. Send the request again when it is not the card you asked for. |

### What is charged when you change the card

Replacing the card makes ChaChing try to charge the new card once for each installment that is billed and not yet paid. It makes no attempt in the cases the rules below describe. ChaChing reports each charge like an automatic installment payment: `invoice.payment_succeeded`, `payment_plan.installment_paid` and `payment_plan.payment_succeeded`, and the receipt email to the customer, when it is approved; `invoice.payment_failed`, `payment_plan.installment_failed` and `payment_plan.payment_failed`, and the failed-payment email to the customer, when it is declined. A declined charge is not retried automatically and the installment stays open. Collect it with a payment on the plan, on the hosted invoice page, or by giving the plan another card. A retry that was scheduled on the previous card does not run. The installments that are not billed yet are charged on the new card when they are billed.

These rules limit the charges:

- An installment whose invoice already has a payment in progress gets no attempt from the change. The result of that payment settles it.
- If the plan's card is changed again while the charges are being made, the remaining installments are not charged on the replaced card.
- An installment is attempted once per card by these changes. When an earlier card change already made a failed attempt with a card, choosing that same card again makes no new attempt. Charge it with `POST /payment-plans/{id}/payments`.
- An installment whose invoice is not open with a balance to collect gets no attempt.
- When the checks ChaChing makes before a charge cannot be completed or do not allow it at that moment, no attempt is made and the installment stays open. Collect it with `POST /payment-plans/{id}/payments`.

Any charge attempt is made before the request answers `200`, and the request answers `200` whether or not an attempt was made; the results are recorded and announced when the billing engine reports them, which can be after the request answers. Read them from the events, from `GET /payment-plans/{id}/installments` and from `GET /invoices/{id}`. Removing the card charges nothing.

A payment of a plan's installment invoice made directly in the billing engine, on a card that is not the plan's card, is refused: it is never moved to another card.

A payment plan's installment is charged only to the plan's card. When automatic payments are turned back on for a customer, an installment that was waiting is not charged to another card: ChaChing's payment processors refuse it, and it stays open until it is paid on the hosted invoice page, recorded on the plan, or charged after you give the plan a card.

## When a Plan Cannot Be Created

- Your failed-payment outcome (`subscription_state_on_payment_failure`) is `unpaid`: `409` `PAYMENT_PLAN_UNPAID_OUTCOME_NOT_SUPPORTED`.
- Your retry schedule (`payment_retries`) sums to less than 7 days: `409` `PAYMENT_PLAN_DUNNING_SCHEDULE_TOO_SHORT`.
- The customer's account is in the late-payment warning or in a failed-payment outcome, or it carries an invoice unpaid for one day or more: `409` `PAYMENT_PLAN_CUSTOMER_DELINQUENT`. Collect the balance first.
- The request has `payment_method` and a down payment, and automatic payments are turned off for the customer in the billing engine: `409` `PAYMENT_PLAN_AUTO_PAY_OFF`. Create the plan without a down payment.
- The customer was already invoiced through a date on or after `start_date`, because an invoice was generated ahead of time for the customer's account: `409` `PAYMENT_PLAN_ACCOUNT_INVOICED_AHEAD`. The message names that date; choose a `start_date` after it.
- The customer's billing state cannot be read: `502` `PAYMENT_PLAN_BILLING_ENGINE_UNAVAILABLE`. Nothing was created; the request is safe to retry with a new `Idempotency-Key`.

Change the first two through `PUT /revenue-recovery`; see [Configure Failed-Payment Settings](./subscription-lifecycle.md).

## Duplicate Requests

`POST /payment-plans` and `POST /payment-plans/{id}/payments` accept an `Idempotency-Key` header of 1 to 255 printable ASCII characters with no spaces, and the two refund operations require one (see **Refund a Payment**); any other value answers `400` `IDEMPOTENCY_KEY_INVALID`. A key is scoped to your account and to the endpoint, and it is kept for 24 hours.

- The same key with the same request returns the stored result — the same status code and body — with the response header `Idempotent-Replayed: true`. Nothing is created or charged again, and no webhook is sent again.
- The same key with a different request answers `422` `IDEMPOTENCY_KEY_REUSED`.
- The same key while the first request is still running answers `409` `IDEMPOTENCY_KEY_IN_PROGRESS`.
- When the first request never completed, the key answers `502` `IDEMPOTENCY_REQUEST_INCOMPLETE` after ten minutes. Read the plan before retrying with a new key.
- A request refused with a `4xx` other than `402` is not stored: fix it and retry with the same key.
- A `402`, `500` or `502` answer is stored and replayed for the same key for 24 hours, so a retry after one of them needs a new key. The exceptions are `502` `PAYMENT_PLAN_DOWN_PAYMENT_UNCONFIRMED` and `502` `PAYMENT_PLAN_EXTERNAL_ID_CHECK_FAILED`: they are not stored, and the same key is evaluated again on the next request.

---

# Register a Dejavoo Terminal

Moved to [Payment Plans on a Terminal](./payment-plans-terminal.md#register-a-dejavoo-terminal).

---

# Collect the First Installment on a Terminal

Moved to [Payment Plans on a Terminal](./payment-plans-terminal.md#collect-the-first-installment-on-a-terminal).

## Deposit Attempt States

Moved to [Payment Plans on a Terminal](./payment-plans-terminal.md#deposit-attempt-states).

---

# Record a Deposit Collected Outside Chaching

When the customer pays the deposit of a plan in `awaiting_deposit` outside ChaChing — cash, a check, or a card machine that is not one of your registered terminals — record it with `POST /payment-plans/{id}/payments` and `external: true`. Nothing is charged. The plan becomes `active`, or `paid_off` when the payment covers the whole remaining balance, and the request answers `201` with the Payment object.

```json
{
  "amount": 10000,
  "external": true,
  "payment_method": "9f3a1c2e-7b45-4d18-a6c0-2f8e5b7d1a94"
}
```

| **Field** | **Rules** |
| --- | --- |
| `amount` | Integer cents, at least the plan's `deposit.amount` and at most its `remaining_balance`. The deposit is recorded whole; any amount above it is applied to the part of the plan not billed yet and shortens the plan from its end. The whole `remaining_balance` pays the plan off. |
| `external` | `true`. A payment without it answers `409` `PAYMENT_PLAN_NOT_PAYABLE`. |
| `payment_method` | No. A saved card of the plan's customer, saved with your account's Dejavoo gateway and set up for automatic payments. It becomes the plan's card, and the remaining installments are charged to it automatically. It becomes the customer's default payment method only when the customer has none; the customer's subscriptions and other payment plans keep their own cards. |

**Without `payment_method`:**

- The plan is activated without a card, and its `payment_method` reads `null`, whether or not the customer has a default payment method. Each remaining installment is invoiced on its due date and nothing is charged automatically. A card that later becomes the customer's default payment method does not change that: the plan stays without a card until you give it one with `PATCH /payment-plans/{id}`. Collect each installment with `POST /payment-plans/{id}/payments` (`external: true` for money you collected, or a terminal payment), or on the hosted invoice page. `PUT /invoices/{id}` does not collect it: for an installment invoice of a plan without a card it answers `409` `PAYMENT_PLAN_PAYMENT_METHOD_REQUIRED`. An installment left unpaid moves the plan to `past_due` and `defaulted` on the schedule of your revenue recovery settings, and ChaChing sends no failed-payment notice for an automatic charge that was never attempted.

**Before you record it:**

- A plan whose latest terminal deposit attempt is not resolved (`in_flight`, `resolving` or `approved_no_credential`) answers `409` `PAYMENT_PLAN_DEPOSIT_IN_FLIGHT`. Resolve the attempt with `POST /payment-plans/{id}/deposit`, or force-close it, first.
- After a force-closed attempt, or an attempt in `partial_approval`, check the sale in your Dejavoo (iPOSpays) account. When money was taken, void or refund it there before you record the deposit, so the customer does not pay twice.

**What ChaChing sends**, emitted in this order (deliveries can arrive in any order): `invoice.payment_succeeded` for the deposit invoice, `invoice.payment_succeeded` for the invoice of the amount above the deposit when there is one, `payment_plan.activated`, `payment_plan.installment_paid` for the deposit (sequence `0` for a down payment, else sequence `1`), `payment_plan.payment_succeeded`, and `payment_plan.completed` when the payment paid the plan off. The customer receives the `Payment recorded` email. The plan's `deposit` object then reads `status: "collected"`, `external: true`, `transaction_reference: null` and `collected_at` set to the time of the request.

| **Status** | **Error code** | **When** |
| --- | --- | --- |
| `400` | `PAYMENT_PLAN_AMOUNT_BELOW_DEPOSIT` | `amount` is below the deposit. |
| `400` | `PAYMENT_PLAN_AMOUNT_EXCEEDS_BALANCE` | `amount` is above the remaining balance. |
| `404` | `PAYMENT_METHOD_NOT_FOUND` | `payment_method` does not resolve, or does not belong to the customer. |
| `409` | `ACCOUNT_DEACTIVATED` | The account is deactivated and the request names a card. On a deactivated account the deposit is recorded only without a card. |
| `409` | `PAYMENT_PLAN_PAYMENT_METHOD_NOT_SUPPORTED` | `payment_method` is not a card saved with your account's Dejavoo gateway. |
| `409` | `PAYMENT_PLAN_CREDENTIAL_NOT_ESTABLISHED` | The card named in `payment_method` was not set up for automatic payments. Ask the customer to enter the card again, or record the deposit without a card. |
| `409` | `PAYMENT_PLAN_DEPOSIT_IN_FLIGHT` | A terminal deposit attempt on the plan is unresolved. |
| `409` | `PAYMENT_PLAN_START_DATE_TOO_SOON` | The first installment charged after the deposit bills within 1 hour. Cancel the plan and create it again with a later start date. The same code answers when the billing time of the customer cannot be determined; contact support. |
| `409` | `PAYMENT_PLAN_INSTALLMENT_BILLING_IN_PROGRESS` | The amount goes beyond the deposit and the next installment is being billed or is about to bill. |
| `502` | `PAYMENT_PLAN_DEPOSIT_INVOICE_NOT_RECORDED` | The deposit could not be recorded on its invoice, or its invoice payment could not be confirmed. The plan was not activated; send the request again with a new `Idempotency-Key`: the new request reuses the same invoice, so the deposit is not recorded twice. |
| `502` | `PAYMENT_PLAN_PAYMENT_UNCONFIRMED` | The deposit was recorded and the plan is `active`, but the payment could not be read back. Read the plan and its payments with `GET /payment-plans/{id}` and `GET /payment-plans/{id}/payments` before you send anything else: the same body with a new `Idempotency-Key` would record the money a second time. |
| `502` | `PAYMENT_PLAN_BILLING_ENGINE_UNAVAILABLE` | Billing data could not be read. Nothing was recorded on the plan; send the request again, with the same `Idempotency-Key` or a new one. |

The errors common to every payment on a plan, such as `409` `PAYMENT_PLAN_BUSY` and `409` `PAYMENT_PLAN_NOT_AWAITING_DEPOSIT` (another request activated or canceled the plan meanwhile), are listed under [Error Codes](#error-codes).

---

# Read Plans and Installments

- `GET /payment-plans` lists your plans, newest first, with the page parameters `page` and `take` (`take` at most `50`). Filter with `search`, a URL-encoded JSON object: `{"customer": "cus_739c07713fd4b68565a969cd", "status": "past_due,defaulted"}`. `status` takes one status or a comma-separated list. A `search` value that is not valid JSON answers `400`. Add `external_id` to return only the plan whose `external_id` equals that value. Leading and trailing spaces, tabs and line breaks of the value are removed first, then it must equal the stored value exactly (case-sensitive). A value that matches nothing returns an empty page; a value that is empty, blank, longer than 255 characters, contains a control character, or is sent twice answers `400`.
- `GET /payment-plans/{id}` returns one plan.
- `GET /payment-plans/{id}/installments` returns the plan's installments in ascending `sequence` order, the down payment first when there is one. `GET /payment-plans/{id}/installments` is paged like the other lists: `page` and `take` (default `10`, at most `50`); it answers `{ "data": [...], "meta": { "page", "take", "items_total", "pages_total" } }`. A 60-installment plan needs two pages at `take=50`. `GET /invoices` has no plan filter: read each installment's `invoice` id.

The Payment Plan object carries `description`: the text sent when the plan was created, or `null` when none was sent. It cannot be changed afterwards.

An installment's `surcharge_amount` and `amount_charged` report the credit card surcharge of the charges that paid it; see [Credit Card Surcharges](#credit-card-surcharges).

An installment's `invoice` is the invoice of its charge. For a deposit collected on a terminal (the down payment, or the first installment), it is the invoice ChaChing raises when the plan is activated; a deposit collected on a terminal before ChaChing raised these invoices reads `null`.

An installment's `status` is `scheduled`, `paid`, `failed`, `canceled`, `refunded` or `written_off`. The installment `status` list can grow: handle a value you do not know. An installment reads `written_off` after its invoice was written off; see Written-Off Installments. It reads `failed` after a declined automatic attempt and stays `failed` until it is paid. `attempt_count` counts the automatic charge attempts, retries included, and `next_payment_attempt` is the date of the next retry, or `null` when none is scheduled. An installment not yet billed reads `canceled` when the plan was canceled or paid off, or when a payment that reached the principal shortened the schedule and removed it. An installment reads `refunded` after a refund of a payment that paid it, while the refunded part is still owed on it: `amount_due` is that part and `amount_refunded` the total refunded of what was paid toward its `amount_paid` (a refund of a refunded amount collected again raises only `amount_due`). A refund of the down payment or the deposit, and of a payment that paid only part of an installment not yet paid in full, leaves the installment's `status` and `amount_due` unchanged and is owed on the plan (the plan's `amount_due`); the installment's `amount_refunded` still counts it. It returns to `paid` when the owed part is collected or forgiven. `payments` lists the payments that paid it.

The Invoice object, on the invoice endpoints and in every `invoice.*` webhook, carries `payment_plan`, `installment` and `payment_plan_charge`. All three are `null` on an invoice that belongs to no payment plan, and an invoice raised for a payment plan reports `subscription: null`. `payment_plan_charge` describes the plan charge the invoice records: `{ "id": "pp_1ef17ace4b4d96ff32461285", "installment": "ppi_9c1a3d0e2f4b8756a1c0d3ef", "sequence": 3, "payment": "ppp_5d2c8e1f0a3b4c6d7e8f9a0b", "kind": "installment" }`. `id` is the plan; `installment` and `sequence` are the installment the charge pays, both `null` for an extra payment, a payoff and a refunded amount owed on the plan collected again; `payment` is the Payment that paid the invoice — the most recent payment that collected money on it, whatever its `origin` — and is `null` for an invoice settled entirely by account credit, and on the `invoice.*` events sent for a charge before its payment is recorded (read the payment id from `payment_plan.payment_succeeded` or by reading the invoice again); `kind` is `down_payment`, `deposit` (the first installment collected on a terminal), `installment`, `extra_payment`, `payoff` or `refunded_amount` (a refunded amount collected again). `payment_plan_charge` is also `null` on a plan invoice not matched to an installment yet. An invoice that records the principal part of a payment reports `installment: null`.

## Preview a Schedule

This endpoint requires the account secret API key (`sk_`), or a platform key with `Chaching-Account`.

`POST /payment-plans/preview` takes the body of `POST /payment-plans` and answers `200` with the schedule that body would produce, without creating anything:

```json
{
  "object": "payment_plan_preview",
  "mode": "card_on_file",
  "customer": "cus_739c07713fd4b68565a969cd",
  "total_amount": 120000,
  "down_payment_amount": 20000,
  "financed_amount": 100000,
  "installment_amount": 25000,
  "installment_count": 4,
  "interval": "monthly",
  "remainder_placement": "first",
  "start_date": 1796083200,
  "installments": [
    { "sequence": 1, "due_date": 1796083200, "amount": 25000 },
    { "sequence": 2, "due_date": 1798761600, "amount": 25000 },
    { "sequence": 3, "due_date": 1801440000, "amount": 25000 },
    { "sequence": 4, "due_date": 1803859200, "amount": 25000 }
  ]
}
```

The values shown are illustrative.

`mode` is `card_on_file` when the body carries `payment_method` and `terminal_deposit` when it does not. `installments` lists installments 1 to N; the down payment appears only in `down_payment_amount`. `description` is accepted and ignored.

The preview checks the customer, your payment gateway, the interval, `start_date` and the amounts, with the codes `POST /payment-plans` uses for the same checks: `404` `CUSTOMER_NOT_FOUND`, and `400` `PAYMENT_PLAN_GATEWAY_NOT_SUPPORTED`, `PAYMENT_PLAN_INTERVAL_INVALID`, `PAYMENT_PLAN_START_DATE_INVALID`, `PAYMENT_PLAN_SCHEDULE_INVALID` and `PAYMENT_PLAN_AMOUNT_INVALID`. It does not check the payment method, the customer's account standing and unpaid invoices, an earlier down payment still pending, nor your retry settings, so `POST /payment-plans` can still refuse a body whose preview succeeded: with `404` `PAYMENT_METHOD_NOT_FOUND`; with `409` `PAYMENT_PLAN_PAYMENT_METHOD_NOT_SUPPORTED`, `PAYMENT_PLAN_CREDENTIAL_NOT_ESTABLISHED`, `PAYMENT_PLAN_AUTO_PAY_OFF`, `PAYMENT_PLAN_ACCOUNT_INVOICED_AHEAD`, `PAYMENT_PLAN_CUSTOMER_DELINQUENT`, `PAYMENT_PLAN_DUNNING_SCHEDULE_TOO_SHORT`, `PAYMENT_PLAN_UNPAID_OUTCOME_NOT_SUPPORTED` and `PAYMENT_PLAN_DOWN_PAYMENT_PENDING`; and with every outcome of the down-payment charge, the billing engine and the `Idempotency-Key` that [When a Plan Cannot Be Created](#when-a-plan-cannot-be-created) and [Duplicate Requests](#duplicate-requests) describe. The preview checks only the shape of `payment_method`; it does not check that the payment method belongs to the customer.

| **Status** | **`error`** | **When** |
| --- | --- | --- |
| `502` | `PAYMENT_PLAN_BILLING_ENGINE_UNAVAILABLE` | The billing state could not be read; nothing was started. Retry. |

## List a Plan's Payments

`GET /payment-plans/{id}/payments` lists the plan's payments, newest first, with the page parameters `page` and `take` (`take` at most `50`). Each entry is the Payment object that `GET /payment-plans/{id}/payments/{payment_id}` returns. Filter by status with `search`, a URL-encoded JSON object holding one status: `{"status": "declined"}`. `status` is `processing`, `applied`, `partially_applied`, `declined`, `not_attempted` or `unconfirmed`; any other value, or a `search` value that is not a JSON object, answers `400`.

`payment_plan_status`, `remaining_balance` and `installments_remaining` describe the plan at the time of the request, not at the time of each payment. Every attempt to collect money on the plan is a payment: the payments you record, each automatic installment charge and retry — declined ones included — the down payment, the terminal deposit, a payment of an installment invoice through its hosted page, the dashboard or `PUT /invoices/{id}`, also when it pays only part of the invoice, a charge of a new card, and any other payment of an installment invoice recorded directly in the billing engine (`origin` `automatic`). `origin` tells them apart: `payment`, `automatic`, `plan_card_charge`, `down_payment`, `deposit` or `invoice_payment`. A failed attempt reads `status` `declined` or `not_attempted` with `amount` `0` and its reason in `failure_code` and `failure_message`; `unconfirmed` means the outcome could not be confirmed: a payment you record keeps that status, so read the plan before retrying; any other payment changes to its final status when its outcome becomes known. Use `search={"status":"declined"}` to list declined attempts. An installment settled entirely by account credit has no payment; an automatic charge that was never started (a plan without a card, automatic payment switched off) and an automatic attempt that ChaChing's payment plan safeguards or a restart of ChaChing's connection to your processor refused before sending it (for example a card that does not belong to the plan) are not listed; a terminal deposit attempt that did not collect the deposit of a plan in `awaiting_deposit` is not a payment either: read it in the plan's `deposit.attempt` and with `GET /dejavoo/terminal-attempts`; a payment you record that those safeguards refuse, and an attempt your processor could not be reached for, are listed as `not_attempted`. Each payment lists its refunds in `refunds`, newest first, and `amount_refunded` is the plan amount refunded from it.

This endpoint requires the account secret API key (`sk_`), or a platform key with `Chaching-Account`.

---

# Billing Time of Day

ChaChing schedules each installment's charge at 10:00 UTC on its due date: 5:00 a.m. Eastern Standard Time in winter and 6:00 a.m. Eastern Daylight Time in summer. The time is fixed in UTC and does not move with daylight saving time. The due date is a calendar date in your account time zone.

- Each installment is invoiced and charged at 10:00 UTC on its due date, unless one of the events in the next bullet happens earlier on that date in UTC.
- An installment is invoiced and charged at once when, between 00:00 UTC and its scheduled time on its due date, you create a subscription for the same customer, pause, resume, cancel immediately or change one of the customer's subscriptions, or re-plan another payment plan of the customer; when the customer subscribes through one of your hosted checkout pages with the same e-mail address; when the customer pays a balance that had blocked its billing; or when another subscription of the customer starts, reaches the end of a fixed term or ends its trial at that time. In a US time zone such a charge can come as early as the evening before the due date: 00:00 UTC is 7:00 p.m. EST (8:00 p.m. EDT) in New York and 2:00 p.m. in Hawaii, and in New York a change after 5:00 UTC (4:00 UTC during daylight saving time) falls on the due date itself. In a time zone at UTC+00:00, 00:00 UTC is the start of the due date. This timing is not new; what changed is that due dates are now shown as dates in your account time zone.
- An automatic retry after a declined charge runs the number of days after the declined attempt that your retry schedule (`payment_retries`) sets, at about 10:00 UTC; each retry can start a few seconds later than the attempt before it.
- The down payment is charged when the plan is created.
- A customer created before ChaChing introduced this fixed billing time keeps the time of day at which the customer was created: that customer's installments and retries run at that time. In your account time zone that time can fall on the evening before the due date: this happens when the time of day in UTC is earlier than your account time zone is behind UTC (before 05:00 UTC for Eastern Standard Time, 04:00 UTC for Eastern Daylight Time, 10:00 UTC for Hawaii). For example, a customer created at 02:30 UTC whose installment is due on November 3, 2026 is charged on November 2, 2026 at 9:30 p.m. EST.

---

# Record Payments, Extra Payments and a Payoff

`POST /payment-plans/{id}/payments` records a payment you make on the plan, outside its automatic schedule, and answers `201` with the Payment object.

```json
{
  "amount": 50000,
  "payment_method": "9f3a1c2e-7b45-4d18-a6c0-2f8e5b7d1a94",
  "external": false
}
```

| **Field** | **Required** | **Rules** |
| --- | --- | --- |
| `amount` | yes | Integer cents, at least `1`, and at most the plan's `remaining_balance`. On a `canceled` plan, at most what is still owed on its billed installments that are not paid, plus `amount_due`. On a `paid_off` plan, at most `amount_due`. |
| `payment_method` | no | The card to charge for this payment only; it must belong to the plan's customer. Default: the plan's card; a plan without a card answers `400` `PAYMENT_METHOD_REQUIRED`. Ignored when `external` is `true`, except on a plan in `awaiting_deposit`, where it names the card for the remaining installments. Send `{ "type": "terminal", "terminal_id": "<terminal id>" }` instead to take the payment on one of your Dejavoo terminals; see **Take a Payment on a Terminal**. |
| `external` | no | `true` records money you collected outside ChaChing; nothing is charged. Default `false`. |

The amount is applied first to refunded amounts still owed — refunded installments oldest first, then the refunded amount owed on the plan — then to the plan's billed installments that are not fully paid, oldest first — a smaller amount pays part of an installment — and then to the part of the plan not billed yet, the principal.

- **Extra payment.** An amount below `remaining_balance`. When it reaches the principal, the installment amount and the due dates do not change: the plan ends sooner, and its last installment is reduced to what is left.
- **Payoff.** An amount equal to `remaining_balance`. It pays every open installment, closes the rest of the schedule, and moves the plan to `paid_off`.
- **Several charges.** A payment spanning several billed installments, or billed installments and the principal, is charged as one charge per installment invoice plus one charge for the principal, in that order. When the first charge is declined the request answers `402` `PAYMENT_DECLINED` and nothing is applied. When a later charge is declined, the charges already approved stay applied and the request answers `201` with the `amount` actually applied. When a later charge's outcome cannot be confirmed, the request still answers `201`, with `status` `processing` and `amount` `0` even when earlier charges of the same payment were approved: the payment is recorded only once every one of its charges is resolved. Do not retry with a new `Idempotency-Key`. Wait for `payment_plan.payment_succeeded`, which ChaChing sends with the amount actually applied once it finishes recording the payment, and read the plan's installments when the event does not arrive.
- **The principal has its own invoice.** The principal part of a payment is recorded on its own invoice, returned by `GET /invoices` and `GET /invoices/{id}` with `installment: null` and `payment_plan_charge.kind` `extra_payment`, or `payoff` for a payoff. When the principal is recorded, ChaChing sends `invoice.payment_succeeded` for that invoice, and `invoice.payment_failed` when its card charge is declined. `payment_plan.payment_succeeded` reports the same money as the invoice and installment events: count money from one event family only, and use `payment_plan_charge.kind` and `payment_plan_charge.payment` to recognize the invoice events of a plan payment. A principal recorded before ChaChing raised these invoices has no invoice. A principal recorded while the account holds credit, while automatic or manual payment is switched off for the account, or while invoicing was unavailable at that moment is recorded without an invoice and raises no `invoice.*` event; `payment_plan.payment_succeeded` still reports it. Send the invoice by email with `POST /invoices/{id}/send`, create its hosted payment link with `GET /invoices/{id}/hosted-link` and download its PDF with `GET /invoices/{id}/pdf`.
- A payment on a `past_due` or `defaulted` plan is never refused because of its status. While an earlier payment's schedule rebuild is still being completed, a further payment answers `409` `PAYMENT_PLAN_REPLAN_PENDING`; retry it later.

The Payment object's `status` tells you how far the recording went:

| **`status`** | **Meaning** |
| --- | --- |
| `applied` | Every charge was approved and the whole amount was applied. |
| `partially_applied` | A later charge was declined or not attempted, or the terminal approved less than the amount sent. `amount` is the part that was applied. |
| `processing` | At least one charge has no confirmed outcome yet. Nothing is recorded yet and `amount` reads `0`. ChaChing finishes recording the payment and then sends `payment_plan.payment_succeeded` with the final `status` and `amount`, or `payment_plan.payment_failed` when it ends `declined` or `not_attempted`, except a payment not taken on a terminal whose charge the payment plan safeguards or a restart of ChaChing's connection to your processor refused, or that ChaChing could not send to the billing engine, which ends `not_attempted` without an event. |
| `declined` | The first charge was declined, or the terminal sale was declined. Nothing was applied. |
| `not_attempted` | The first charge was not attempted, or the terminal sale did not take place: it was cancelled, the terminal confirmed that nothing was charged, the terminal refused the request, or ChaChing staff closed it. Nothing was applied. A payment also reads `not_attempted` when the terminal approved and charged the card but the plan could absorb none of the amount: `terminal.state` is then `approved` or `partial_approval`, `terminal.amount_charged` carries the collected amount and the payment's own `amount_charged` is `0`, so do not ask the customer to pay again. |
| `unconfirmed` | The outcome of the first charge could not be confirmed. Nothing was applied. For a payment you record, read the plan before retrying; any other payment (an automatic charge, an invoice payment) changes to its final status when its outcome becomes known. |

The `201` of `POST /payment-plans/{id}/payments` and `payment_plan.payment_succeeded` carry only `processing`, `applied` and `partially_applied`. `GET /payment-plans/{id}/payments/{payment_id}` returns any payment of the plan with its `status` as recorded, so it can also read `declined`, `not_attempted` or `unconfirmed`. `payment_plan.payment_failed` carries a payment that reads `declined` or `not_attempted`.

Each recorded payment sends `payment_plan.payment_succeeded`, one `payment_plan.installment_paid` for every installment it paid in full, and `payment_plan.completed` when it is a payoff. For one payment recorded with `POST /payment-plans/{id}/payments` they are emitted in this order (deliveries can arrive in any order): `payment_plan.installment_paid` for each installment it paid in full, then `payment_plan.payment_succeeded`, then `payment_plan.completed`. A declined or refused payment sends none of them. When a request ends before its payment is fully recorded (for example a `502` `PAYMENT_PLAN_PAYMENT_UNCONFIRMED`, or a `201` whose later charge could not be confirmed), ChaChing finishes recording the payment later and sends these events then. Every other payment that collects money sends `payment_plan.payment_succeeded` too: an automatic charge, a down payment and a terminal deposit each send both `payment_plan.installment_paid` and `payment_plan.payment_succeeded`. A payment that ends `declined` or `not_attempted` sends `payment_plan.payment_failed` (except a payment you record, not on a terminal, that the payment plan safeguards or a restart of ChaChing's connection to your processor refused, or that ChaChing could not send to the billing engine: it is listed as `not_attempted` without an event) — a payment you record that is declined (the request still answers `402`, or `201` for a payment taken on a terminal, which sends it when the sale ends), and each failed automatic attempt: a declined one together with its `payment_plan.installment_failed`, and one your processor was never reached for (`not_attempted`, for example while your iPOSpays credentials are rejected) alone, because such an attempt does not count against the installment. A declined payment of an installment invoice through its hosted page, the dashboard or `PUT /invoices/{id}` sends `payment_plan.payment_failed` alone, and a declined charge of a new card sends it together with its `payment_plan.installment_failed`. The installment events (`payment_plan.installment_paid`, `payment_plan.installment_failed`, `payment_plan.completed`) and the payment events (`payment_plan.payment_succeeded`, `payment_plan.payment_failed`) of one charge can arrive in any order, as webhook delivery order is never guaranteed: match a succeeded payment to its installments by its `allocations[].installment`, never by arrival order; a failed payment carries no installment (its `allocations` is empty), so its `payment_plan.payment_failed` and the attempt's `payment_plan.installment_failed` share only the plan and customer ids. Each automatic retry is its own payment and sends its own event. A payment whose outcome is not known yet (`unconfirmed`) sends nothing. Payments of attempts made before these events were extended to them are added to the list without an event; around that one-time switch-over, an attempt made shortly before it can still send its event, and one made while a plan's earlier payments are being added can send none.

## Take a Payment on a Terminal

Moved to [Payment Plans on a Terminal](./payment-plans-terminal.md#take-a-payment-on-a-terminal).

---

# Find and Force-Close a Stuck Terminal Attempt

Moved to [Payment Plans on a Terminal](./payment-plans-terminal.md#find-and-force-close-a-stuck-terminal-attempt).

---

# Refund a Payment

`POST /payment-plans/{id}/payments/{payment_id}/refunds` returns all or part of a payment to the card that paid it and answers `201` with the Refund object.

```bash
curl -X POST https://stage-api.chaching.io/payment-plans/$PLAN_ID/payments/$PAYMENT_ID/refunds \
  -H "cc-api-key: $CHACHING_SECRET_KEY" \
  -H "Idempotency-Key: refund-wrong-card-1" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 40,
    "reason": "Charged to the wrong card"
  }'
```

<!-- captured: the payment_plan.payment_refunded delivery of a refund of 40 on stage (the Refund object the request returns), redacted -->
```json
{
  "id": "pprf_04612cce2a00ba25978d674f",
  "object": "payment_plan_refund",
  "payment_plan": "pp_oTFFXTxvDT9zMSjwTs3Q6R1g",
  "status": "succeeded",
  "kind": "void",
  "source": "api",
  "amount": 40,
  "amount_requested": 40,
  "surcharge_amount": 1,
  "amount_returned": 41,
  "forgiven": false,
  "amount_forgiven": 0,
  "amount_due": 40,
  "reason": "Charged to the wrong card",
  "failure_code": null,
  "failure_message": null,
  "installments": [
    {
      "installment": null,
      "sequence": null,
      "amount": 40,
      "amount_due": 40
    }
  ],
  "payment_plan_amount_due": 40,
  "payment_plan_status": "active",
  "created": 1795644839,
  "payment": {
    "id": "ppp_l3vMXlIXSzQFTqIYtKHCCBFV",
    "object": "payment_plan_payment",
    "payment_plan": "pp_oTFFXTxvDT9zMSjwTs3Q6R1g",
    "payment_plan_external_id": "treatment-1",
    "customer": "cus_f0d9c6b125586ae0c550a09a",
    "customer_external_id": null,
    "status": "applied",
    "origin": "payment",
    "failure_code": null,
    "failure_message": null,
    "amount": 100,
    "surcharge_amount": 3,
    "amount_charged": 103,
    "amount_refunded": 40,
    "surcharge_status": "final",
    "external": false,
    "payment_method": "07c7a3f1-191d-4bbd-b7a6-25f6dafe1d0d",
    "card": {
      "brand": "visa",
      "last4": "5439"
    },
    "invoice": "9818eb6c-fa53-4b2c-a015-cf406c12e9d8",
    "hosted_invoice_url": "https://stage.chaching.io/checkout/9818eb6c-fa53-4b2c-a015-cf406c12e9d8",
    "payoff": false,
    "created": 1795644831,
    "payment_plan_status": "active",
    "remaining_balance": 140,
    "installments_remaining": 1,
    "allocations": [
      {
        "installment": null,
        "sequence": null,
        "kind": "principal",
        "amount": 100,
        "surcharge_amount": 3,
        "amount_charged": 103,
        "surcharge_status": "final",
        "invoice": "9818eb6c-fa53-4b2c-a015-cf406c12e9d8",
        "hosted_invoice_url": "https://stage.chaching.io/checkout/9818eb6c-fa53-4b2c-a015-cf406c12e9d8"
      }
    ],
    "refunds": [
      {
        "id": "pprf_04612cce2a00ba25978d674f",
        "object": "payment_plan_refund",
        "payment_plan": "pp_oTFFXTxvDT9zMSjwTs3Q6R1g",
        "status": "succeeded",
        "kind": "void",
        "source": "api",
        "amount": 40,
        "amount_requested": 40,
        "surcharge_amount": 1,
        "amount_returned": 41,
        "forgiven": false,
        "amount_forgiven": 0,
        "amount_due": 40,
        "reason": "Charged to the wrong card",
        "failure_code": null,
        "failure_message": null,
        "installments": [
          {
            "installment": null,
            "sequence": null,
            "amount": 40,
            "amount_due": 40
          }
        ],
        "payment_plan_amount_due": 40,
        "payment_plan_status": "active",
        "created": 1795644839
      }
    ],
    "payment_plan_summary": {
      "installment_count": 2,
      "currency": "USD",
      "description": "Orthodontic treatment, upper and lower braces",
      "remaining_balance": 140,
      "installments_paid": 0,
      "installments_remaining": 1,
      "next_due_date": 1798156800,
      "status": "active"
    },
    "terminal": null
  }
}
```

Returns the Refund object, with the updated Payment object in `payment`.

Any payment that collected money can be refunded, an automatic installment charge and a terminal deposit included; a payment taken on a terminal is refunded without the terminal. Money collected outside ChaChing (a payment recorded with `external: true`, or any payment ChaChing holds no processor reference for), a payment that collected nothing for the plan (`declined`, `not_attempted`, `unconfirmed`; a terminal sale the plan could absorb none of reads `not_attempted` although the card was charged, and ChaChing's team refunds it), and a payment still being recorded (`processing`), cannot be refunded. The request needs an `Idempotency-Key`.

| **Field** | **Rules** |
| --- | --- |
| `amount` | No. Integer cents of the plan amount to refund, at least `1`, below `100000000` and at most what is left to refund on the payment. Default: everything left to refund. |
| `reason` | No. Your note, 1 to 500 characters with no control, line-separator, bidirectional or zero-width characters. |
| `forgive` | No. `true` returns the money and leaves nothing owed. Default `false`: the refunded amount is owed again. |

Void or refund: ChaChing sends a refund and your processor decides. On a Dejavoo account a payment not settled yet is voided, a settled payment is refunded; `kind` reads `void` or `refund`.

Surcharge: the card gets back the refunded amount plus the same share of the payment's surcharge. `surcharge_amount` is the surcharge returned and `amount_returned` the total returned to the card. The last refund of a payment returns the rest of its surcharge.

ChaChing has tested refunds on Dejavoo (iPOSpays) accounts. On another gateway the processor's own void and refund rules apply, `kind` reads `refund`, no surcharge is returned, and a refusal of the processor answers `402`.

What is owed after a refund: the refunded amount is owed again. A refunded installment reads `refunded` with `amount_due`; a refunded down payment or deposit, the refunded principal part of a payment (an extra payment, or the unbilled part of a payoff), and a refund of a payment that paid only part of an installment not yet paid in full, are owed on the plan in the plan's `amount_due`; the part of a payment that paid an installment that is paid in full when you refund is owed on that installment. Nothing is charged automatically, no date changes, the plan keeps its status, and no reminder or late fee follows. Collect it with `POST /payment-plans/{id}/payments`, on a card of the customer or on a terminal: refunded amounts are paid first. A plan is not `paid_off` while a refunded amount is owed; a plan that was already `paid_off` stays `paid_off`.

Forgive: send `forgive: true` to return the money without leaving anything owed. To forgive what is still owed from an earlier refund, call `POST /payment-plans/{id}/payments/{payment_id}/refunds/{refund_id}/forgive`. Forgiving cannot be undone. A refund that is still being confirmed cannot be forgiven yet: the call answers `409` `PAYMENT_PLAN_REFUND_PENDING` until the refund is final; retry shortly. Confirming a refund with the processor usually takes up to 24 hours.

Refunds made in your iPOSpays account: when the account's FEED notifications reach ChaChing (see [The Dejavoo FEED Webhook URL](./processor-credentials.md#the-dejavoo-feed-webhook-url)), ChaChing applies them the same way and sends `payment_plan.payment_refunded` with `source` `processor`; without FEED, ChaChing does not see them. Make refunds through ChaChing: a refund made in iPOSpays cannot carry `forgive`.

Charge on the wrong card: refund the payment, then collect the owed amount on a terminal with `save_payment_method: true`. On a plan that is `active`, `past_due` or `defaulted`, the card tapped becomes the plan's card, unless that payment pays the plan off; a `paid_off` or `canceled` plan keeps its card.

Chargebacks are not reported.

If a refund was created but its result could not be read, the request answers `500` with the message `The refund was created but its result could not be read. Find it in the payment's refunds: <refund id>`; read the payment to find that refund before retrying.

Through a platform key, a deactivated account cannot refund or forgive: both answer `409 ACCOUNT_DEACTIVATED`, like every other change outside the operations a deactivated account accepts. A refund made in your iPOSpays account is still applied to the plan when the account's FEED notifications reach ChaChing.

| **Status** | **Error code** | **When** | **What to do** |
| --- | --- | --- | --- |
| `400` | `IDEMPOTENCY_KEY_REQUIRED` | The request has no `Idempotency-Key` header. Nothing was refunded or forgiven. | Send the request again with an `Idempotency-Key`. |
| `400` | `IDEMPOTENCY_KEY_INVALID` | The `Idempotency-Key` header is malformed. | Send 1 to 255 printable ASCII characters with no spaces. |
| `400` | `PAYMENT_PLAN_REFUND_AMOUNT_INVALID` | `amount` is not an integer of at least `1` and below `100000000`. | Send the amount in integer cents. |
| `400` | `PAYMENT_PLAN_REFUND_AMOUNT_EXCEEDS_REFUNDABLE` | `amount` is larger than what is left to refund on the payment. Nothing was refunded. | Send at most the payment's `amount` minus its `amount_refunded`. |
| `400` | none | The body carries another field, `reason` is not 1 to 500 characters with no control characters, or `forgive` is not a boolean. The body has no `error` field; `message` describes the problem. | Fix the body. |
| `402` | `PAYMENT_PLAN_REFUND_DECLINED` | The processor declined or refused the refund, or ChaChing could not send the refund to the processor, for example when it could not authenticate with iPOSpays for a terminal refund; `message` reads "The refund was not completed." and the refund's `failure_code` and `failure_message` give the reason. Nothing was refunded. | Check the sale and the API credentials in your processor's account (iPOSpays on a Dejavoo account), then retry with a new `Idempotency-Key`. |
| `403` | `PAYMENT_PLANS_NOT_ENABLED` | Payment Plans are not enabled for this account. | Contact Chaching to enable them. |
| `403` | `SECRET_API_KEY_REQUIRED` | The key is a publishable key. | Use the account secret API key (`sk_...`). |
| `403` | `API_KEY_SCOPE_INSUFFICIENT` | The key's scopes include neither `*` nor the scope of the operation. | Create a key with the scope the message names. |
| `404` | `PAYMENT_PLAN_NOT_FOUND` | The plan id does not resolve for this account. | Check the id. |
| `404` | `PAYMENT_PLAN_PAYMENT_NOT_FOUND` | The payment id does not resolve to a payment of this plan. | Check the id. |
| `409` | `ACCOUNT_DEACTIVATED` | The request uses a platform key and the account is deactivated. Nothing was refunded or forgiven. | Reactivate the account first. A refund made in your iPOSpays account is still applied to the plan when the account's FEED notifications reach ChaChing. |
| `409` | `PAYMENT_PLAN_PAYMENT_NOT_REFUNDABLE` | The payment collected nothing for the plan or is still being recorded, it was collected outside ChaChing, nothing is left to refund on it, or its invoice no longer carries the amount to refund. `message` reads `This payment cannot be refunded.` when the payment collected nothing for the plan or is still being recorded, and states the reason in the other cases. Nothing was refunded. | Refund money collected outside ChaChing where it was collected. |
| `409` | `PAYMENT_PLAN_REFUND_PENDING` | An earlier refund of this payment is still being confirmed. | Retry shortly. Confirming a refund with the processor usually takes up to 24 hours. |
| `409` | `PAYMENT_PLAN_REFUND_SURCHARGE_UNCONFIRMED` | The surcharge of the charge could not be established, so ChaChing cannot refund it. Nothing was refunded. | Refund it in your iPOSpays account; with FEED notifications on, ChaChing applies that refund to the plan. |
| `409` | `PAYMENT_PLAN_BUSY` | Another payment, a refund or a cancellation on this plan is in progress. | Retry shortly. |
| `409` | `IDEMPOTENCY_KEY_IN_PROGRESS` | A request with this Idempotency-Key is still being processed. | Wait for it to finish, then repeat the request with the same key. |
| `422` | `IDEMPOTENCY_KEY_REUSED` | The `Idempotency-Key` was already used with a different request. | Use a new key for a different request. |
| `502` | `PAYMENT_PLAN_BILLING_ENGINE_UNAVAILABLE` | Billing data could not be read before the refund was sent, or the processor refunded nothing and the billing engine did not record the refund. Nothing was refunded. | Retry later with a new `Idempotency-Key`. |
| `502` | `IDEMPOTENCY_REQUEST_INCOMPLETE` | The first request with this `Idempotency-Key` never completed. | Read the payment's `refunds`, then retry with a new key. |

## Forgive a Refund

`POST /payment-plans/{id}/payments/{payment_id}/refunds/{refund_id}/forgive`, with an empty body and an `Idempotency-Key`, forgives what is still owed from the refund and answers `200` with the Refund object.

```bash
curl -X POST https://stage-api.chaching.io/payment-plans/$PLAN_ID/payments/$PAYMENT_ID/refunds/$REFUND_ID/forgive \
  -H "cc-api-key: $CHACHING_SECRET_KEY" \
  -H "Idempotency-Key: forgive-refund-1"
```

<!-- captured: the payment_plan.refund_forgiven delivery of a forgiven refund on stage (the Refund object the request returns), redacted -->
```json
{
  "id": "pprf_9e7799f58917100ff9936c60",
  "object": "payment_plan_refund",
  "payment_plan": "pp_ngeYZg5RAHRfruhuP6VD1guW",
  "status": "succeeded",
  "kind": "void",
  "source": "api",
  "amount": 100,
  "amount_requested": 100,
  "surcharge_amount": 3,
  "amount_returned": 103,
  "forgiven": true,
  "amount_forgiven": 100,
  "amount_due": 0,
  "reason": null,
  "failure_code": null,
  "failure_message": null,
  "installments": [
    {
      "installment": null,
      "sequence": null,
      "amount": 100,
      "amount_due": 0
    }
  ],
  "payment_plan_amount_due": 0,
  "payment_plan_status": "active",
  "created": 1795640098,
  "payment": {
    "id": "ppp_xbZLOcSkE4BKkDkQdY7ScaY0",
    "object": "payment_plan_payment",
    "payment_plan": "pp_ngeYZg5RAHRfruhuP6VD1guW",
    "payment_plan_external_id": "treatment-1",
    "customer": "cus_23a18adaf37366de8f70bcbd",
    "customer_external_id": "patient-55821",
    "status": "applied",
    "origin": "payment",
    "failure_code": null,
    "failure_message": null,
    "amount": 100,
    "surcharge_amount": 3,
    "amount_charged": 103,
    "amount_refunded": 100,
    "surcharge_status": "final",
    "external": false,
    "payment_method": "0fa737b9-5c41-4634-bcb8-29733bf51db7",
    "card": {
      "brand": "mastercard",
      "last4": "0055"
    },
    "invoice": "a09b2a86-5e7e-45a2-80a4-ef13b829b53b",
    "hosted_invoice_url": "https://stage.chaching.io/checkout/a09b2a86-5e7e-45a2-80a4-ef13b829b53b",
    "payoff": false,
    "created": 1795640076,
    "payment_plan_status": "active",
    "remaining_balance": 800,
    "installments_remaining": 8,
    "allocations": [
      {
        "installment": null,
        "sequence": null,
        "kind": "principal",
        "amount": 100,
        "surcharge_amount": 3,
        "amount_charged": 103,
        "surcharge_status": "final",
        "invoice": "a09b2a86-5e7e-45a2-80a4-ef13b829b53b",
        "hosted_invoice_url": "https://stage.chaching.io/checkout/a09b2a86-5e7e-45a2-80a4-ef13b829b53b"
      }
    ],
    "refunds": [
      {
        "id": "pprf_9e7799f58917100ff9936c60",
        "object": "payment_plan_refund",
        "payment_plan": "pp_ngeYZg5RAHRfruhuP6VD1guW",
        "status": "succeeded",
        "kind": "void",
        "source": "api",
        "amount": 100,
        "amount_requested": 100,
        "surcharge_amount": 3,
        "amount_returned": 103,
        "forgiven": true,
        "amount_forgiven": 100,
        "amount_due": 0,
        "reason": null,
        "failure_code": null,
        "failure_message": null,
        "installments": [
          {
            "installment": null,
            "sequence": null,
            "amount": 100,
            "amount_due": 0
          }
        ],
        "payment_plan_amount_due": 0,
        "payment_plan_status": "active",
        "created": 1795640098
      }
    ],
    "payment_plan_summary": {
      "installment_count": 10,
      "currency": "USD",
      "description": "Orthodontic treatment, upper and lower braces",
      "remaining_balance": 800,
      "installments_paid": 0,
      "installments_remaining": 8,
      "next_due_date": 1796169600,
      "status": "active"
    },
    "terminal": null
  }
}
```

Returns the Refund object with `forgiven: true`, with the updated Payment object in `payment`.

| **Status** | **Error code** | **When** | **What to do** |
| --- | --- | --- | --- |
| `400` | `IDEMPOTENCY_KEY_REQUIRED` | The request has no `Idempotency-Key` header. Nothing was refunded or forgiven. | Send the request again with an `Idempotency-Key`. |
| `400` | `IDEMPOTENCY_KEY_INVALID` | The `Idempotency-Key` header is malformed. | Send 1 to 255 printable ASCII characters with no spaces. |
| `400` | none | The body carries a field. The body has no `error` field; `message` describes the problem. | Send an empty body. |
| `403` | `PAYMENT_PLANS_NOT_ENABLED` | Payment Plans are not enabled for this account. | Contact Chaching to enable them. |
| `403` | `SECRET_API_KEY_REQUIRED` | The key is a publishable key. | Use the account secret API key (`sk_...`). |
| `403` | `API_KEY_SCOPE_INSUFFICIENT` | The key's scopes include neither `*` nor the scope of the operation. | Create a key with the scope the message names. |
| `404` | `PAYMENT_PLAN_NOT_FOUND` | The plan id does not resolve for this account. | Check the id. |
| `404` | `PAYMENT_PLAN_PAYMENT_NOT_FOUND` | The payment id does not resolve to a payment of this plan. | Check the id. |
| `404` | `PAYMENT_PLAN_REFUND_NOT_FOUND` | The refund id does not resolve to a refund of this payment. | Check the id. |
| `409` | `ACCOUNT_DEACTIVATED` | The request uses a platform key and the account is deactivated. Nothing was refunded or forgiven. | Reactivate the account first. A refund made in your iPOSpays account is still applied to the plan when the account's FEED notifications reach ChaChing. |
| `409` | `PAYMENT_PLAN_BUSY` | Another payment, a refund or a cancellation on this plan is in progress. | Retry shortly. |
| `409` | `PAYMENT_PLAN_REFUND_PENDING` | This refund is still being confirmed. | Retry shortly. Confirming a refund with the processor usually takes up to 24 hours. |
| `409` | `PAYMENT_PLAN_PAYMENT_PENDING_CONFIRMATION` | A payment that collects what is still owed from this refund is still being confirmed. | Retry once that payment reads its final status. |
| `409` | `PAYMENT_PLAN_REFUND_NOT_FORGIVABLE` | Nothing of this refund is still owed. | Nothing to do. |
| `409` | `IDEMPOTENCY_KEY_IN_PROGRESS` | A request with this Idempotency-Key is still being processed. | Wait for it to finish, then repeat the request with the same key. |
| `422` | `IDEMPOTENCY_KEY_REUSED` | The `Idempotency-Key` was already used with a different request. | Use a new key for a different request. |
| `502` | `IDEMPOTENCY_REQUEST_INCOMPLETE` | The first request with this `Idempotency-Key` never completed. | Read the payment's `refunds`, then retry with a new key. |

---

# Written-Off Installments

When the merchant writes an installment's invoice off (`DELETE /invoices/{id}` on an open period invoice, or the dashboard write-off), the installment's `status` becomes `written_off` at once: ChaChing no longer collects it, `amount_written_off` is what was not collected (`amount` minus `amount_paid`), `written_off_at` is when, and it leaves `remaining_balance`, `installments_remaining` and `next_due_date` and does not count in `installments_paid`. `amount_paid` stays the money actually collected. The plan's `amount_written_off` is the total of its written-off installments. ChaChing sends `payment_plan.installment_written_off` once, and `invoice.written_off` for the invoice.

Several installments that share one invoice are all written off together.

A written-off installment can still be paid through its invoice (the hosted invoice page, `PUT /invoices/{id}`): it then becomes `paid` with the amount collected, `amount_written_off` returns to `0` and ChaChing sends `payment_plan.installment_paid`. A declined attempt leaves it `written_off` and sends no `payment_plan.installment_failed`. `POST /payment-plans/{id}/payments` never applies money to a written-off installment.

When the written-off installment was the last one still owed and the schedule has ended, the plan becomes `paid_off` and ChaChing sends `payment_plan.completed` once; a plan completed this way has `amount_written_off` above `0`. A `canceled` plan stays `canceled`.

The worked example (amounts in cents): a plan of 3 installments of 10000, no down payment. Installment 1 is paid (`payment_plan.installment_paid`); installment 2 is declined (`payment_plan.installment_failed`); the merchant voids installment 2's invoice: installment 2 reads `written_off` with `amount_written_off` 10000, `remaining_balance` 10000 (installment 3), `installments_remaining` 1, `payment_plan.installment_written_off` and `invoice.written_off` are sent, and automatic retries of the invoice charge nothing. Installment 3 is billed and paid: `installments_paid` 2, `remaining_balance` 0, and when the schedule ends the plan is `paid_off` with `amount_written_off` 10000 and one `payment_plan.completed`. Variant A: `POST /payment-plans/{id}/payments` with `amount: 10000` before installment 3 is billed is a payoff of what is still owed (installment 3, unbilled): the plan becomes `paid_off`, installment 3 reads `canceled` (covered by the early payment), installment 2 stays `written_off` and nothing is applied to it. Variant B: the customer later pays installment 2's invoice through the hosted page: installment 2 becomes `paid` with `amount_paid` 10000, one `payment_plan.installment_paid` is sent, and a plan already `paid_off` stays `paid_off` with no second `payment_plan.completed`. Variant C: installment 3 is the one written off after the schedule ended: the plan completes at the write-off.

---

# Credit Card Surcharges

On an account that charges through iPOSpays (Dejavoo), every credit card charge can carry a credit card surcharge, payment plan charges included, and debit card charges carry none; [Credit Card Surcharges](./card-surcharges.md) describes the rule for every charge. Whether a credit card charge is surcharged, and how much, is decided by the S.T.E.A.M configuration of your account in the iPOSpays portal. ChaChing records the surcharge separately and never changes a plan amount because of it.

These payment plan charges carry it:

- the down payment;
- every automatic installment charge and every automatic retry;
- every installment invoice paid from the payment page or from the dashboard;
- every charge of `POST /payment-plans/{id}/payments` that is not `external`, the principal part included, and every payment taken on a terminal.

A payment plan invoice paid with a credit card, directly or through Google Pay, through any route — the automatic charge, `PUT /invoices/{id}`, the payment page or the dashboard — carries your account's credit card surcharge.

Accounts that charge through NMI never carry a surcharge, and neither does a payment recorded with `external: true`, because nothing is charged.

The plan amounts never include the surcharge: `total_amount`, `installment_amount`, every installment `amount`, `amount_paid`, `remaining_balance`, the invoice totals and the payoff amount stay as agreed.

Two fields report it, always present and in integer cents, on the Installment, Payment, Transaction and Invoice objects:

| **Field** | **Meaning** |
| --- | --- |
| `surcharge_amount` | The credit card surcharge added to the charge or charges. `0` when none was added, which is always the case for a debit card charge. |
| `amount_charged` | The total charged to the card. It equals `amount_paid` on an installment, `amount` on a payment and on a transaction, and `amount_paid` on an invoice, when nothing was added and nothing was refunded. |

For example, a $1.00 payment that carried a 3 cent surcharge reads `amount: 100`, `surcharge_amount: 3`, `amount_charged: 103`; the installment that payment paid reads `amount_paid: 100`, `surcharge_amount: 3`, `amount_charged: 103`.

- `amount_charged` can exceed the amount plus `surcharge_amount` when the gateway adds another S.T.E.A.M amount for your account. That difference is included in `amount_charged` and is not itemised.
- A payment that did not succeed never shows a surcharge.
- ChaChing reads the surcharge from the gateway after the charge. While it cannot be read yet, `surcharge_amount` is `0` and `amount_charged` equals the amount paid. ChaChing keeps trying for up to 7 days, and the objects show the real values as soon as the read succeeds. When the surcharge of a charge still cannot be read after those 7 days, or at once when the charge cannot be matched, the installment and payment objects keep `surcharge_amount` at `0` and `amount_charged` equal to the amount paid, and that does not change later. The exact amount charged to the card is then the one shown for that transaction in your Dejavoo (iPOSpays) account.
- `surcharge_status` on the Installment and Payment objects and on each payment allocation is `final` once the surcharge is read, and `unavailable` while it cannot be read; `surcharge_amount` then reads `0` and `amount_charged` the amount paid. `payment_plan.installment_paid` and `payment_plan.payment_succeeded` are sent once it is read, or after an hour with `surcharge_status` `unavailable`.
- A refund returns the surcharge in proportion to the refunded amount; see **Refund a Payment**.

Before the payer pays, the payment page of an open invoice and the dashboard's `Charge customer` sheet tell them that a credit card surcharge can be added and that debit cards are not surcharged; [Credit Card Surcharges](./card-surcharges.md#what-your-customer-sees-before-paying) lists every notice and its exact wording.

In the dashboard, the transactions list, the transaction details and the invoice details show the surcharge and the total charged for a charge that carried one. The dashboard's Total revenue and a customer's total spend include the surcharge for accounts that charge through Dejavoo. A refund lowers both by the refunded amount and, for a card charge ChaChing sent to iPOSpays, the surcharge returned with it.

---

# Failed Payments and Payment Plans

Your retry schedule and your failed-payment outcome apply to the whole account — every subscription and every payment plan — and they are read and changed with `GET /revenue-recovery` and `PUT /revenue-recovery`, described in [Configure Failed-Payment Settings](./subscription-lifecycle.md). The two outcome fields take these exact values: `subscription_state_on_payment_failure` is `cancel`, `unpaid` or `past-due`, and `invoice_state_on_payment_failure` is `past-due` or `uncollectible`.

The late-payment warning and the failed-payment outcome are evaluated per customer, not per plan. An unpaid subscription invoice moves the customer's plans too, and an unpaid installment moves the customer's subscriptions.

- **Late-payment warning.** When the customer's account enters it, each of the customer's `active` plans changes to `past_due` and ChaChing sends `payment_plan.past_due`. Automatic charging continues.
- **Failed-payment outcome.** When the customer's account reaches your outcome, each `active` or `past_due` plan changes to `defaulted` and ChaChing sends `payment_plan.defaulted`. With `cancel`, the plan's schedule is cut at the outcome day: the installment already billed stays owed in full, no further installment is billed, and no automatic charge is attempted again. With `past-due`, the schedule keeps running and every installment is still charged on its due date.
- **Back to `active`.** When the customer's account clears, a `past_due` plan, and a `defaulted` plan under the `past-due` outcome, whose schedule is still running return to `active` and ChaChing sends `payment_plan.reactivated`. Under the `cancel` outcome, a `defaulted` plan whose schedule was cut stays `defaulted` until it is paid off, canceled, or a payment with a principal portion through `POST /payment-plans/{id}/payments` rebuilds its remaining schedule while the customer's account is clear.
- **Settings for accounts with Payment Plans.** On an account with Payment Plans enabled, or holding a plan that has not ended, `payment_retries` must sum to at least 7 days and `subscription_state_on_payment_failure` cannot be `unpaid`; `PUT /revenue-recovery` rejects either with `400`. With these rules the outcome day is day 8 or later.
- **The debt survives.** A `defaulted` plan and a `canceled` plan keep their outstanding balance. ChaChing never forgives a balance on its own: `uncollectible` is a reporting label, and you can still collect through `POST /payment-plans/{id}/payments`. The merchant's write-off of an installment's invoice is the act that stops collecting it.
- **A plan without a card.** A plan has no card when it is activated by a deposit recorded as collected outside ChaChing without `payment_method`, and after its card is removed with `PATCH /payment-plans/{id}` and `payment_method` `null`. In both cases nothing is charged automatically: each installment is invoiced on its due date and paid manually. An unpaid installment moves the plan to `past_due` and `defaulted` on the same schedule as any other plan, and ChaChing sends no failed-payment notice for an installment on which no charge was attempted. See **Record a Deposit Collected Outside Chaching**.

Do not confuse the plan status `past_due`, which means the late-payment warning, with the outcome value `past-due`. A customer who reaches the `past-due` outcome makes the plan `defaulted`, and ChaChing sends `payment_plan.defaulted`, not `payment_plan.past_due`.

For the subscription side of the same rules, see [Track subscription lifecycle and failed payments](./subscription-lifecycle.md) and [Handle failed subscription payments](../Using%20Chaching/Subscriptions/failed-payments.md).

---

# Emails Your Customer Receives

ChaChing sends the emails below unless the account's customer emails are turned off (`PATCH /email-settings` with `customer_emails_enabled: false`, or the dashboard Settings page). When they are off, ChaChing sends none of them and the webhooks keep being sent; the account, which for a platform is the platform, sends the receipts and decline notices itself and meets the card-network obligations listed in [Send Your Own Customer Notifications](./customer-notifications.md).

ChaChing emails the plan's customer at the email address stored on the customer. These emails are best effort: a customer with no email address receives none, and an email that fails to send does not change the response, the plan or the webhooks. A receipt is not sent for a charge more than 48 hours old when the receipt step runs; for a deposit collected on a terminal, the 48 hours count from the moment the plan is activated.

**Receipts.** The customer receives one receipt for each of these card charges:

- the down payment, when the plan is created;
- the deposit collected on a terminal (the down payment, or the first installment), when the plan is activated;
- each installment charged automatically on its due date or by an automatic retry;
- each payment recorded with `POST /payment-plans/{id}/payments` that is not `external`, a payment taken on a terminal included. One receipt lists every charge of that payment;
- an installment invoice paid by card through ChaChing outside those endpoints: by your customer from the payment page, or by you from the dashboard or with `PUT /invoices/{id}`.

The subject is `Receipt from <your account name>: payment 3 of 9` for an installment, `Receipt from <your account name>: down payment` for the down payment, and `Receipt from <your account name>: payment plan payment` for a recorded payment that covers several charges or reaches the principal. `9` is the plan's `installment_count`. The receipt states your account name, your city and state when your account's billing contact holds both, the plan's `description`, the amount and currency, the date, the card brand and its last four digits, a `Credit card surcharge` line after the payment lines when the charge carried a surcharge, with `Total charged` as the amount the card was charged, the remaining balance and the number of payments remaining (the receipt of the last installment, or of a payment that completes the plan, says the plan is paid in full instead), and a button that opens the invoice of the installment. The receipt of the principal part of a payment and of a deposit collected on a terminal carries no invoice button; their invoices are returned by `GET /invoices` and `GET /invoices/{id}`. When the card details cannot be read, the receipt prints `Card on file`.

A payment recorded with `external: true` is not a card charge. The customer receives a `Payment recorded` email with the amount and the remaining balance, and no card details. When that payment pays the plan off, the email's subject says the plan is paid in full and its text says so instead of giving a remaining balance.

A refund sends the customer no email; `payment_plan.payment_refunded` reports it to your integration.

**Failed payments.** Each declined automatic attempt on an installment, retries included, sends the customer a notice with the invoice number and amount, the line `Installment 3 of 9 of your payment plan.`, the plan's `description`, the date of the next automatic retry when one is scheduled, a button to pay the invoice with another payment method, and this sentence: `You have at least 7 calendar days from the first failed attempt on this installment to pay it with another payment method.` For accounts that charge through iPOSpays, the notice also says: `If the card is a credit card, <your account name> can add a credit card surcharge to this amount. Debit cards are not surcharged.` Your account owner is copied on these notices.

A declined down payment sends the customer no email: the request answers `402` `PAYMENT_PLAN_DOWN_PAYMENT_DECLINED`, and no plan exists.

**Description.** A plan created without `description` prints `Payment plan` followed by the plan's `reference` for a plan created without `payment_method`, and by the plan id otherwise.

**Copies of receipts.** Receipts go to the customer. A copy of a receipt goes to your account owner only when both the ChaChing-enabled receipt copy and the account's owner copies (`owner_email_copies_enabled`) are on; contact ChaChing to enable the receipt copy. Failed-payment and overdue notices copy the owner while `owner_email_copies_enabled` is `true`.

These emails do not replace the webhooks: `payment_plan.installment_paid`, `payment_plan.installment_failed` and `payment_plan.payment_succeeded` report the same payments to your integration. Count money from one event family only: `payment_plan.payment_succeeded` reports each payment, `payment_plan.installment_paid` each installment paid in full.

---

# Cancel a Plan

`POST /payment-plans/{id}/cancel` answers `200` with the plan in status `canceled` and `canceled_at` set, and ChaChing sends `payment_plan.canceled`. Installments not yet billed are removed at the end of the period already billed and are never charged. Cancellation is not a refund and does not forgive debt: amounts already invoiced stay collectible through `POST /payment-plans/{id}/payments`, and `remaining_balance` shows what is still owed. After the cancel, ChaChing makes no automatic charge for the plan: an installment invoice that is still open is collected only by an explicit payment, through `POST /payment-plans/{id}/payments` or the invoice payment routes. Voiding that invoice with `DELETE /invoices/{id}` writes it off, so its period is never billed again, the installment reads `written_off`, and ChaChing sends `invoice.written_off` and `payment_plan.installment_written_off` (see [Written-Off Installments](#written-off-installments)); if two requests for the same invoice arrive together, one answers `409 INVOICE_CHANGE_IN_PROGRESS` and can be retried.

This endpoint is the only way to end a plan early. A payment plan is not a subscription: the subscription endpoints answer `404` for it.

---

# External IDs

Customers and payment plans accept an optional `external_id` when you create them: your own identifier, such as the patient id or the plan id in your system. It holds up to 255 Unicode characters (code points; an emoji counts as one) with no control characters; leading and trailing spaces, tabs and line breaks are removed, and a blank value means none.

- **Scope.** A customer's `external_id` is unique among the active customers of your account, and a plan's among all payment plans of your account, whatever their status. A customer and a plan can carry the same value, and another account can use the same value.
- **Set once.** The value is set when the object is created and never changes. `PATCH /customers/{id}` with `external_id` in the body answers `400` `CUSTOMER_EXTERNAL_ID_IMMUTABLE`; payment plans have no update endpoint. Customers and plans created before external ids existed have `external_id: null`.
- **Duplicates.** `POST /customers` with a value an active customer of your account already carries answers `409` `CUSTOMER_EXTERNAL_ID_EXISTS`; `POST /payment-plans` with a value a plan of your account already carries answers `409` `PAYMENT_PLAN_EXTERNAL_ID_EXISTS`. The message names the existing `cus_…` or `pp_…` id and does not repeat the external id. A deleted customer frees its value. `POST /customers` and `POST /payment-plans` with an `external_id` that is not 1 to 255 characters or contains a control character answer `400` with no `error` field.
- **Retries.** Send an `Idempotency-Key` header on every `POST /payment-plans` that carries an `external_id`. A retry with the same key replays the first answer instead of creating, or refusing, a second plan. Two concurrent creates with the same `external_id` and different keys can leave a charged down payment without a plan. Contact Chaching support with the invoice id of the down payment: a down payment charged without a plan has no payment that `POST /payment-plans/{id}/payments/{payment_id}/refunds` can refund.
- **Lookup.** `GET /customers?external_id=…` and `GET /payment-plans?external_id=…` return the normal page, filtered by an exact, case-sensitive match. Leading and trailing spaces, tabs and line breaks of the value are removed first, then it must equal the stored value exactly (case-sensitive). A value that matches nothing returns an empty page; a value that is empty, blank, longer than 255 characters, contains a control character, or is sent twice answers `400`.
- **Echo.** The Customer and Payment Plan objects carry `external_id`. The Payment Plan object also carries `customer_external_id`; the Installment and Payment objects carry `customer`, `customer_external_id` and `payment_plan_external_id`; the Invoice object carries `customer_external_id` and `payment_plan_external_id`. Webhooks carry the same objects.

Chaching stores only the customer name, payment method, amounts and external ids. Do not put clinical or other protected health information — no diagnosis, no procedure, no date of birth — in `description` or `external_id`.

---

# Error Codes

Errors use the standard body `{ "statusCode", "message", "error", "timestamp", "path" }` plus the fields a row names. Branch on `statusCode` and `error`; `message` can change. A request body that fails field validation — a value below its minimum such as `total_amount: 0` or `installment_count: 0`, a negative `down_payment_amount`, a value of the wrong type, a `description` longer than 255 characters or containing a line break or another control character, an `external_id` that is not 1 to 255 characters or contains a control character, or an unknown field such as `terminal` — and a `search` value that is not valid JSON are rejected with `400` and a body that carries no `error` field; `message` describes the problem.

| **Status** | **`error`** | **When** |
| --- | --- | --- |
| `400` | `PAYMENT_PLAN_SCHEDULE_INVALID` | Both or neither of `installment_count` and `installment_amount` were sent, `installment_count` is above `60` or larger than the amount to finance allows, `installment_amount` is larger than the amount to finance, or the amounts do not form a valid schedule, or `installment_amount` would need more than `60` installments. |
| `400` | `PAYMENT_PLAN_AMOUNT_INVALID` | `down_payment_amount` is not below `total_amount`, or a payment's `amount` is below `1`. |
| `400` | `PAYMENT_PLAN_INTERVAL_INVALID` | `interval` is not one of the values this environment accepts: `weekly`, `biweekly`, `monthly` and `annual` everywhere, and `daily` on test environments only. |
| `400` | `PAYMENT_PLAN_START_DATE_INVALID` | `start_date` is not after the day of the request, or `interval` is `annual` and `start_date` is 29 February (UTC), or the start is within an hour of the customer's billing time, or is not after today in the account time zone. The message names the earliest valid date. |
| `400` | `PAYMENT_PLAN_GATEWAY_NOT_SUPPORTED` | The account's payment gateway does not support Payment Plans. |
| `400` | `PAYMENT_PLAN_AMOUNT_EXCEEDS_BALANCE` | A payment is larger than what the plan accepts. Nothing is charged. |
| `400` | `PAYMENT_PLAN_AMOUNT_BELOW_DEPOSIT` | On a plan in `awaiting_deposit`, a payment's `amount` is below the deposit. Nothing is recorded. |
| `400` | `PAYMENT_METHOD_REQUIRED` | A payment was sent with no `payment_method`, it is not external, and the plan has no card. |
| `400` | `PAYMENT_PLAN_TERMINAL_EXTERNAL_NOT_ALLOWED` | A payment named a terminal and `external: true` together. |
| `400` | `IDEMPOTENCY_KEY_INVALID` | The `Idempotency-Key` header is malformed. |
| `400` | `IDEMPOTENCY_KEY_REQUIRED` | The request has no `Idempotency-Key` header. Nothing was refunded or forgiven. |
| `400` | `PAYMENT_PLAN_REFUND_AMOUNT_INVALID` | `amount` is not an integer of at least `1` and below `100000000`. |
| `400` | `PAYMENT_PLAN_REFUND_AMOUNT_EXCEEDS_REFUNDABLE` | `amount` is larger than what is left to refund on the payment. Nothing was refunded. |
| `402` | `PAYMENT_PLAN_DOWN_PAYMENT_DECLINED` | The down payment was declined. No plan was created. |
| `402` | `PAYMENT_DECLINED` | The first charge of a payment was declined. Nothing was applied. |
| `402` | `PAYMENT_PLAN_REFUND_DECLINED` | The processor declined or refused the refund, or ChaChing could not send the refund to the processor, for example when it could not authenticate with iPOSpays for a terminal refund; `message` reads "The refund was not completed." and the refund's `failure_code` and `failure_message` give the reason. Nothing was refunded. |
| `403` | `PAYMENT_PLANS_NOT_ENABLED` | Payment Plans are not enabled for this account. |
| `403` | `SECRET_API_KEY_REQUIRED` | Any endpoint was called with a publishable (`pk_...`) API key. |
| `404` | `PAYMENT_PLAN_NOT_FOUND` | The plan id does not resolve for this account. |
| `404` | `CUSTOMER_NOT_FOUND` | The customer id does not resolve for this account. |
| `404` | `PAYMENT_METHOD_NOT_FOUND` | The payment method does not resolve, or it does not belong to the customer. |
| `404` | `DEJAVOO_TERMINAL_NOT_FOUND` | The `terminal_id` does not resolve to a terminal of this account. |
| `404` | `PAYMENT_PLAN_PAYMENT_NOT_FOUND` | The payment id does not resolve to a payment of this plan. |
| `404` | `PAYMENT_PLAN_TERMINAL_ATTEMPT_NOT_FOUND` | The terminal attempt id does not resolve for this account. |
| `404` | `PAYMENT_PLAN_REFUND_NOT_FOUND` | The refund id does not resolve to a refund of this payment. |
| `409` | `PAYMENT_PLAN_CREDENTIAL_NOT_ESTABLISHED` | The saved card was not established for automatic payments. Create the plan with a down payment, or ask the customer to enter the card again. On a deposit collected outside, the card named in `payment_method` was not set up for automatic payments: name another card, or record the deposit without a card. |
| `409` | `PAYMENT_PLAN_PAYMENT_METHOD_NOT_SUPPORTED` | On a create, the payment method is not a card saved with the account's current payment gateway. On `POST /payment-plans/{id}/payments`, on `PUT /invoices/{id}` and on a payment from the hosted invoice page or the dashboard of any invoice that carries an unpaid installment, the card that would be charged — `payment_method`, or the plan's card when it is omitted — is not a card saved with the payment gateway the plan was created on; a plan created before you changed your payment gateway keeps accepting its own cards. Nothing was created or charged. Send a payment method returned by the payment-methods endpoints, or record money you collected yourself with `external: true` on `POST /payment-plans/{id}/payments`. On a deposit collected outside, `payment_method` is not a card saved with your account's Dejavoo gateway. |
| `409` | `PAYMENT_PLAN_PAYMENT_METHOD_REQUIRED` | An invoice that is an installment of a payment plan without a card, or that holds installments of plans with different cards, was paid with `PUT /invoices/{id}` or from the hosted invoice page or the dashboard without a payment method. It is not charged to a saved card automatically. Nothing is charged; pay it on the hosted invoice page, or record the payment on the payment plan. |
| `409` | `PAYMENT_PLAN_PAYMENT_METHOD_NOT_CHANGEABLE` | On `PATCH /payment-plans/{id}`, the plan is `awaiting_deposit`, `canceled` or `paid_off`. Nothing was changed. |
| `409` | `PAYMENT_METHOD_IN_USE_BY_PAYMENT_PLAN` | The payment method is the card of an open payment plan (`active`, `past_due` or `defaulted`), so it cannot be detached. Nothing was deleted; give the plan another card, or remove its card, first. |
| `409` | `PAYMENT_METHOD_CHANGE_IN_PROGRESS` | Another change to this customer's payment methods is in progress. Nothing was changed; try again. |
| `409` | `ACCOUNT_DEACTIVATED` | On `PATCH /payment-plans/{id}`, the account is deactivated and the request names a card. Send `payment_method` `null`, or reactivate the account first. On a refund or a forgive sent with a platform key, the account is deactivated. |
| `409` | `PAYMENT_PLAN_UNPAID_OUTCOME_NOT_SUPPORTED` | Your failed-payment outcome is `unpaid`. |
| `409` | `PAYMENT_PLAN_DUNNING_SCHEDULE_TOO_SHORT` | Your `payment_retries` sum to less than 7 days. |
| `409` | `PAYMENT_PLAN_CUSTOMER_DELINQUENT` | The customer's account is in the late-payment warning or a failed-payment outcome, or carries an invoice unpaid for one day or more. |
| `409` | `PAYMENT_PLAN_AUTO_PAY_OFF` | A plan created with `payment_method` has a down payment and automatic payments are turned off for the customer in the billing engine. Nothing was created or charged. Create the plan without a down payment. |
| `409` | `PAYMENT_PLAN_ACCOUNT_INVOICED_AHEAD` | The customer was already invoiced through a date on or after `start_date`. The message names that date. Nothing was created; choose a `start_date` after it. |
| `409` | `PAYMENT_PLAN_DOWN_PAYMENT_NOT_ATTEMPTED` | The down payment was not attempted because the payment processor was restarting. Nothing was charged and no plan was created; retry later. |
| `409` | `PAYMENT_PLAN_DOWN_PAYMENT_PENDING` | An earlier down-payment invoice of the same amount for this customer is still open, or was settled in a way this request cannot use (a different payment method, no card payment, or more than one such invoice). Nothing was created or charged by this request. Check the customer's invoices and transactions: wait for an open charge to finish, send the `payment_method` that paid, or refund or void the invoices that should not count, then send the request again. |
| `409` | `PAYMENT_PLAN_EXTERNAL_ID_EXISTS` | A payment plan of your account already carries this `external_id`. The message names that plan's `pp_…` id. Nothing was created or charged by this request. |
| `409` | `PAYMENT_PLAN_NOT_PAYABLE` | The plan is `paid_off` and owes no refunded amount (`amount_due` `0`), or it is `awaiting_deposit` and the payment is not `external: true`. |
| `409` | `PAYMENT_PLAN_NOT_CANCELABLE` | The plan is already `paid_off` or `canceled`. |
| `409` | `CUSTOMER_HAS_OPEN_PAYMENT_PLANS` | On `DELETE /customers/{customer_id}`: the customer has a plan that is `awaiting_deposit`, `active`, `past_due` or `defaulted`. Nothing is deleted; cancel the plan, or collect its remaining balance, first. |
| `409` | `PAYMENT_PLAN_BUSY` | Another payment, a refund or a cancellation on this plan is in progress. Retry shortly. |
| `409` | `PAYMENT_PLAN_NOT_AWAITING_DEPOSIT` | A deposit was started or confirmed on a plan that is not `awaiting_deposit`. It also answers when the plan stopped being `awaiting_deposit` while a deposit collected outside was being recorded. |
| `409` | `PAYMENT_PLAN_DEPOSIT_IN_FLIGHT` | The plan has a deposit attempt that is not resolved yet; for the confirm endpoint, one that is still `in_flight`. A deposit collected outside is refused while an attempt is unresolved. |
| `409` | `PAYMENT_PLAN_DEPOSIT_NO_ATTEMPT` | The plan has no deposit attempt to confirm. |
| `409` | `PAYMENT_PLAN_DEPOSIT_NOT_IN_FLIGHT` | There is no sale waiting at the terminal to cancel. |
| `409` | `PAYMENT_PLAN_DEPOSIT_ATTEMPTS_EXHAUSTED` | The plan reached 99 deposit attempts. |
| `409` | `DEJAVOO_TERMINAL_DISABLED` | The terminal is disabled. |
| `409` | `DEJAVOO_TERMINAL_BUSY` | The terminal is running another sale, a deposit or a payment. |
| `409` | `DEJAVOO_TERMINAL_OFFLINE` | The terminal is not connected. Turn it on and try again. |
| `409` | `SPIN_AUTH_KEY_UNREADABLE` | The terminal's stored Auth Key cannot be read. Re-enter it with `PATCH /dejavoo/terminals/{id}`. |
| `409` | `PAYMENT_PLAN_START_DATE_TOO_SOON` | The first installment charged after the deposit is due within 1 hour, so the terminal deposit cannot start, or the deposit collected outside cannot be recorded. Cancel the plan and create it again with a later start date. |
| `409` | `PAYMENT_PLAN_INSTALLMENT_BILLING_IN_PROGRESS` | The payment reaches the principal and the next installment is being billed or is about to bill: for a `daily` plan, its billing time is less than 1 hour away; for every other cadence, it is due within 24 hours. It can also answer for a `daily` plan whose billing time cannot be determined. Nothing is charged. |
| `409` | `PAYMENT_PLAN_PAYMENT_PENDING_CONFIRMATION` | An earlier payment on this plan is still being confirmed, a terminal payment that reads `processing` included. Nothing is charged; retry later. On `POST /payment-plans/{id}/cancel`, a terminal payment of the plan is unresolved. On a forgive, a payment that collects what is still owed from the refund is still being confirmed. |
| `409` | `PAYMENT_PLAN_REPLAN_PENDING` | An earlier payment's schedule rebuild is still being completed. Nothing is charged; retry later. |
| `409` | `PAYMENT_PLAN_PAYMENT_NOT_ATTEMPTED` | The first charge was not attempted. Nothing is charged. |
| `409` | `PAYMENT_PLAN_TERMINAL_NOT_SUPPORTED` | A terminal payment was sent for a plan that is not on the Dejavoo gateway. |
| `409` | `PAYMENT_PLAN_PAYMENT_NOT_TERMINAL` | The resolve or cancel endpoint was called for a payment that was not taken on a terminal. |
| `409` | `PAYMENT_PLAN_TERMINAL_PAYMENT_IN_FLIGHT` | The customer is still at the terminal for this payment. |
| `409` | `PAYMENT_PLAN_TERMINAL_PAYMENT_NOT_IN_FLIGHT` | There is no sale of this payment waiting at the terminal to cancel. |
| `409` | `PAYMENT_PLAN_TERMINAL_ATTEMPTS_EXHAUSTED` | The plan reached 99 terminal attempts, deposit attempts included. |
| `409` | `PAYMENT_PLAN_TERMINAL_ATTEMPT_LIVE` | A force-close of a terminal attempt whose sale can still be open at the terminal, for which the terminal reported an approval, or whose plan has another request running. Nothing was changed. |
| `409` | `PAYMENT_PLAN_TERMINAL_ATTEMPT_TOO_RECENT` | A force-close less than 15 minutes after the attempt's `deadline`. `closable_at` is the earliest time a force-close is accepted, Unix seconds. Nothing was changed. |
| `409` | `PAYMENT_PLAN_TERMINAL_ATTEMPT_FINAL` | A force-close of a terminal attempt that is already final. Nothing was changed. |
| `409` | `PAYMENT_PLAN_TERMINAL_ATTEMPT_RESOLVED` | The status read of a force-close found an outcome; it was recorded and nothing was closed. `message` names the attempt's state. |
| `409` | `IDEMPOTENCY_KEY_IN_PROGRESS` | A request with the same `Idempotency-Key` is still running. |
| `409` | `PAYMENT_PLAN_PAYMENT_NOT_REFUNDABLE` | The payment collected nothing for the plan or is still being recorded, it was collected outside ChaChing, nothing is left to refund on it, or its invoice no longer carries the amount to refund. `message` reads `This payment cannot be refunded.` when the payment collected nothing for the plan or is still being recorded, and states the reason in the other cases. Nothing was refunded. |
| `409` | `PAYMENT_PLAN_REFUND_SURCHARGE_UNCONFIRMED` | The surcharge of the charge could not be established, so ChaChing cannot refund it. Nothing was refunded. |
| `409` | `PAYMENT_PLAN_REFUND_NOT_FORGIVABLE` | Nothing of this refund is still owed. |
| `409` | `PAYMENT_PLAN_REFUND_PENDING` | On a refund, an earlier refund of the payment is still being confirmed; on a forgive, the refund itself is. Retry shortly. Confirming a refund with the processor usually takes up to 24 hours. |
| `422` | `IDEMPOTENCY_KEY_REUSED` | The `Idempotency-Key` was already used with a different request. |
| `422` | `SPIN_CREDENTIALS_REJECTED` | SPIn did not accept the terminal's TPN and Auth Key. |
| `422` | `DEJAVOO_TERMINAL_ISV_ID_INVALID` | The terminal's `isv_id` is not 1 to 16 characters, as SPIn requires. Update it with `PATCH /dejavoo/terminals/{id}`. |
| `500` | `PAYMENT_PLAN_SCHEDULE_MISMATCH` | The registered schedule did not match the expected installment dates. No plan was created. |
| `502` | `PAYMENT_PLAN_DOWN_PAYMENT_UNCONFIRMED` | The down payment's outcome could not be confirmed. No plan was created and the charge is not reversed. `down_payment_invoice_id` is the id of the down-payment invoice to check. Once its charge has settled, send the same request again; this answer is not stored, so the same `Idempotency-Key` is accepted. A retry that itself answers a stored `502`, for example `PAYMENT_PLAN_BILLING_ENGINE_UNAVAILABLE`, needs a new key. A charge paid with the same `payment_method` is used for the new plan and is not taken again. From the next UTC day, the request answers `409` `PAYMENT_PLAN_CUSTOMER_DELINQUENT` instead until that invoice is paid or voided. Check the customer's invoices before you change the amount. |
| `502` | `PAYMENT_PLAN_PAYMENT_UNCONFIRMED` | The first charge's outcome could not be confirmed. It is not reversed; read the plan before retrying. |
| `502` | `PAYMENT_PLAN_DEPOSIT_INVOICE_NOT_RECORDED` | A terminal deposit, or a deposit collected outside, could not be recorded on its invoice. For a deposit collected outside the plan was not activated, and its invoice payment can already be recorded: send the request again with a new `Idempotency-Key`, which reuses that invoice. For a terminal deposit, resolve it again. |
| `502` | `PAYMENT_PLAN_BILLING_ENGINE_UNAVAILABLE` | Billing data could not be read, including the billing account of the customer when `start_date` is checked (create, preview, terminal deposit and deposit collected outside). Nothing was created or charged; safe to retry with a new `Idempotency-Key`. On a refund, the billing data could not be read before the refund was sent, or the processor refunded nothing and the billing engine did not record the refund; nothing was refunded. |
| `502` | `PAYMENT_PLAN_EXTERNAL_ID_CHECK_FAILED` | The duplicate check of `external_id` right before the down payment could not be read. Nothing was created or charged; the same `Idempotency-Key` is evaluated again on the next request. |
| `502` | `PAYMENT_PLAN_CANCEL_INCOMPLETE` | The cancellation did not complete. The plan keeps its status and installments, but part of its schedule can already be stopped. Repeat the request: a repeat completes the rest. |
| `502` | `IDEMPOTENCY_REQUEST_INCOMPLETE` | The first request with this `Idempotency-Key` never completed. |
| `503` | `SPIN_UNREACHABLE` | SPIn could not be reached to check the terminal. Nothing was started; try again. |

---

# Payment Plan Events

Payment Plans send fifteen webhook event types on the same endpoints, envelope, signature and retry policy as every other ChaChing event. Their payloads are described in [Webhooks](./webhook.md). To send your own receipts and notices from these events, see [Send Your Own Customer Notifications](./customer-notifications.md).

| **Event** | **Sent when** | **`data`** |
| --- | --- | --- |
| `payment_plan.created` | A plan is created. | Payment Plan |
| `payment_plan.activated` | A plan starts running. A plan created with `payment_method` sends it right after `payment_plan.created`; a plan created without it sends it when its deposit, collected on a terminal, is confirmed, or when you record its deposit collected outside ChaChing. | Payment Plan |
| `payment_plan.installment_paid` | An installment is paid by its automatic charge or a retry, by a payment through `POST /payment-plans/{id}/payments`, by a payment of its invoice made through ChaChing (the hosted invoice page, the dashboard or `PUT /invoices/{id}`), by credit on the customer's account that settles its invoice, as the down payment, charged at creation, collected on a terminal or recorded as collected outside ChaChing (sequence `0`), or as the first installment collected on a terminal or recorded as collected outside ChaChing (sequence `1`). An installment that was refunded and is `paid` again, because its owed part is collected or forgiven, sends no second `payment_plan.installment_paid`. | Installment |
| `payment_plan.installment_failed` | An automatic charge attempt on an installment is declined, once per attempt, retries included. | Installment |
| `payment_plan.past_due` | The plan changes to `past_due`. | Payment Plan |
| `payment_plan.defaulted` | The plan changes to `defaulted`. | Payment Plan |
| `payment_plan.reactivated` | The plan returns to `active` from `past_due` or `defaulted`. | Payment Plan |
| `payment_plan.completed` | The plan changes to `paid_off`. | Payment Plan |
| `payment_plan.canceled` | The plan is canceled. | Payment Plan |
| `payment_plan.payment_succeeded` | A payment of the plan collected money: a payment you record, an automatic charge, a down payment, a terminal deposit, an invoice payment or a charge of a new card. | Payment |
| `payment_plan.payment_method_updated` | A plan's card is set, replaced or removed: with `PATCH /payment-plans/{id}`, or when a terminal payment sent with `save_payment_method: true` saves its card. It is not sent when the card does not change, nor when a plan is created, activated or has its deposit recorded. | Payment Plan, plus `previous_payment_method` |
| `payment_plan.payment_failed` | A payment of the plan is declined or could not be processed: a payment you record, an automatic attempt, an invoice payment or a charge of a new card. | Payment |
| `payment_plan.payment_refunded` | A payment of the plan is refunded or voided, through ChaChing (the API or the dashboard), or in your iPOSpays account when the account's FEED notifications reach ChaChing. | Refund |
| `payment_plan.refund_forgiven` | The amount still owed from a refund is forgiven with `POST …/refunds/{refund_id}/forgive`. A refund sent with `forgive: true` sends only `payment_plan.payment_refunded`, with `forgiven: true`. | Refund |
| `payment_plan.installment_written_off` | An installment's invoice is written off; the installment reads `written_off`. Sent once per installment. | Installment |

An endpoint whose `enabled_events` is an explicit list receives `payment_plan.payment_failed`, `payment_plan.payment_refunded`, `payment_plan.refund_forgiven` and `payment_plan.installment_written_off` only after you add them to its list (a sent `enabled_events` replaces the whole list: send the current list plus the new types). An endpoint subscribed to `["*"]` receives them at once.

A deposit recorded as collected outside ChaChing emits, in this order (deliveries can arrive in any order): `invoice.payment_succeeded` for the deposit invoice, `invoice.payment_succeeded` for the invoice of the amount above the deposit when there is one, `payment_plan.activated`, `payment_plan.installment_paid` for the deposit, `payment_plan.payment_succeeded`, and `payment_plan.completed` when it paid the plan off. `payment_plan.activated` shows the deposit as paid; the amount above it is reported by the events that follow.

An installment whose invoice is settled by credit on the customer's account, with no card charge, becomes `paid` with `amount_paid` equal to the amount the credit covered and `surcharge_amount` `0`. `paid_at` is the moment ChaChing recorded it, which can be up to about 10 minutes after the credit was applied. `payment_plan.installment_paid` is sent once and no receipt email is sent.

When an installment's invoice is reduced by an adjustment and the rest is paid, the installment becomes `paid` and `amount_paid` is the amount actually collected; the adjusted part is not counted as paid. The item adjustment ChaChing makes for a refund is not such a reduction: the installment keeps its `amount_paid`; the refunded part is owed on the installment, which then reads `refunded`, or on the plan, as **Refund a Payment** describes.

Read the plan with `GET /payment-plans/{id}` whenever you need its current state: the plan and installment endpoints are the source of truth, and deliveries can arrive in a different order than the changes happened.
