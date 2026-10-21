# Payment Plans

A payment plan collects a fixed total amount from one customer in a fixed number of scheduled installments. ChaChing stores the plan, charges each installment automatically on its due date against the plan's payment method, retries a declined charge on your account's retry schedule, and reports every change over webhooks. There is no interest. The installments always add up to the total agreed when the plan was created; on an account that charges through iPOSpays (Dejavoo), each card charge of a plan also carries the card surcharge configured for your account in the iPOSpays S.T.E.A.M portal, recorded separately from the plan amounts (see [Card Surcharges](#card-surcharges)).

Payment Plans are available through the API. They are enabled per merchant account by ChaChing and are off by default. While they are off, every `/payment-plans` endpoint answers `403` with the error code `PAYMENT_PLANS_NOT_ENABLED` and the message `Payment Plans are not enabled for this account. Contact Chaching to enable them.`

Merchants also work with payment plans in the ChaChing dashboard. On an account that charges through iPOSpays (Dejavoo), the dashboard collects the deposit of a plan — its down payment, or its first installment when it has none — on one of the account's registered terminals and takes payments on a terminal, through the same deposit and payment flows this page describes. Nothing changes for an integration: every endpoint, object and webhook on this page works as documented.

Every endpoint on this page authenticates with the account API key in the `cc-api-key` header, like every other public endpoint. Amounts are integers in cents, dates and timestamps are Unix seconds, and field names are `snake_case`. For every request and response schema, see the [API Reference](./api.json). For the webhook payloads, see [Webhooks](./webhook.md).

---

# Concepts

| **Term** | **Meaning** |
| --- | --- |
| Payment plan | An agreement to collect `total_amount` from one customer in a fixed number of installments. Its id starts with `pp_`. |
| Installment | One scheduled payment of a plan. Its id starts with `ppi_`. Each installment has a sequence number, a due date, an amount and a status, and it produces its own invoice when it is billed. |
| Down payment | An amount paid before any installment: charged when a plan is created with `payment_method`, or collected on one of your Dejavoo terminals for a plan created without one. It is recorded as installment sequence `0`, and it is not counted in `installment_count`. |
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
| `awaiting_deposit` | The plan was created without a payment method and waits for its deposit — the down payment when the plan has one, else its first installment — which the customer pays on one of your Dejavoo terminals. See **Collect the First Installment on a Terminal**. |

`paid_off` and `canceled` are final. The status value is `canceled`, with one `l`.

| **From** | **To** | **When** | **Event** |
| --- | --- | --- | --- |
| — | `active` | You create the plan with `payment_method`. | `payment_plan.created`, then `payment_plan.activated` |
| — | `awaiting_deposit` | You create the plan without `payment_method`. | `payment_plan.created` |
| `awaiting_deposit` | `active` | The deposit collected on your terminal is confirmed and its card is saved. A plan with one installment and no down payment goes to `paid_off` instead. | `payment_plan.activated` and `payment_plan.installment_paid`, plus `payment_plan.completed` for `paid_off` |
| `active` | `past_due` | The customer's account enters the late-payment warning. | `payment_plan.past_due` |
| `active`, `past_due` | `defaulted` | The customer's account reaches your failed-payment outcome. | `payment_plan.defaulted` |
| `defaulted` | `past_due` | Under the `past-due` outcome, the customer's account falls back from the outcome to the late-payment warning. | `payment_plan.past_due` |
| `past_due`, `defaulted` | `active` | The customer's account clears, or a payment with a principal portion rebuilds the remaining schedule while the account is clear. See **Failed Payments and Payment Plans** for the `cancel` outcome. | `payment_plan.reactivated` |
| `active`, `past_due`, `defaulted` | `paid_off` | The schedule ends with every installment paid, a payoff payment is recorded, or the last unpaid installment is paid through the hosted invoice page, the dashboard or `PUT /invoices/{id}`, or settled by credit on the customer's account. A `past_due` or `defaulted` plan whose schedule ends while an installment is still unpaid keeps its status until that installment is paid: it changes at once when the payment goes through ChaChing or credit settles it, and when the customer's account clears when its automatic charge or a retry collects it. | `payment_plan.completed` |
| `active`, `past_due`, `defaulted`, `awaiting_deposit` | `canceled` | You call `POST /payment-plans/{id}/cancel`. | `payment_plan.canceled` |

---

# Create a Plan

`POST /payment-plans` creates a plan and answers `201` with the Payment Plan object, in status `active`. A plan created without `payment_method` answers in status `awaiting_deposit` instead; see **Collect the First Installment on a Terminal**.

Request body:

```json
{
  "customer": "cus_739c07713fd4b68565a969cd",
  "description": "Orthodontic treatment, upper and lower braces",
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
| `description` | no | A short text describing what the plan pays for. ChaChing shows it to the customer on the plan's receipts and failed-payment emails. Up to 255 characters, with no line breaks or other control characters. Leading and trailing spaces are removed, and a value that is empty or only spaces is stored as no description. With an `Idempotency-Key`, sending such a value and leaving `description` out are the same request; adding a non-empty `description` to a request you retry with the same key, or changing it, answers `422`. |
| `total_amount` | yes | Integer cents, at least `1`. |
| `installment_count` | one of the two | Integer from `1` to `60`. |
| `installment_amount` | one of the two | Integer cents, at least `1`. Send exactly one of `installment_count` and `installment_amount`; ChaChing derives the other. |
| `down_payment_amount` | no | Integer cents, `0` or more and less than `total_amount`. Default `0`. |
| `interval` | no | `weekly`, `biweekly`, `monthly` or `annual`. Default `monthly`. On test environments, such as stage, `daily` is also accepted; production refuses `daily` with `400` `PAYMENT_PLAN_INTERVAL_INVALID`. An `annual` plan cannot start on 29 February. |
| `start_date` | no | Unix seconds: the due date of installment 1. Only the UTC calendar date is used: the time of day is dropped, the plan starts at 00:00:00 UTC of that date, and that is the `start_date` the plan returns. It must fall on a later calendar day (UTC) than the request; a date on the day of the request is refused. Default: one interval after the day of the request. |
| `remainder_placement` | no | `first` or `last`. Default `first`. |
| `payment_method` | no | The id of a payment method of this customer, as returned by the payment-methods endpoints. Leave it out to collect the deposit on a terminal: the down payment when `down_payment_amount` is above `0`, else installment 1. |

Without `payment_method`, the plan opens in `awaiting_deposit`, and its deposit is collected on one of your Dejavoo terminals: the down payment when `down_payment_amount` is above `0`, else installment 1. Nothing is charged and nothing is billed until the deposit is collected. With a down payment, the terminal collects that amount only, and installment 1 is charged on `start_date` to the card the terminal saved. Collect the deposit before `start_date`; installment 1 is due on `start_date` whenever the deposit is collected. A plan in this mode runs only on an account whose payment gateway is Dejavoo. The request fields `terminal` and `reference` are rejected with `400`: you choose the terminal when you collect the deposit, and ChaChing creates the plan's `reference`.

What happens when a plan is created with `payment_method`:

- Before charging anything, ChaChing checks the schedule it registered against the expected installment dates: every date for a plan of up to 12 installments, and, for a longer plan, the first installment date (plus the second when the first installment carries the odd amount). A date that cannot be projected because the customer has another subscription that bills often, such as a daily one, is not checked, and the plan is still created. A date more than 36 months after the day the check runs (UTC) is not checked either, and the plan is still created. A mismatch answers `500` `PAYMENT_PLAN_SCHEDULE_MISMATCH`, and no plan is created.
- When `down_payment_amount` is greater than `0`, ChaChing charges it immediately against `payment_method` and records it as installment sequence `0`, already paid. When the charge is declined, the request answers `402` `PAYMENT_PLAN_DOWN_PAYMENT_DECLINED` and no plan is created. When its outcome cannot be confirmed, the request answers `502` `PAYMENT_PLAN_DOWN_PAYMENT_UNCONFIRMED`, no plan is created, and the charge is not reversed because it can still settle. The response's `down_payment_invoice_id` is the id of that down-payment invoice; once its charge has settled, send the same request again, with the same `Idempotency-Key` or a new one, and ChaChing handles the invoice as the next item describes.
- Before it charges a down payment, ChaChing looks for an earlier down-payment invoice of the same customer and the same amount that no plan uses, which is what a create that did not finish leaves behind. When that invoice was paid with the same `payment_method` and was not refunded, the new plan uses that payment: nothing is charged again, and installment `0` carries that earlier invoice in `invoice` and the time of that payment in `paid_at`. When its charge failed, ChaChing voids it and charges the down payment again. When that invoice is still open without a confirmed outcome, when it was settled with a different payment method or without a card payment, when more than one such invoice was paid, or when a failed invoice cannot be voided, the request answers `409` `PAYMENT_PLAN_DOWN_PAYMENT_PENDING`, no plan is created and nothing is charged. From the next UTC day, an earlier down-payment invoice that is still unpaid answers `409` `PAYMENT_PLAN_CUSTOMER_DELINQUENT` instead, until it is paid or voided. An earlier down-payment invoice of a different amount is not matched, so check the customer's invoices before you change `down_payment_amount` after a `502`.
- `payment_method` becomes the customer's default payment method, and the installments are charged against it.
- ChaChing sends `payment_plan.created` and `payment_plan.activated`, and, when there is a down payment, `payment_plan.installment_paid` for installment `0`.
- When there is a down payment, ChaChing emails the customer a receipt for it. See **Emails Your Customer Receives**.

## The Card on File

A plan with no down payment runs entirely on a card the customer saved earlier. ChaChing does not run a $0.00 card verification. It accepts the saved card only when the card was saved through an approved authorization that established it for automatic payments; otherwise the request answers `409` `PAYMENT_PLAN_CREDENTIAL_NOT_ESTABLISHED` and nothing is charged. Create the plan with a down payment — its charge establishes the card — or ask the customer to enter the card again. On the NMI gateway, a card added to the customer vault without a transaction is refused this way.

Payment Plans run on accounts whose payment gateway is Dejavoo or NMI. On any other gateway the request answers `400` `PAYMENT_PLAN_GATEWAY_NOT_SUPPORTED`.

## When a Plan Cannot Be Created

- Your failed-payment outcome (`subscription_state_on_payment_failure`) is `unpaid`: `409` `PAYMENT_PLAN_UNPAID_OUTCOME_NOT_SUPPORTED`.
- Your retry schedule (`payment_retries`) sums to less than 7 days: `409` `PAYMENT_PLAN_DUNNING_SCHEDULE_TOO_SHORT`.
- The customer's account is in the late-payment warning or in a failed-payment outcome, or it carries an invoice unpaid for one day or more: `409` `PAYMENT_PLAN_CUSTOMER_DELINQUENT`. Collect the balance first.
- The request has `payment_method` and a down payment, and automatic payments are turned off for the customer in the billing engine: `409` `PAYMENT_PLAN_AUTO_PAY_OFF`. Create the plan without a down payment.
- The customer was already invoiced through a date on or after `start_date`, because an invoice was generated ahead of time for the customer's account: `409` `PAYMENT_PLAN_ACCOUNT_INVOICED_AHEAD`. The message names that date; choose a `start_date` after it.
- The customer's billing state cannot be read: `502` `PAYMENT_PLAN_BILLING_ENGINE_UNAVAILABLE`. Nothing was created; the request is safe to retry with a new `Idempotency-Key`.

Change the first two through `PUT /revenue-recovery`; see [Configure Failed-Payment Settings](./subscription-lifecycle.md).

## Duplicate Requests

`POST /payment-plans` and `POST /payment-plans/{id}/payments` accept an `Idempotency-Key` header of 1 to 255 printable ASCII characters with no spaces; any other value answers `400` `IDEMPOTENCY_KEY_INVALID`. A key is scoped to your account and to the endpoint, and it is kept for 24 hours.

- The same key with the same request returns the stored result — the same status code and body — with the response header `Idempotent-Replayed: true`. Nothing is created or charged again, and no webhook is sent again.
- The same key with a different request answers `422` `IDEMPOTENCY_KEY_REUSED`.
- The same key while the first request is still running answers `409` `IDEMPOTENCY_KEY_IN_PROGRESS`.
- When the first request never completed, the key answers `502` `IDEMPOTENCY_REQUEST_INCOMPLETE` after ten minutes. Read the plan before retrying with a new key.
- A request refused with a `4xx` other than `402` is not stored: fix it and retry with the same key.
- A `402`, `500` or `502` answer is stored and replayed for the same key for 24 hours, so a retry after one of them needs a new key. The exception is `502` `PAYMENT_PLAN_DOWN_PAYMENT_UNCONFIRMED`: it is not stored, and the same key is evaluated again on the next request.

---

# Collect the First Installment on a Terminal

A plan created without `payment_method` collects its deposit on one of your Dejavoo terminals: the down payment when the plan has one, else its first installment. The customer pays it with their card at the counter. ChaChing saves that card, activates the plan, and charges the remaining installments to the same card automatically — installments 1 onward after a down payment, installments 2 onward otherwise — exactly like a plan created with a card on file.

## Before You Start

- Register the terminal with `POST /dejavoo/terminals` and keep the terminal's `id`. The Dejavoo Terminals endpoints are described in the [API Reference](./api.json).
- The terminal must be enabled (`enabled: true`) and turned on and connected. ChaChing checks its connection before each deposit.
- Your ISO sets up each terminal TPN with SPIn Cloud enabled, FEED enabled for the TPN, a Card Entry Screen Timeout of 90 seconds, Tip and Tax disabled, and the same card fee as your e-commerce TPN.
- The three deposit endpoints below accept only your account's secret API key (`sk_…`) in the `cc-api-key` header. A publishable key (`pk_…`) answers `403` `SECRET_API_KEY_REQUIRED`, and a request that also sends an `Authorization` header answers `409` with no `error` field.

## Start the Deposit

1. Create the plan without `payment_method`. It answers `201` in status `awaiting_deposit`, and its `deposit` object carries the plan's `reference`, its `kind`, the `amount` the terminal collects, `status: "pending"` and `attempt: null`. `kind` is `down_payment` when the plan has a down payment, and `amount` is then the down payment; otherwise `kind` is `installment_1` and `amount` is the amount of installment 1. Until the deposit is collected, a down payment counts as an unpaid installment: `remaining_balance` is `total_amount`, `installments_remaining` counts it, and `next_due_date` is 00:00:00 UTC of the day the plan was created.
2. Call `POST /payment-plans/{id}/deposit/terminal` with the terminal:

   ```json
   {
     "terminal_id": "3f6b2a4e-9c1d-4e8f-a2b7-5d0c9e1f4a63"
   }
   ```

   It answers `202` with the Payment Plan object, and `deposit.attempt.state` is `in_flight`. The terminal shows the amount, including the card surcharge your account adds; the customer confirms it and presents the card.
3. Poll `GET /payment-plans/{id}` every 2 to 3 seconds until `deposit.attempt.state` is no longer `in_flight`. The request allows the customer up to 180 seconds at the terminal; the terminal itself times out the card prompt earlier (about 30 seconds idle plus 90 seconds of card entry on the P1, measured 2026-10-14). When the terminal's answer is lost, ChaChing keeps confirming the sale for up to 60 more seconds, so an attempt leaves `in_flight` at most 240 seconds after it started.

The `deposit` object while the customer is at the terminal:

```json
"deposit": {
  "reference": "K7Q2M9XA",
  "kind": "installment_1",
  "amount": 20000,
  "status": "pending",
  "transaction_reference": null,
  "collected_at": null,
  "attempt": {
    "id": "ppda_4f9c2e1b7a6d3c8e5f0a1b2c",
    "object": "payment_plan_deposit_attempt",
    "terminal": "3f6b2a4e-9c1d-4e8f-a2b7-5d0c9e1f4a63",
    "reference_id": "K7Q2M9XA01",
    "state": "in_flight",
    "started": 1760400000,
    "resolved": null,
    "decline_code": null,
    "decline_message": null,
    "surcharge_amount": 0,
    "amount_charged": 0
  }
}
```

## Deposit Attempt States

`deposit.attempt` is the plan's latest attempt, or `null` before the first one. The state `cancelled` is spelled with two `l`s, the way the terminal reports it; the plan status `canceled` keeps one. Only `declined`, `cancelled`, `not_charged` and `failed` mean that nothing was charged; while an attempt is `in_flight` or `resolving` — including after a communication error between the terminal and the card host — the card can have been charged.

| **State** | **Meaning** | **What you do** |
| --- | --- | --- |
| `in_flight` | The customer is at the terminal, or ChaChing is confirming the sale. | Keep polling. To stop the sale while the card prompt is open, call `POST /payment-plans/{id}/deposit/cancel`. |
| `approved` | The deposit is paid and the card is saved: the down payment, recorded as installment `0`, or installment 1. The plan is `active`, or `paid_off` for a plan with one installment and no down payment, and `deposit.status` is `collected`. `deposit.transaction_reference` is the gateway transaction id of the sale, or the attempt's `reference_id` when the sale was confirmed only through the iPOSpays Transaction Status Check. | Nothing. |
| `declined` | The card was declined. Nothing was charged. | Start a new attempt, with another card when the customer has one. |
| `cancelled` | The sale was cancelled on the terminal or with the cancel endpoint. Nothing was charged. | Start a new attempt when the customer is ready. |
| `not_charged` | The terminal confirmed that no sale took place. | Start a new attempt. |
| `failed` | The terminal or the SPIn proxy refused the request without producing an authorization, for example because the terminal was busy with another sale, was not connected, or cannot accept that card type; nothing was charged. `decline_message` gives the reason. | Start a new attempt; when `decline_message` reads `busy; retry after <n> s`, wait that long first. |
| `resolving` | The terminal's answer did not arrive, or did not say whether the card was charged, and the outcome is not known yet. The card can have been charged. | Call `POST /payment-plans/{id}/deposit`. Do not ask the customer to pay again: a new attempt is refused until this one is resolved. |
| `approved_no_credential` | The card was charged, but ChaChing has not saved it for the next installments yet. The plan stays `awaiting_deposit`. | Call `POST /payment-plans/{id}/deposit` again five minutes later. Do not charge the customer again. |
| `partial_approval` | The card was charged an amount different from the deposit (`deposit.amount`), or only part of it. The plan is not activated. | Void the sale on the terminal, then start a new attempt. |
| `force_closed` | ChaChing staff closed an attempt whose outcome could not be confirmed. The plan stays `awaiting_deposit`; you can start a new attempt or cancel the plan. The card can have been charged: check the sale in your Dejavoo (iPOSpays) account. | Start a new attempt only after that check. |

Fields of the attempt object:

| **Field** | **Meaning** |
| --- | --- |
| `id` | The attempt id, `ppda_…`. |
| `terminal` | The `id` of the terminal the attempt ran on. |
| `reference_id` | The reference and invoice number the terminal and your Dejavoo (iPOSpays) reports show for this sale: the plan's `reference` followed by the two-digit attempt number. |
| `state` | One of the states above. |
| `started` | When the attempt started, Unix seconds. |
| `resolved` | When its outcome was read, Unix seconds; `null` while it is not known. |
| `decline_code` | The terminal's status code for a `declined`, `cancelled`, `failed` or `partial_approval` attempt; `null` otherwise. |
| `decline_message` | The terminal's message for those same states; `null` otherwise. |
| `surcharge_amount` | The card surcharge the terminal added, in integer cents; `0` when none or not known. |
| `amount_charged` | The total charged to the card, in integer cents; `0` until the sale is approved. |

## Confirm a Deposit Whose Answer Was Lost

`POST /payment-plans/{id}/deposit`, with an empty body, asks the terminal and iPOSpays what happened to the plan's latest attempt and answers `200` with the Payment Plan object.

- For a `resolving` attempt it confirms the outcome. A sale that went through activates the plan; a sale that never took place ends the attempt `not_charged`. When the outcome still cannot be confirmed, the attempt stays `resolving`: call the endpoint again a minute later.
- For an `approved_no_credential` attempt it tries again to save the card and activate the plan.
- For an attempt that is still `in_flight` it answers `409` `PAYMENT_PLAN_DEPOSIT_IN_FLIGHT`; keep polling instead.
- For an attempt that already ended while the plan still awaits its deposit, it returns the plan unchanged.
- A plan with no attempt answers `409` `PAYMENT_PLAN_DEPOSIT_NO_ATTEMPT`.
- A plan that is no longer `awaiting_deposit`, because it was activated or canceled, answers `409` `PAYMENT_PLAN_NOT_AWAITING_DEPOSIT`, and ChaChing asks nothing of the terminal or iPOSpays.
- The confirmation can take more than a minute. When your request times out before it answers, the confirmation still completes: read the plan with `GET /payment-plans/{id}`. A repeat sent while the confirmation is still running answers `409` `PAYMENT_PLAN_BUSY`.

## Cancel the Sale at the Terminal

`POST /payment-plans/{id}/deposit/cancel`, with an empty body, asks the terminal to cancel the sale while its card prompt is open, and answers `202` with the Payment Plan object. Keep polling: the attempt ends `cancelled` when the terminal confirms the cancellation, and `approved` when the customer had already completed the payment. Once the terminal has answered, or when there is no attempt in `in_flight`, the endpoint answers `409` `PAYMENT_PLAN_DEPOSIT_NOT_IN_FLIGHT`.

## Rules

- One attempt runs at a time on a plan, and one on a terminal. A second start answers `409` `PAYMENT_PLAN_DEPOSIT_IN_FLIGHT` for the same plan, and `409` `DEJAVOO_TERMINAL_BUSY` for the same terminal.
- A plan accepts up to 99 attempts; the next one answers `409` `PAYMENT_PLAN_DEPOSIT_ATTEMPTS_EXHAUSTED`.
- A plan whose latest attempt is not resolved yet (`in_flight`, `resolving` or `approved_no_credential`) cannot be canceled: `POST /payment-plans/{id}/cancel` answers `409` `PAYMENT_PLAN_DEPOSIT_IN_FLIGHT` with the message `A terminal deposit attempt is unresolved. Resolve it with POST /payment-plans/{id}/deposit before canceling the plan.` until the attempt reaches a final state. Resolve it first with `POST /payment-plans/{id}/deposit`.
- When the plan is activated, ChaChing sends `payment_plan.activated` and `payment_plan.installment_paid` for the installment the deposit paid — sequence `0` for a down payment, else sequence `1` — (and `payment_plan.completed` for a plan with one installment and no down payment), and emails the customer the receipt for the down payment or for payment 1.
- Every Payment Plan object, in the responses and in the webhooks, carries `deposit.attempt`.

---

# Read Plans and Installments

- `GET /payment-plans` lists your plans, newest first, with the page parameters `page` and `take` (`take` at most `50`). Filter with `search`, a URL-encoded JSON object: `{"customer": "cus_739c07713fd4b68565a969cd", "status": "past_due,defaulted"}`. `status` takes one status or a comma-separated list. A `search` value that is not valid JSON answers `400`.
- `GET /payment-plans/{id}` returns one plan.
- `GET /payment-plans/{id}/installments` returns the plan's installments in ascending `sequence` order, the down payment first when there is one.

The Payment Plan object carries `description`: the text sent when the plan was created, or `null` when none was sent. It cannot be changed afterwards.

An installment's `surcharge_amount` and `amount_charged` report the card surcharge of the charges that paid it; see [Card Surcharges](#card-surcharges).

An installment's `invoice` is `null` for a down payment collected on a terminal.

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
| `payment_method` | no | The card to charge for this payment only; it must belong to the plan's customer. Default: the customer's default payment method. Ignored when `external` is `true`. Send `{ "type": "terminal", "terminal_id": "<terminal id>" }` instead to take the payment on one of your Dejavoo terminals; see **Take a Payment on a Terminal**. |
| `external` | no | `true` records money you collected outside ChaChing; nothing is charged. Default `false`. |

The amount is applied first to the plan's billed installments that are not fully paid, oldest first — a smaller amount pays part of an installment — and then to the part of the plan not billed yet, the principal.

- **Extra payment.** An amount below `remaining_balance`. When it reaches the principal, the installment amount and the due dates do not change: the plan ends sooner, and its last installment is reduced to what is left.
- **Payoff.** An amount equal to `remaining_balance`. It pays every open installment, closes the rest of the schedule, and moves the plan to `paid_off`.
- **Several charges.** A payment spanning several billed installments, or billed installments and the principal, is charged as one charge per installment invoice plus one charge for the principal, in that order. When the first charge is declined the request answers `402` `PAYMENT_DECLINED` and nothing is applied. When a later charge is declined, the charges already approved stay applied and the request answers `201` with the `amount` actually applied. When a later charge's outcome cannot be confirmed, the request still answers `201`, with `status` `processing` and `amount` `0` even when earlier charges of the same payment were approved: the payment is recorded only once every one of its charges is resolved. Do not retry with a new `Idempotency-Key`. Wait for `payment_plan.payment_succeeded`, which ChaChing sends with the amount actually applied once it finishes recording the payment, and read the plan's installments when the event does not arrive.
- **The principal has no invoice.** The principal part of a payment belongs to no invoice: it does not appear on the invoice endpoints and sends no `invoice.*` webhook. `payment_plan.payment_succeeded` reports it.
- A payment on a `past_due` or `defaulted` plan is never refused because of its status. While an earlier payment's schedule rebuild is still being completed, a further payment answers `409` `PAYMENT_PLAN_REPLAN_PENDING`; retry it later.

The Payment object's `status` tells you how far the recording went:

| **`status`** | **Meaning** |
| --- | --- |
| `applied` | Every charge was approved and the whole amount was applied. |
| `partially_applied` | A later charge was declined or not attempted, or the terminal approved less than the amount sent. `amount` is the part that was applied. |
| `processing` | At least one charge has no confirmed outcome yet. Nothing is recorded yet and `amount` reads `0`. ChaChing finishes recording the payment and then sends `payment_plan.payment_succeeded` with the final `status` and `amount`. |
| `declined` | The first charge was declined, or the terminal sale was declined. Nothing was applied. |
| `not_attempted` | The first charge was not attempted, or the terminal sale did not take place: it was cancelled, the terminal confirmed that nothing was charged, the terminal refused the request, or ChaChing staff closed it. Nothing was applied. A payment also reads `not_attempted` when the terminal approved and charged the card but the plan could absorb none of the amount: `terminal.state` is then `approved` or `partial_approval`, `terminal.amount_charged` carries the collected amount and the payment's own `amount_charged` is `0`, so do not ask the customer to pay again. |
| `unconfirmed` | The outcome of the first charge could not be confirmed. Nothing was recorded; read the plan before retrying. |

The `201` of `POST /payment-plans/{id}/payments` and `payment_plan.payment_succeeded` carry only `processing`, `applied` and `partially_applied`. `GET /payment-plans/{id}/payments/{payment_id}` returns any payment of the plan with its `status` as recorded, so it can also read `declined`, `not_attempted` or `unconfirmed`.

Each recorded payment sends `payment_plan.payment_succeeded`, one `payment_plan.installment_paid` for every installment it paid in full, and `payment_plan.completed` when it is a payoff. For one payment they are sent in this order: `payment_plan.installment_paid` for each installment it paid in full, then `payment_plan.payment_succeeded`, then `payment_plan.completed`. A declined or refused payment sends none of them. When a request ends before its payment is fully recorded (for example a `502` `PAYMENT_PLAN_PAYMENT_UNCONFIRMED`, or a `201` whose later charge could not be confirmed), ChaChing finishes recording the payment later and sends these events then.

## Take a Payment on a Terminal

A customer who comes to your counter can pay an installment, make an extra payment or pay the plan off on one of your Dejavoo terminals. The amount is applied like any other payment: billed installments first, oldest first, then the principal. The sale does not change the card saved on the plan, and installments keep being charged to that card.

1. Call `POST /payment-plans/{id}/payments` with the terminal in `payment_method`, using your account's secret API key (`sk_…`):

   ```json
   {
     "amount": 50000,
     "payment_method": { "type": "terminal", "terminal_id": "3f6b2a4e-9c1d-4e8f-a2b7-5d0c9e1f4a63" }
   }
   ```

   It answers `201` with the Payment object in `status: processing`, with `amount: 0` and a `terminal` object whose `state` is `in_flight`. The terminal shows the amount, including the card surcharge your account adds; the customer confirms it and presents the card.
2. Poll `GET /payment-plans/{id}/payments/{payment_id}` every 2 to 3 seconds until `terminal.state` is no longer `in_flight`. The request allows the customer up to 180 seconds at the terminal; the terminal itself times out the card prompt earlier (about 30 seconds idle plus 90 seconds of card entry on the P1, measured 2026-10-14). When the terminal's answer is lost, ChaChing keeps confirming the sale for up to 60 more seconds, so a payment leaves `in_flight` at most 240 seconds after it started.
3. When the sale is approved, ChaChing records the payment: `status` becomes `applied` and `amount` the amount applied, `surcharge_amount` and `amount_charged` report the surcharge the terminal added, ChaChing sends `payment_plan.payment_succeeded` (with `payment_plan.installment_paid` for each installment it paid in full and `payment_plan.completed` for a payoff), and the customer receives the receipt. On this payment `external` reads `false` and `payment_method` reads `null`.

The `terminal` object while the customer is at the terminal:

```json
"terminal": {
  "id": "ppda_9b2e4c7a1f3d5e8b0c6a2d4f",
  "object": "payment_plan_terminal_attempt",
  "terminal": "3f6b2a4e-9c1d-4e8f-a2b7-5d0c9e1f4a63",
  "reference_id": "Q4T8M2ZK03",
  "state": "in_flight",
  "started": 1760500000,
  "resolved": null,
  "decline_code": null,
  "decline_message": null,
  "surcharge_amount": 0,
  "amount_charged": 0,
  "transaction_id": null,
  "rrn": null,
  "auth_code": null,
  "card_last4": null,
  "card_brand": null,
  "card_entry_type": null
}
```

| **`terminal.state`** | **Meaning** | **What you do** |
| --- | --- | --- |
| `in_flight` | The customer is at the terminal, or ChaChing is confirming the sale. The payment reads `processing`. | Keep polling. To stop the sale while the card prompt is open, call `POST /payment-plans/{id}/payments/{payment_id}/cancel`. |
| `approved` | The sale was approved. Once ChaChing has recorded it the payment reads `applied`; when the terminal approved more than the amount sent, the amount sent is applied and `amount_charged` includes the difference. When the plan could absorb none of the amount, the terminal approved and charged the card but the payment reads `not_attempted`: `terminal.state` is `approved` or `partial_approval`, `terminal.amount_charged` carries the collected amount and the payment's own `amount_charged` is `0`, so do not ask the customer to pay again. | Nothing. When the payment still reads `processing` a minute later, call `POST /payment-plans/{id}/payments/{payment_id}/resolve`. |
| `partial_approval` | The terminal approved less than the amount sent. ChaChing records the approved amount and the payment reads `partially_applied`. | Take the rest as a new payment. |
| `declined` | The card was declined. The payment reads `declined`; nothing was applied. | Take a new payment, with another card when the customer has one. |
| `cancelled` | The sale was cancelled on the terminal or with the cancel endpoint. The payment reads `not_attempted`; nothing was charged. | Take a new payment when the customer is ready. |
| `not_charged` | The terminal confirmed that no sale took place. The payment reads `not_attempted`. | Take a new payment. |
| `failed` | The terminal or the SPIn proxy refused the request without producing an authorization; nothing was charged. The payment reads `not_attempted` and `decline_message` gives the reason. | Take a new payment; when `decline_message` reads `busy; retry after <n> s`, wait that long first. |
| `resolving` | The terminal's answer did not arrive, or did not say whether the card was charged. The card can have been charged. The payment reads `processing`. | Call `POST /payment-plans/{id}/payments/{payment_id}/resolve`. Do not ask the customer to pay again. |
| `approved_no_credential` | The card was charged, but ChaChing has not confirmed the amount or recorded the payment yet. The payment reads `processing`. | Call `POST /payment-plans/{id}/payments/{payment_id}/resolve` again five minutes later. Do not charge the customer again. |
| `force_closed` | ChaChing staff closed a payment whose outcome could not be confirmed. The payment reads `not_attempted`. The card can have been charged: check the sale in your Dejavoo (iPOSpays) account. | Take a new payment only after that check. |

Fields of the `terminal` object (`null` on a payment that was not taken on a terminal):

| **Field** | **Meaning** |
| --- | --- |
| `id` | The terminal attempt id, `ppda_…`. |
| `terminal` | The `id` of the terminal the sale ran on. |
| `reference_id` | The reference and invoice number the terminal and your Dejavoo (iPOSpays) reports show for this sale: ten characters, generated for each terminal payment. |
| `state` | One of the states above. |
| `started` | When the sale started, Unix seconds. |
| `resolved` | When its outcome was read, Unix seconds; `null` while it is not known. |
| `decline_code` | The terminal's status code for a `declined`, `cancelled`, `failed` or `partial_approval` sale; `null` otherwise. |
| `decline_message` | The terminal's message for those same states; `null` otherwise. |
| `surcharge_amount` | The card surcharge the terminal added, in integer cents; `0` when none or not known. |
| `amount_charged` | The total charged to the card, in integer cents; `0` until the sale is approved. |
| `transaction_id` | The iPOSpays transaction id of the sale, as your Dejavoo reports show it; `null` until it is known. |
| `rrn` | The sale's retrieval reference number; `null` until it is known. |
| `auth_code` | The authorization code the terminal returned for the sale; `null` until it is known. |
| `card_last4` | The last four digits of the card; `null` until they are known. |
| `card_brand` | The card brand in lowercase, for example `visa`; `null` until it is known. |
| `card_entry_type` | How the card was read, as the terminal reports it, for example `Contactless` or `Chip`; `null` until it is known. |

### Confirm a Terminal Payment Whose Answer Was Lost

`POST /payment-plans/{id}/payments/{payment_id}/resolve`, with an empty body, asks the terminal and iPOSpays what happened to the sale and answers `200` with the Payment object.

- For a `resolving` sale it confirms the outcome. A sale that went through is recorded; a sale that never took place ends `not_charged` and the payment `not_attempted`. When the outcome still cannot be confirmed, the sale stays `resolving`: call the endpoint again a minute later.
- For an `approved_no_credential` sale, or an `approved` one whose payment still reads `processing`, it records the payment.
- For a sale that is still `in_flight` it answers `409` `PAYMENT_PLAN_TERMINAL_PAYMENT_IN_FLIGHT`; keep polling instead.
- For a payment that no longer reads `processing`, it returns the payment unchanged.
- For a payment that was not taken on a terminal it answers `409` `PAYMENT_PLAN_PAYMENT_NOT_TERMINAL`.
- The confirmation can take more than a minute. When your request times out before it answers, the confirmation still completes: read the payment with `GET /payment-plans/{id}/payments/{payment_id}`. A repeat sent while the confirmation is still running answers `409` `PAYMENT_PLAN_BUSY`.

### Cancel the Sale at the Terminal

`POST /payment-plans/{id}/payments/{payment_id}/cancel`, with an empty body, asks the terminal to cancel the sale while its card prompt is open, and answers `202` with the Payment object. Keep polling: the sale ends `cancelled` when the terminal confirms the cancellation, and `approved` when the customer had already completed the payment. Once the terminal has answered, or when the sale is not `in_flight`, the endpoint answers `409` `PAYMENT_PLAN_TERMINAL_PAYMENT_NOT_IN_FLIGHT`.

### Rules for Terminal Payments

- `POST /payment-plans/{id}/payments` with a terminal, the resolve endpoint and the cancel endpoint accept only your account's secret API key (`sk_…`): a publishable key (`pk_…`) answers `403` `SECRET_API_KEY_REQUIRED`, and a request that also sends an `Authorization` header answers `409` with no `error` field. `GET /payment-plans/{id}/payments/{payment_id}` accepts any API key of your account.
- Only a plan on the Dejavoo gateway takes terminal payments; any other plan answers `409` `PAYMENT_PLAN_TERMINAL_NOT_SUPPORTED`. `external: true` with a terminal answers `400` `PAYMENT_PLAN_TERMINAL_EXTERNAL_NOT_ALLOWED`.
- The terminal must be registered, enabled, turned on and connected, exactly as for a deposit: the same codes apply (`DEJAVOO_TERMINAL_NOT_FOUND`, `DEJAVOO_TERMINAL_DISABLED`, `DEJAVOO_TERMINAL_OFFLINE`, `SPIN_AUTH_KEY_UNREADABLE`, `DEJAVOO_TERMINAL_ISV_ID_INVALID`, `SPIN_CREDENTIALS_REJECTED`, `SPIN_UNREACHABLE`).
- While a terminal payment of the plan reads `processing`, every new payment on the plan answers `409` `PAYMENT_PLAN_PAYMENT_PENDING_CONFIRMATION`, and `POST /payment-plans/{id}/cancel` answers `409` `PAYMENT_PLAN_PAYMENT_PENDING_CONFIRMATION` with the message `A terminal payment on this plan is unresolved. Resolve it with POST /payment-plans/{id}/payments/{payment_id}/resolve before canceling the plan.` A sale on a terminal that is already running another one answers `409` `DEJAVOO_TERMINAL_BUSY`.
- A plan accepts up to 99 terminal attempts, its deposit attempts included; the next one answers `409` `PAYMENT_PLAN_TERMINAL_ATTEMPTS_EXHAUSTED`.
- A request repeated with the same `Idempotency-Key` answers the stored `201` again and starts no second sale; read the payment with `GET /payment-plans/{id}/payments/{payment_id}` for its current state.
- The amount is applied to the plan as it stands when ChaChing records the sale. When an installment was paid another way, or its invoice was voided, before the payment was recorded, the amount goes to the next open installment or to the principal. An approved amount the plan can no longer absorb is not applied; ChaChing's team is alerted to refund it at the terminal. When the plan absorbs only part of an approved sale the payment reads `partially_applied`, and when it absorbs none of it, `not_attempted`.

---

# Card Surcharges

On an account that charges through iPOSpays (Dejavoo), every card charge can carry a card surcharge, payment plan charges included; [Card Surcharges](./card-surcharges.md) describes the rule for every charge. Whether a card is surcharged, and how much, is decided by the S.T.E.A.M configuration of your account in the iPOSpays portal and by the card used; in our tests a debit card was not surcharged. ChaChing records the surcharge separately and never changes a plan amount because of it.

These payment plan charges carry it:

- the down payment;
- every automatic installment charge and every automatic retry;
- every installment invoice paid from the payment page or from the dashboard;
- every charge of `POST /payment-plans/{id}/payments` that is not `external`, the principal part included, and every payment taken on a terminal.

A payment plan invoice paid with a card or with Google Pay through any route — the automatic charge, `POST /invoices/{id}/pay`, the payment page or the dashboard — carries your account's surcharge.

Accounts that charge through NMI never carry a surcharge, and neither does a payment recorded with `external: true`, because nothing is charged.

The plan amounts never include the surcharge: `total_amount`, `installment_amount`, every installment `amount`, `amount_paid`, `remaining_balance`, the invoice totals and the payoff amount stay as agreed.

Two fields report it, always present and in integer cents, on the Installment, Payment, Transaction and Invoice objects:

| **Field** | **Meaning** |
| --- | --- |
| `surcharge_amount` | The card surcharge added to the charge or charges. `0` when none was added. |
| `amount_charged` | The total charged to the card. It equals `amount_paid` on an installment, `amount` on a payment and on a transaction, and `amount_paid` on an invoice, when nothing was added. |

For example, a $1.00 payment that carried a 3 cent surcharge reads `amount: 100`, `surcharge_amount: 3`, `amount_charged: 103`; the installment that payment paid reads `amount_paid: 100`, `surcharge_amount: 3`, `amount_charged: 103`.

- `amount_charged` can exceed the amount plus `surcharge_amount` when the gateway adds another S.T.E.A.M amount for your account. That difference is included in `amount_charged` and is not itemised.
- A payment that did not succeed never shows a surcharge.
- ChaChing reads the surcharge from the gateway after the charge. While it cannot be read yet, `surcharge_amount` is `0` and `amount_charged` equals the amount paid. ChaChing keeps trying for up to 7 days, and the objects show the real values as soon as the read succeeds. When the surcharge of a charge still cannot be read after those 7 days, or at once when the charge cannot be matched, the installment and payment objects keep `surcharge_amount` at `0` and `amount_charged` equal to the amount paid, and that does not change later. The exact amount charged to the card is then the one shown for that transaction in your Dejavoo (iPOSpays) account.
- ChaChing does not refund a surcharge: there is no refund operation for payment plan charges.

Before the payer pays, the payment page of an open invoice and the dashboard's `Charge customer` sheet tell them that a card surcharge can be added; [Card Surcharges](./card-surcharges.md#what-your-customer-sees-before-paying) lists every notice and its exact wording.

In the dashboard, the transactions list, the transaction details and the invoice details show the surcharge and the total charged for a charge that carried one. The dashboard's Total revenue and a customer's total spend include the surcharge for accounts that charge through Dejavoo.

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

# Emails Your Customer Receives

ChaChing emails the plan's customer at the email address stored on the customer. These emails are best effort: a customer with no email address receives none, and an email that fails to send does not change the response, the plan or the webhooks. A receipt is not sent for a charge more than 48 hours old when the receipt step runs; for a deposit collected on a terminal, the 48 hours count from the moment the plan is activated.

**Receipts.** The customer receives one receipt for each of these card charges:

- the down payment, when the plan is created;
- the deposit collected on a terminal (the down payment, or the first installment), when the plan is activated;
- each installment charged automatically on its due date or by an automatic retry;
- each payment recorded with `POST /payment-plans/{id}/payments` that is not `external`, a payment taken on a terminal included. One receipt lists every charge of that payment;
- an installment invoice paid by card through ChaChing outside those endpoints: by your customer from the payment page, or by you from the dashboard or with `PUT /invoices/{id}`.

The subject is `Receipt from <your account name>: payment 3 of 9` for an installment, `Receipt from <your account name>: down payment` for the down payment, and `Receipt from <your account name>: payment plan payment` for a recorded payment that covers several charges or reaches the principal. `9` is the plan's `installment_count`. The receipt states your account name, your city and state when your account's billing contact holds both, the plan's `description`, the amount and currency, the date, the card brand and its last four digits, a `Surcharge` line after the payment lines when the charge carried a surcharge, with `Total charged` as the amount the card was charged, the remaining balance and the number of payments remaining (the receipt of the last installment, or of a payment that completes the plan, says the plan is paid in full instead), and a button that opens the invoice of the installment. The principal part of a payment has no invoice and no button. A deposit collected on a terminal has no invoice and no button. When the card details cannot be read, the receipt prints `Card on file`.

A payment recorded with `external: true` is not a card charge. The customer receives a `Payment recorded` email with the amount and the remaining balance, and no card details. When that payment pays the plan off, the email's subject says the plan is paid in full and its text says so instead of giving a remaining balance.

**Failed payments.** Each declined automatic attempt on an installment, retries included, sends the customer a notice with the invoice number and amount, the line `Installment 3 of 9 of your payment plan.`, the plan's `description`, the date of the next automatic retry when one is scheduled, a button to pay the invoice with another payment method, and this sentence: `You have at least 7 calendar days from the first failed attempt on this installment to pay it with another payment method.` For accounts that charge through iPOSpays, the notice also says: `When your card is charged, <your account name> can add a card surcharge to this amount.` Your account owner is copied on these notices.

A declined down payment sends the customer no email: the request answers `402` `PAYMENT_PLAN_DOWN_PAYMENT_DECLINED`, and no plan exists.

**Description.** A plan created without `description` prints `Payment plan` followed by the plan id in its place.

**Copies of receipts.** Receipts go to the customer only. ChaChing can turn on a copy of every receipt to your account owner; contact ChaChing to enable it.

These emails do not replace the webhooks: `payment_plan.installment_paid`, `payment_plan.installment_failed` and `payment_plan.payment_succeeded` report the same payments to your integration.

---

# Cancel a Plan

`POST /payment-plans/{id}/cancel` answers `200` with the plan in status `canceled` and `canceled_at` set, and ChaChing sends `payment_plan.canceled`. Installments not yet billed are removed at the end of the period already billed and are never charged. Cancellation is not a refund and does not forgive debt: amounts already invoiced stay collectible through `POST /payment-plans/{id}/payments`, and `remaining_balance` shows what is still owed.

This endpoint is the only way to end a plan early. A payment plan is not a subscription: the subscription endpoints answer `404` for it.

---

# Error Codes

Errors use the standard body `{ "statusCode", "message", "error", "timestamp", "path" }` plus the fields a row names. Branch on `statusCode` and `error`; `message` can change. A request body that fails field validation — a value below its minimum such as `total_amount: 0` or `installment_count: 0`, a negative `down_payment_amount`, a value of the wrong type, a `description` longer than 255 characters or containing a line break or another control character, or an unknown field such as `terminal` — and a `search` value that is not valid JSON are rejected with `400` and a body that carries no `error` field; `message` describes the problem.

| **Status** | **`error`** | **When** |
| --- | --- | --- |
| `400` | `PAYMENT_PLAN_SCHEDULE_INVALID` | Both or neither of `installment_count` and `installment_amount` were sent, `installment_count` is above `60` or larger than the amount to finance allows, `installment_amount` is larger than the amount to finance, or the amounts do not form a valid schedule. |
| `400` | `PAYMENT_PLAN_AMOUNT_INVALID` | `down_payment_amount` is not below `total_amount`, or a payment's `amount` is below `1`. |
| `400` | `PAYMENT_PLAN_INTERVAL_INVALID` | `interval` is not one of the values this environment accepts: `weekly`, `biweekly`, `monthly` and `annual` everywhere, and `daily` on test environments only. |
| `400` | `PAYMENT_PLAN_START_DATE_INVALID` | `start_date` is not after the day of the request, or `interval` is `annual` and `start_date` is 29 February (UTC). |
| `400` | `PAYMENT_PLAN_GATEWAY_NOT_SUPPORTED` | The account's payment gateway does not support Payment Plans. |
| `400` | `PAYMENT_PLAN_AMOUNT_EXCEEDS_BALANCE` | A payment is larger than what the plan accepts. Nothing is charged. |
| `400` | `PAYMENT_METHOD_REQUIRED` | A payment was sent with no `payment_method`, it is not external, and the customer has no default payment method. |
| `400` | `PAYMENT_PLAN_TERMINAL_EXTERNAL_NOT_ALLOWED` | A payment named a terminal and `external: true` together. |
| `400` | `IDEMPOTENCY_KEY_INVALID` | The `Idempotency-Key` header is malformed. |
| `402` | `PAYMENT_PLAN_DOWN_PAYMENT_DECLINED` | The down payment was declined. No plan was created. |
| `402` | `PAYMENT_DECLINED` | The first charge of a payment was declined. Nothing was applied. |
| `403` | `PAYMENT_PLANS_NOT_ENABLED` | Payment Plans are not enabled for this account. |
| `403` | `SECRET_API_KEY_REQUIRED` | A deposit endpoint or a terminal payment endpoint was called with a publishable (`pk_…`) API key. |
| `404` | `PAYMENT_PLAN_NOT_FOUND` | The plan id does not resolve for this account. |
| `404` | `CUSTOMER_NOT_FOUND` | The customer id does not resolve for this account. |
| `404` | `PAYMENT_METHOD_NOT_FOUND` | The payment method does not resolve, or it does not belong to the customer. |
| `404` | `DEJAVOO_TERMINAL_NOT_FOUND` | The `terminal_id` does not resolve to a terminal of this account. |
| `404` | `PAYMENT_PLAN_PAYMENT_NOT_FOUND` | The payment id does not resolve to a payment of this plan. |
| `409` | `PAYMENT_PLAN_CREDENTIAL_NOT_ESTABLISHED` | The saved card was not established for automatic payments. Create the plan with a down payment, or ask the customer to enter the card again. |
| `409` | `PAYMENT_PLAN_PAYMENT_METHOD_NOT_SUPPORTED` | On a create, the payment method is not a card saved with the account's current payment gateway. On `POST /payment-plans/{id}/payments`, on `PUT /invoices/{id}` and on a payment from the hosted invoice page or the dashboard of an invoice that carries an unpaid installment, the card that would be charged — `payment_method`, or the customer's default payment method when it is omitted — is not a card saved with the payment gateway the plan was created on; a plan created before you changed your payment gateway keeps accepting its own cards. Nothing was created or charged. Send a payment method returned by the payment-methods endpoints, or record money you collected yourself with `external: true` on `POST /payment-plans/{id}/payments`. |
| `409` | `PAYMENT_PLAN_UNPAID_OUTCOME_NOT_SUPPORTED` | Your failed-payment outcome is `unpaid`. |
| `409` | `PAYMENT_PLAN_DUNNING_SCHEDULE_TOO_SHORT` | Your `payment_retries` sum to less than 7 days. |
| `409` | `PAYMENT_PLAN_CUSTOMER_DELINQUENT` | The customer's account is in the late-payment warning or a failed-payment outcome, or carries an invoice unpaid for one day or more. |
| `409` | `PAYMENT_PLAN_AUTO_PAY_OFF` | A plan created with `payment_method` has a down payment and automatic payments are turned off for the customer in the billing engine. Nothing was created or charged. Create the plan without a down payment. |
| `409` | `PAYMENT_PLAN_ACCOUNT_INVOICED_AHEAD` | The customer was already invoiced through a date on or after `start_date`. The message names that date. Nothing was created; choose a `start_date` after it. |
| `409` | `PAYMENT_PLAN_DOWN_PAYMENT_PENDING` | An earlier down-payment invoice of the same amount for this customer is still open, or was settled in a way this request cannot use (a different payment method, no card payment, or more than one such invoice). Nothing was created or charged by this request. Check the customer's invoices and transactions: wait for an open charge to finish, send the `payment_method` that paid, or refund or void the invoices that should not count, then send the request again. |
| `409` | `PAYMENT_PLAN_NOT_PAYABLE` | The plan is `awaiting_deposit` or `paid_off`. |
| `409` | `PAYMENT_PLAN_NOT_CANCELABLE` | The plan is already `paid_off` or `canceled`. |
| `409` | `PAYMENT_PLAN_BUSY` | Another payment or a cancellation on this plan is in progress. Retry shortly. |
| `409` | `PAYMENT_PLAN_NOT_AWAITING_DEPOSIT` | A deposit was started or confirmed on a plan that is not `awaiting_deposit`. |
| `409` | `PAYMENT_PLAN_DEPOSIT_IN_FLIGHT` | The plan has a deposit attempt that is not resolved yet; for the confirm endpoint, one that is still `in_flight`. |
| `409` | `PAYMENT_PLAN_DEPOSIT_NO_ATTEMPT` | The plan has no deposit attempt to confirm. |
| `409` | `PAYMENT_PLAN_DEPOSIT_NOT_IN_FLIGHT` | There is no sale waiting at the terminal to cancel. |
| `409` | `PAYMENT_PLAN_DEPOSIT_ATTEMPTS_EXHAUSTED` | The plan reached 99 deposit attempts. |
| `409` | `DEJAVOO_TERMINAL_DISABLED` | The terminal is disabled. |
| `409` | `DEJAVOO_TERMINAL_BUSY` | The terminal is running another sale, a deposit or a payment. |
| `409` | `DEJAVOO_TERMINAL_OFFLINE` | The terminal is not connected. Turn it on and try again. |
| `409` | `SPIN_AUTH_KEY_UNREADABLE` | The terminal's stored Auth Key cannot be read. Re-enter it with `PATCH /dejavoo/terminals/{id}`. |
| `409` | `PAYMENT_PLAN_INSTALLMENT_BILLING_IN_PROGRESS` | The payment reaches the principal and the next installment is being billed or is about to bill: for a `daily` plan, its billing time is less than 1 hour away; for every other cadence, it is due within 24 hours. It can also answer for a `daily` plan whose billing time cannot be determined. Nothing is charged. |
| `409` | `PAYMENT_PLAN_PAYMENT_PENDING_CONFIRMATION` | An earlier payment on this plan is still being confirmed, a terminal payment that reads `processing` included. Nothing is charged; retry later. On `POST /payment-plans/{id}/cancel`, a terminal payment of the plan is unresolved. |
| `409` | `PAYMENT_PLAN_REPLAN_PENDING` | An earlier payment's schedule rebuild is still being completed. Nothing is charged; retry later. |
| `409` | `PAYMENT_PLAN_PAYMENT_NOT_ATTEMPTED` | The first charge was not attempted. Nothing is charged. |
| `409` | `PAYMENT_PLAN_TERMINAL_NOT_SUPPORTED` | A terminal payment was sent for a plan that is not on the Dejavoo gateway. |
| `409` | `PAYMENT_PLAN_PAYMENT_NOT_TERMINAL` | The resolve or cancel endpoint was called for a payment that was not taken on a terminal. |
| `409` | `PAYMENT_PLAN_TERMINAL_PAYMENT_IN_FLIGHT` | The customer is still at the terminal for this payment. |
| `409` | `PAYMENT_PLAN_TERMINAL_PAYMENT_NOT_IN_FLIGHT` | There is no sale of this payment waiting at the terminal to cancel. |
| `409` | `PAYMENT_PLAN_TERMINAL_ATTEMPTS_EXHAUSTED` | The plan reached 99 terminal attempts, deposit attempts included. |
| `409` | `IDEMPOTENCY_KEY_IN_PROGRESS` | A request with the same `Idempotency-Key` is still running. |
| `422` | `IDEMPOTENCY_KEY_REUSED` | The `Idempotency-Key` was already used with a different request. |
| `422` | `SPIN_CREDENTIALS_REJECTED` | SPIn did not accept the terminal's TPN and Auth Key. |
| `422` | `DEJAVOO_TERMINAL_ISV_ID_INVALID` | The terminal's `isv_id` is not 1 to 16 characters, as SPIn requires. Update it with `PATCH /dejavoo/terminals/{id}`. |
| `500` | `PAYMENT_PLAN_SCHEDULE_MISMATCH` | The registered schedule did not match the expected installment dates. No plan was created. |
| `502` | `PAYMENT_PLAN_DOWN_PAYMENT_UNCONFIRMED` | The down payment's outcome could not be confirmed. No plan was created and the charge is not reversed. `down_payment_invoice_id` is the id of the down-payment invoice to check. Once its charge has settled, send the same request again; this answer is not stored, so the same `Idempotency-Key` is accepted. A retry that itself answers a stored `502`, for example `PAYMENT_PLAN_BILLING_ENGINE_UNAVAILABLE`, needs a new key. A charge paid with the same `payment_method` is used for the new plan and is not taken again. From the next UTC day, the request answers `409` `PAYMENT_PLAN_CUSTOMER_DELINQUENT` instead until that invoice is paid or voided. Check the customer's invoices before you change the amount. |
| `502` | `PAYMENT_PLAN_PAYMENT_UNCONFIRMED` | The first charge's outcome could not be confirmed. It is not reversed; read the plan before retrying. |
| `502` | `PAYMENT_PLAN_BILLING_ENGINE_UNAVAILABLE` | Billing data could not be read. Nothing was created or charged; safe to retry with a new `Idempotency-Key`. |
| `502` | `PAYMENT_PLAN_CANCEL_INCOMPLETE` | The cancellation did not complete. The plan keeps its status and installments, but part of its schedule can already be stopped. Repeat the request: a repeat completes the rest. |
| `502` | `IDEMPOTENCY_REQUEST_INCOMPLETE` | The first request with this `Idempotency-Key` never completed. |
| `503` | `SPIN_UNREACHABLE` | SPIn could not be reached to check the terminal. Nothing was started; try again. |

---

# Payment Plan Events

Payment Plans send ten webhook event types on the same endpoints, envelope, signature and retry policy as every other ChaChing event. Their payloads are described in [Webhooks](./webhook.md).

| **Event** | **Sent when** | **`data`** |
| --- | --- | --- |
| `payment_plan.created` | A plan is created. | Payment Plan |
| `payment_plan.activated` | A plan starts running. A plan created with `payment_method` sends it right after `payment_plan.created`; a plan created without it sends it when its deposit, collected on a terminal, is confirmed. | Payment Plan |
| `payment_plan.installment_paid` | An installment is paid by its automatic charge or a retry, by a payment through `POST /payment-plans/{id}/payments`, by a payment of its invoice made through ChaChing (the hosted invoice page, the dashboard or `PUT /invoices/{id}`), by credit on the customer's account that settles its invoice, as the down payment, charged at creation or collected on a terminal (sequence `0`), or as the first installment collected on a terminal (sequence `1`). | Installment |
| `payment_plan.installment_failed` | An automatic charge attempt on an installment is declined, once per attempt, retries included. | Installment |
| `payment_plan.past_due` | The plan changes to `past_due`. | Payment Plan |
| `payment_plan.defaulted` | The plan changes to `defaulted`. | Payment Plan |
| `payment_plan.reactivated` | The plan returns to `active` from `past_due` or `defaulted`. | Payment Plan |
| `payment_plan.completed` | The plan changes to `paid_off`. | Payment Plan |
| `payment_plan.canceled` | The plan is canceled. | Payment Plan |
| `payment_plan.payment_succeeded` | `POST /payment-plans/{id}/payments` records a payment. | Payment |

An installment whose invoice is settled by credit on the customer's account, with no card charge, becomes `paid` with `amount_paid` equal to the amount the credit covered and `surcharge_amount` `0`. `paid_at` is the moment ChaChing recorded it, which can be up to about 10 minutes after the credit was applied. `payment_plan.installment_paid` is sent once and no receipt email is sent.

When an installment's invoice is reduced by an adjustment and the rest is paid, the installment becomes `paid` and `amount_paid` is the amount actually collected; the adjusted part is not counted as paid.

Read the plan with `GET /payment-plans/{id}` whenever you need its current state: the plan and installment endpoints are the source of truth, and deliveries can arrive in a different order than the changes happened.
