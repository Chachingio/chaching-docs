# Payment Plans on a Terminal

This page covers the Dejavoo terminal side of payment plans: registering terminals, collecting a plan's deposit at the counter, taking payments on a terminal and closing an attempt that stays open. It moved here from [Payment Plans](./payment-plans.md), which keeps a pointer at each former section.

---

# Register a Dejavoo Terminal

A terminal is a physical Dejavoo device of your account, addressed by its own TPN. These three endpoints accept only your account's secret API key (`sk_…`). A publishable key answers `403` `SECRET_API_KEY_REQUIRED`, and a request that also sends an `Authorization` header answers `409`.

| **Request** | **What it does** |
| --- | --- |
| `GET /dejavoo/terminals` | Lists the terminals of the account as `{ "data": [...] }`, with no `meta`. |
| `POST /dejavoo/terminals` | Registers a terminal from its `tpn`, `label` and 10-character `spin_auth_key`. ChaChing checks the pair with the terminal network before saving it, and a failed check saves nothing. |
| `PATCH /dejavoo/terminals/{id}` | Changes a terminal. The updatable fields are `label`, `enabled`, `spin_auth_key`, `register_id`, `isv_id` and `feed_enable_requested`. There is no delete: send `{ "enabled": false }` to disable it. |

The `dejavoo_terminal` object carries `object`, `id`, `tpn`, `label`, `enabled`, `register_id`, `isv_id`, `spin_auth_key_set`, `credential_rotated_at`, `last_probe_at`, `last_probe_status` (`Online`, `Offline` or `null`), `build`, `serial`, `feed_enable_requested_at` and `created`. Timestamps are Unix seconds. The Auth Key is write-only: `spin_auth_key_set` is always `true` and the key is never returned. The terminal object carries the full TPN; attempt objects never do.

| **Status** | **Error** | **When** |
| --- | --- | --- |
| `400` | no code | A field fails validation. |
| `400` | `DEJAVOO_TERMINAL_UPDATE_EMPTY` | `PATCH` with none of the updatable fields. |
| `401` | no code | The key is unknown or belongs to a deactivated account. |
| `403` | `SECRET_API_KEY_REQUIRED` | The key is a publishable key. |
| `404` | `DEJAVOO_TERMINAL_NOT_FOUND` | The `id` is not one of your terminals. |
| `409` | `DEJAVOO_NOT_CONFIGURED` | `POST` only: the account has no 12-digit e-commerce TPN set. Set it with `PUT /dejavoo/credentials`, see [Processor Credentials](./processor-credentials.md). |
| `409` | `TERMINAL_TPN_EXISTS` | The TPN is already registered. |
| `409` | `SPIN_AUTH_KEY_UNREADABLE` | The stored Auth Key cannot be read; send it again. |
| `422` | `SPIN_CREDENTIALS_REJECTED` | The TPN and Auth Key pair was refused. A wrong key and an unknown TPN carry the same error code. |
| `503` | `SPIN_UNREACHABLE` | The terminal network did not answer. |

A terminal `id` that is not a UUID answers `400` `Validation failed (uuid is expected)`. `PATCH` with an empty body answers `DEJAVOO_TERMINAL_UPDATE_EMPTY` before the id is looked up. There is no single-attempt read: `GET /dejavoo/terminal-attempts` with `search` is the only read of attempts.

---

# Collect the First Installment on a Terminal

A plan created without `payment_method` collects its deposit on one of your Dejavoo terminals: the down payment when the plan has one, else its first installment. The customer pays it with their card at the counter. ChaChing saves that card, activates the plan, and charges the remaining installments to the same card automatically — installments 1 onward after a down payment, installments 2 onward otherwise — exactly like a plan created with a card on file.

## Before You Start

- Register the terminal with `POST /dejavoo/terminals` and keep the terminal's `id`. See [Register a Dejavoo Terminal](#register-a-dejavoo-terminal).
- The terminal must be enabled (`enabled: true`) and turned on and connected. ChaChing checks its connection before each deposit.
- Your ISO sets up each terminal TPN with SPIn Cloud enabled, FEED enabled for the TPN, a Card Entry Screen Timeout of 90 seconds, Tip and Tax disabled, and the same card fee as your e-commerce TPN.
- The three deposit endpoints below accept only your account's secret API key (`sk_…`) in the `cc-api-key` header. A publishable key (`pk_…`) answers `403` `SECRET_API_KEY_REQUIRED`, and a request that also sends an `Authorization` header answers `409` with no `error` field.

## Start the Deposit

1. Create the plan without `payment_method`. It answers `201` in status `awaiting_deposit`, and its `deposit` object carries the plan's `reference`, its `kind`, the `amount` the terminal collects, `status: "pending"` and `attempt: null`. `kind` is `down_payment` when the plan has a down payment, and `amount` is then the down payment; otherwise `kind` is `installment_1` and `amount` is the amount of installment 1. Until the deposit is collected, a down payment counts as an unpaid installment: `remaining_balance` is `total_amount`, `installments_remaining` counts it, and `next_due_date` is the due date of the deposit: for a plan with a down payment, 00:00:00 UTC of the creation day in the account time zone; for a plan without one, its `start_date`.
2. Call `POST /payment-plans/{id}/deposit/terminal` with the terminal:

   ```json
   {
     "terminal_id": "3f6b2a4e-9c1d-4e8f-a2b7-5d0c9e1f4a63"
   }
   ```

   It answers `202` with the Payment Plan object, and `deposit.attempt.state` is `in_flight`. The terminal shows the amount, including the credit card surcharge your account adds when the card is a credit card; the customer confirms it and presents the card.
3. Poll `GET /payment-plans/{id}` every 2 to 3 seconds until `deposit.attempt.state` is no longer `in_flight`. The request allows the customer up to 180 seconds at the terminal; the terminal itself times out the card prompt earlier (about 30 seconds idle plus 90 seconds of card entry on the P1, measured 2026-10-14). When the terminal's answer is lost, ChaChing keeps confirming the sale for up to 60 more seconds, so an attempt leaves `in_flight` at most 240 seconds after it started.

`POST /payment-plans/{id}/deposit/terminal` also answers:

| **Status** | **`error`** | **When** |
| --- | --- | --- |
| `409` | `PAYMENT_PLAN_START_DATE_TOO_SOON` | Installment 1 would bill within an hour; create a plan with a later `start_date`. |
| `502` | `PAYMENT_PLAN_BILLING_ENGINE_UNAVAILABLE` | The billing state could not be read; nothing was started. Retry. |

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
| `force_closed` | The attempt was closed as not collected, by ChaChing staff or by your integration with `POST /dejavoo/terminal-attempts/{id}/force-close`, because its outcome could not be confirmed. The plan stays `awaiting_deposit`; you can start a new attempt or cancel the plan. The card can have been charged: check the sale in your Dejavoo (iPOSpays) account. | Start a new attempt only after that check. |

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
| `surcharge_amount` | The credit card surcharge the terminal added, in integer cents; `0` when none or not known. A debit card sale carries none. |
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
- Every Payment Plan object carries `deposit`: an object with `attempt` for a plan created without `payment_method`, and `null` for a plan created with one.
- A plan in `awaiting_deposit` accepts no payment, external payments included, until the deposit is collected on a terminal.
- An attempt that stays `resolving` because the terminal never reports its outcome can be listed and force-closed: see [Find and Force-Close a Stuck Terminal Attempt](#find-and-force-close-a-stuck-terminal-attempt).

---

# Take a Payment on a Terminal

A customer who comes to your counter can pay an installment, make an extra payment or pay the plan off on one of your Dejavoo terminals. The amount is applied like any other payment: billed installments first, oldest first, then the principal. The sale does not change the card saved on the plan, and installments keep being charged to that card.

1. Call `POST /payment-plans/{id}/payments` with the terminal in `payment_method`, using your account's secret API key (`sk_…`):

   ```json
   {
     "amount": 50000,
     "payment_method": { "type": "terminal", "terminal_id": "3f6b2a4e-9c1d-4e8f-a2b7-5d0c9e1f4a63" }
   }
   ```

   It answers `201` with the Payment object in `status: processing`, with `amount: 0` and a `terminal` object whose `state` is `in_flight`. The terminal shows the amount, including the credit card surcharge your account adds when the card is a credit card; the customer confirms it and presents the card.
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
| `force_closed` | The payment was closed as not collected, by ChaChing staff or by your integration with `POST /dejavoo/terminal-attempts/{id}/force-close`, because its outcome could not be confirmed. The payment reads `not_attempted`. The card can have been charged: check the sale in your Dejavoo (iPOSpays) account. | Take a new payment only after that check. |

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
| `surcharge_amount` | The credit card surcharge the terminal added, in integer cents; `0` when none or not known. A debit card sale carries none. |
| `amount_charged` | The total charged to the card, in integer cents; `0` until the sale is approved. |
| `transaction_id` | The iPOSpays transaction id of the sale, as your Dejavoo reports show it; `null` until it is known. |
| `rrn` | The sale's retrieval reference number; `null` until it is known. |
| `auth_code` | The authorization code the terminal returned for the sale; `null` until it is known. |
| `card_last4` | The last four digits of the card; `null` until they are known. |
| `card_brand` | The card brand in lowercase, for example `visa`; `null` until it is known. |
| `card_entry_type` | How the card was read, as the terminal reports it, for example `Contactless` or `Chip`; `null` until it is known. |

## Confirm a Terminal Payment Whose Answer Was Lost

`POST /payment-plans/{id}/payments/{payment_id}/resolve`, with an empty body, asks the terminal and iPOSpays what happened to the sale and answers `200` with the Payment object.

- For a `resolving` sale it confirms the outcome. A sale that went through is recorded; a sale that never took place ends `not_charged` and the payment `not_attempted`. When the outcome still cannot be confirmed, the sale stays `resolving`: call the endpoint again a minute later.
- For an `approved_no_credential` sale, or an `approved` one whose payment still reads `processing`, it records the payment.
- For a sale that is still `in_flight` it answers `409` `PAYMENT_PLAN_TERMINAL_PAYMENT_IN_FLIGHT`; keep polling instead.
- For a payment that no longer reads `processing`, it returns the payment unchanged.
- For a payment that was not taken on a terminal it answers `409` `PAYMENT_PLAN_PAYMENT_NOT_TERMINAL`.
- The confirmation can take more than a minute. When your request times out before it answers, the confirmation still completes: read the payment with `GET /payment-plans/{id}/payments/{payment_id}`. A repeat sent while the confirmation is still running answers `409` `PAYMENT_PLAN_BUSY`.

## Cancel the Sale at the Terminal

`POST /payment-plans/{id}/payments/{payment_id}/cancel`, with an empty body, asks the terminal to cancel the sale while its card prompt is open, and answers `202` with the Payment object. Keep polling: the sale ends `cancelled` when the terminal confirms the cancellation, and `approved` when the customer had already completed the payment. Once the terminal has answered, or when the sale is not `in_flight`, the endpoint answers `409` `PAYMENT_PLAN_TERMINAL_PAYMENT_NOT_IN_FLIGHT`.

## Rules for Terminal Payments

- `POST /payment-plans/{id}/payments` with a terminal, the resolve endpoint and the cancel endpoint accept only your account's secret API key (`sk_…`): a publishable key (`pk_…`) answers `403` `SECRET_API_KEY_REQUIRED`, and a request that also sends an `Authorization` header answers `409` with no `error` field. `GET /payment-plans/{id}/payments/{payment_id}` accepts the secret key like every other endpoint.
- Only a plan on the Dejavoo gateway takes terminal payments; any other plan answers `409` `PAYMENT_PLAN_TERMINAL_NOT_SUPPORTED`. `external: true` with a terminal answers `400` `PAYMENT_PLAN_TERMINAL_EXTERNAL_NOT_ALLOWED`.
- The terminal must be registered, enabled, turned on and connected, exactly as for a deposit: the same codes apply (`DEJAVOO_TERMINAL_NOT_FOUND`, `DEJAVOO_TERMINAL_DISABLED`, `DEJAVOO_TERMINAL_OFFLINE`, `SPIN_AUTH_KEY_UNREADABLE`, `DEJAVOO_TERMINAL_ISV_ID_INVALID`, `SPIN_CREDENTIALS_REJECTED`, `SPIN_UNREACHABLE`).
- While a terminal payment of the plan reads `processing`, every new payment on the plan answers `409` `PAYMENT_PLAN_PAYMENT_PENDING_CONFIRMATION`, and `POST /payment-plans/{id}/cancel` answers `409` `PAYMENT_PLAN_PAYMENT_PENDING_CONFIRMATION` with the message `A terminal payment on this plan is unresolved. Resolve it with POST /payment-plans/{id}/payments/{payment_id}/resolve before canceling the plan.` A sale on a terminal that is already running another one answers `409` `DEJAVOO_TERMINAL_BUSY`.
- A plan accepts up to 99 terminal attempts, its deposit attempts included; the next one answers `409` `PAYMENT_PLAN_TERMINAL_ATTEMPTS_EXHAUSTED`.
- A request repeated with the same `Idempotency-Key` answers the stored `201` again and starts no second sale; read the payment with `GET /payment-plans/{id}/payments/{payment_id}` for its current state.
- The amount is applied to the plan as it stands when ChaChing records the sale. When an installment was paid another way, or its invoice was voided, before the payment was recorded, the amount goes to the next open installment or to the principal. An approved amount the plan can no longer absorb is not applied; ChaChing's team is alerted to refund it at the terminal. When the plan absorbs only part of an approved sale the payment reads `partially_applied`, and when it absorbs none of it, `not_attempted`.
- A terminal payment that stays `resolving` because the terminal never reports its outcome can be listed and force-closed: see [Find and Force-Close a Stuck Terminal Attempt](#find-and-force-close-a-stuck-terminal-attempt).

---

# Find and Force-Close a Stuck Terminal Attempt

Every sale ChaChing sends to one of your terminals, a deposit or a payment, is a terminal attempt with an id `ppda_…`. An attempt normally reaches a final state on its own or through the resolve endpoints. When the terminal never reports an outcome, the attempt stays `resolving`: its plan cannot be canceled, and the plan takes no new deposit or payment until the attempt is resolved. These two endpoints list the attempts of your account and close such an attempt as not collected.

| **Request** | **What it does** |
| --- | --- |
| `GET /dejavoo/terminal-attempts` | Lists the terminal attempts of the account, in every state, newest first. |
| `POST /dejavoo/terminal-attempts/{id}/force-close` | Closes one attempt whose outcome could not be confirmed, as not collected. |

Both endpoints accept only your account's secret API key (`sk_…`): a publishable key (`pk_…`) answers `403` `SECRET_API_KEY_REQUIRED`, and a key that matches no account answers `401` (a `pk_`-shaped key that matches no account answers `403` `SECRET_API_KEY_REQUIRED` on the list route and `401` on the force-close route), and a request that also sends an `Authorization` header answers `409` with no `error` field. They answer whether Payment Plans are enabled on your account or not.

## Before You Force-Close

A force-close records that nothing was collected, and it voids and refunds nothing. When the card was in fact charged and the terminal cannot report it, the money stays collected and ChaChing has no record of it: taking the payment again charges the customer twice. Before you force-close an attempt:

1. Call the resolve endpoint first: `POST /payment-plans/{id}/deposit` for a deposit, `POST /payment-plans/{id}/payments/{payment_id}/resolve` for a payment.
2. Force-close only when the resolve endpoint keeps answering `resolving` and your Dejavoo (iPOSpays) account shows no sale for the attempt's `reference_id`.

Once an attempt's `deadline` has passed, only a status read records a sale that went through: the resolve endpoints make one, and FEED notifications do not record it. ChaChing protects a force-close in two ways:

- It refuses a force-close until 15 minutes after the attempt's `deadline`, which gives a terminal that was briefly offline time to come back.
- After that, the force-close first reads the sale's status from the terminal, exactly as the resolve endpoints do. When the read finds an outcome (approved, approved but the card not saved yet (`approved_no_credential`), partially approved, declined, cancelled or not charged), ChaChing records it as the resolve endpoints record it, closes nothing and answers `409` `PAYMENT_PLAN_TERMINAL_ATTEMPT_RESOLVED`; read the attempt again to see its state. An attempt that reads `approved_no_credential` needs you to act: call its resolve endpoint again, as the state tables above say.
- ChaChing closes the attempt only when the read leaves it unresolved: the terminal cannot be reached, does not answer in time or gives no final answer, or it reports no such sale and the iPOSpays check of that sale cannot confirm it.

The call can take up to about a minute because it reads the terminal. If your connection drops before the answer, read the attempt with `GET /dejavoo/terminal-attempts`: a repeated force-close answers `409` `PAYMENT_PLAN_TERMINAL_ATTEMPT_FINAL` when the first one closed it.

When SPIn reports an authentication failure (for example after the terminal's Auth Key changed), the read cannot see the sale. Update the Auth Key with `PATCH /dejavoo/terminals/{id}` and call the resolve endpoint before you force-close.

## List the Attempts

`GET /dejavoo/terminal-attempts` answers a page `{ "data": [...], "meta": { "page", "take", "items_total", "pages_total" } }`, newest first by `started`, with the page parameters `page` and `take` (`take` at most `50`). Filter with `search`, a URL-encoded JSON object with any of these keys:

| **Key** | **Value** |
| --- | --- |
| `terminal` | The `id` of one of your terminals. |
| `state` | One state or a comma-separated list, for example `"resolving,approved_no_credential"`. `resolving` also returns an attempt still `in_flight` whose `deadline` has passed. |
| `kind` | `deposit`, `payment`, or both separated by a comma. |
| `created` | The UTC days on which the attempt started, for example `{"type": "between", "from": "2026-11-01", "to": "2026-11-04"}`. The types are `equal`, `between`, `after`, `before` and `inTheLast`, as on the other lists. |

A `search` value that is not a JSON object, or a key with a value outside these, answers `400`.

The `dejavoo_terminal_attempt` object of a deposit whose outcome is not known:

```json
{
  "id": "ppda_4f9c2e1b7a6d3c8e5f0a1b2c",
  "object": "dejavoo_terminal_attempt",
  "kind": "deposit",
  "payment_plan": "pp_7d2c9a4e1b6f3c8a5e0d2b4f",
  "payment": null,
  "terminal": "3f6b2a4e-9c1d-4e8f-a2b7-5d0c9e1f4a63",
  "reference_id": "K7Q2M9XA01",
  "state": "resolving",
  "amount": 20000,
  "started": 1760400000,
  "deadline": 1760400240,
  "resolved": null,
  "live": false,
  "closable": true,
  "closable_at": 1760401140,
  "force_closed_at": null,
  "force_closed_by": null,
  "force_close_reason": null
}
```

| **Field** | **Meaning** |
| --- | --- |
| `id` | The attempt id, `ppda_…`: the `id` of `deposit.attempt` on the plan, and of `terminal` on a payment. |
| `kind` | `deposit` for a plan's deposit, `payment` for a payment taken on a terminal. |
| `payment_plan` | The plan's `id`, `pp_…`. |
| `payment` | The payment's `id`, `ppp_…`, for a payment; `null` for a deposit. |
| `terminal` | The `id` of the terminal the sale ran on. |
| `reference_id` | The reference and invoice number the terminal and your Dejavoo (iPOSpays) reports show for this sale. |
| `state` | One of the states of [Deposit Attempt States](#deposit-attempt-states) for a deposit, and of the `terminal.state` table of [Take a Payment on a Terminal](#take-a-payment-on-a-terminal) for a payment. An attempt still `in_flight` after its `deadline` reads `resolving`. |
| `amount` | The amount sent to the terminal, in integer cents. |
| `started` | When the attempt started, Unix seconds. |
| `deadline` | When the sale can no longer be open at the terminal, Unix seconds. |
| `resolved` | When its outcome was read, Unix seconds; `null` while it is not known. |
| `live` | `true` while the sale can still be open at the terminal: `in_flight` before `deadline`. |
| `closable` | `true` when a force-close is accepted now: the attempt is not final, the terminal reported no approval for it, and 15 minutes have passed since `deadline`. The force-close still reads the sale's status first, and it can answer `409` `PAYMENT_PLAN_TERMINAL_ATTEMPT_LIVE` while another request on the same plan is running. |
| `closable_at` | The earliest time a force-close is accepted, Unix seconds: `deadline` plus 15 minutes. `null` when the attempt is final or the terminal reported an approval for it; such an attempt is never force-closed. |
| `force_closed_at` | When the attempt was force-closed, Unix seconds; `null` otherwise. |
| `force_closed_by` | `chaching_staff` when ChaChing staff closed it, `api` when your integration closed it through this endpoint; `null` otherwise. |
| `force_close_reason` | The reason given with the force-close; `null` otherwise. |

The object never carries the terminal's Auth Key, its TPN, card details or gateway ids.

## Force-Close an Attempt

Call `POST /dejavoo/terminal-attempts/{id}/force-close` with the attempt `id` and your reason:

```json
{
  "reason": "Terminal lost power; iPOSpays shows no sale for K7Q2M9XA01."
}
```

`reason` is 1 to 500 characters, not blank, with no control characters such as a line break. ChaChing stores it without its leading and trailing spaces and returns it as `force_close_reason`. When the status read leaves the attempt unresolved, the answer is `200` with the attempt in `force_closed`, `force_closed_by` set to `api` and `force_closed_at` set. A deposit's plan stays `awaiting_deposit`: you can start a new attempt or cancel the plan. A payment reads `not_attempted`. ChaChing sends no webhook for a force-close. The force-close answers whether Payment Plans are enabled on your account or not.

| **Status** | **Error** | **When** |
| --- | --- | --- |
| `400` | no code | The `id` is not a valid id, `reason` fails validation, or the body carries another field. |
| `404` | `PAYMENT_PLAN_TERMINAL_ATTEMPT_NOT_FOUND` | The `id` is not an attempt of your account. |
| `409` | `PAYMENT_PLAN_TERMINAL_ATTEMPT_RESOLVED` | The status read found an outcome. ChaChing recorded it exactly as the resolve endpoints do and closed nothing; `message` names the attempt's state. Read the attempt again. |
| `409` | `PAYMENT_PLAN_TERMINAL_ATTEMPT_LIVE` | Nothing was changed. The message names the cause: the sale is still open at the terminal (`in_flight` before `deadline`; the body carries `closable_at`, the earliest time a force-close is accepted, Unix seconds); the terminal reported an approval for the attempt (call the resolve endpoint of its plan to record it); another request is working on the plan (retry shortly); or ChaChing could not read the sale status (call the resolve endpoint, then retry). |
| `409` | `PAYMENT_PLAN_TERMINAL_ATTEMPT_TOO_RECENT` | Less than 15 minutes have passed since `deadline`. The body field `closable_at` is the earliest time a force-close is accepted, Unix seconds. Nothing was changed. |
| `409` | `PAYMENT_PLAN_TERMINAL_ATTEMPT_FINAL` | The attempt is already final, a force-closed attempt included. Nothing was changed. |
| `409` | `SPIN_AUTH_KEY_UNREADABLE` | The terminal's stored Auth Key cannot be read, so its status cannot be read. Re-enter it with `PATCH /dejavoo/terminals/{id}`; nothing was changed. |
| `409` | `PAYMENT_PLAN_NOT_AWAITING_DEPOSIT` | A deposit attempt whose plan is no longer `awaiting_deposit`: its status cannot be read through the deposit resolve endpoint and nothing was changed. Contact ChaChing to close it. |
