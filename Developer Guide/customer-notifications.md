# Send Your Own Customer Notifications

This page is for platforms that send their customers' payment plan receipts and notices themselves, using the webhooks and objects described in [Webhooks](./webhook.md) and [Payment Plans](./payment-plans.md). Each section names the event to listen to and the field that fills each line of the message.

---

## Before You Start

Each webhook names the merchant account it belongs to in `account.id`, and that account's company in `account.company.id`. Use them to pick the merchant's name, address and branding from your own records: the events do not carry them.

Deduplicate on the event `id`. A retried delivery carries the same `id`; `created` is in seconds and two events can share it, so never use `created` to order events or to detect a duplicate. Deliveries can arrive in any order.

The objects below are the same in webhooks and in REST responses: `GET /payment-plans/{id}/installments`, `GET /payment-plans/{id}/payments` and `GET /invoices/{id}` return the keys the events carry.

---

## Installment and Payment Receipts

Send a receipt for every installment and every payment ChaChing collects by card. Fill it from the Installment object (`payment_plan.installment_paid`) or the Payment object (`payment_plan.payment_succeeded`):

| Receipt line | Field |
| --- | --- |
| Merchant name and location | Your records for `account.id` |
| Description | `payment_plan_summary.description` |
| `Payment N of M` | `sequence` and `payment_plan_summary.installment_count`; `sequence` `0` is the down payment |
| Amount and currency | `amount_paid` and `payment_plan_summary.currency` |
| Credit card surcharge | `surcharge_amount`, printed when above `0` |
| Total charged | `amount_charged` |
| Date | `paid_at` |
| Card brand and last four digits | `card.brand`, `card.last4`; print `Card on file` when `card` is `null`; `card.brand` is the gateway's card network label, not guaranteed lowercase |
| Transaction type | The word `Purchase` |
| Remaining balance and payments remaining | `payment_plan_summary.remaining_balance`, `payment_plan_summary.installments_remaining` |
| Invoice link | `hosted_invoice_url` |

Send one receipt for each `payment_plan.payment_succeeded`: its `allocations` list every installment and principal amount the payment paid, each with its own `hosted_invoice_url`. Send one receipt for each `payment_plan.installment_paid` whose installment `id` is not listed in the `allocations` of a `payment_plan.payment_succeeded` of the same plan. ChaChing sends the `payment_plan.installment_paid` events of a payment before its `payment_plan.payment_succeeded`.

Deliveries can arrive out of order after retries, so an `installment_paid` can reach you before the `payment_succeeded` that lists it. Choose one rule for the whole integration:

- **One receipt per payment.** Send the receipt on `payment_plan.payment_succeeded`, and do not send a separate receipt for an `installment_paid` whose installment is listed in that payment's `allocations`. Before you treat an `installment_paid` as standalone, wait up to 70 minutes for a `payment_succeeded` of the same plan that lists it: ChaChing can hold a `payment_succeeded` for up to an hour, plus the delivery retry. Send the standalone receipt when none arrives.
- **One receipt per installment.** Send one receipt for each `payment_plan.installment_paid` and ignore `payment_plan.payment_succeeded` for receipts.

The card networks set what a receipt for an installment must carry. The receipt ChaChing sends meets these obligations, and a receipt you send in its place has to meet them too, as set out in the Visa Core Rules and Visa Product and Service Rules (Table 5-33 and Table 5-34 on receipt content, and the electronic receipt rules of section 5.9.1.2) and in Visa Business News AI09536:

- the installment sequence number, such as `Payment 2 of 4`;
- the merchant name and location;
- the description of the purchase;
- the amount with its currency;
- the total charged, including the credit card surcharge;
- the date;
- the card brand and the last four digits of the card;
- the transaction type;
- a direct link to the invoice.

---

## Failed Installment Notice

`payment_plan.installment_failed` is sent for each declined automatic attempt, retries included. Fill the notice from the Installment object:

| Notice line | Field |
| --- | --- |
| Installment number | `sequence` and `payment_plan_summary.installment_count` |
| Amount | `amount` |
| Description | `payment_plan_summary.description` |
| Decline reason | `failure_message` and `failure_code` |
| Next automatic retry | `next_payment_attempt` |
| Pay link | `hosted_invoice_url` |
| Card | `card` |
| First decline | `first_failed_at` |

Card network rules give the customer at least 7 calendar days from the first declined attempt on an installment to pay it with another payment method. `first_failed_at` is that first declined attempt, in Unix seconds; it does not change on later retries.

`first_failed_at` is the time ChaChing recorded the first confirmed decline. It is `null` for an installment whose only declines predate this field.

`failure_code` and `failure_message` are `null` when the most recent attempt was not declined.

---

## Late-Payment and Failed-Payment Outcome Notices

`payment_plan.past_due` and `payment_plan.defaulted` carry the Payment Plan object, which has no payment link. Read the plan's installments with `GET /payment-plans/{id}/installments` and use the `hosted_invoice_url` of the installment whose `status` is `failed`.

---

## Plan Paid in Full

`payment_plan.completed` carries the Payment Plan object with `status` `paid_off` and is sent after the `payment_plan.installment_paid` and `payment_plan.payment_succeeded` that completed the plan. Use it to send the paid-in-full notice, and the receipt events above for the last payment's receipt.

---

## When ChaChing Sends These Events

ChaChing sends `payment_plan.installment_paid` and `payment_plan.payment_succeeded` once the credit card surcharge of the charge is known, so `surcharge_amount` and `amount_charged` are final when the event arrives. When the surcharge cannot be read within an hour, ChaChing sends the event with `surcharge_amount` `0` and `surcharge_status` `unavailable`; the exact amount charged is then the one shown for that transaction in your Dejavoo (iPOSpays) account.

For a payment recorded with `POST /payment-plans/{id}/payments` that reaches the unbilled part of the plan, ChaChing sends `payment_plan.payment_succeeded` after it rebuilds the plan's remaining schedule, so `remaining_balance`, `installments_remaining` and `payment_plan_summary` match the response of that request.

Each event is sent once.

See [Webhooks](./webhook.md) for the envelope, the retry policy and the signature, and [Payment Plans](./payment-plans.md) for the plan lifecycle and the surcharge rules.
