# Track subscription lifecycle and failed payments

This page lists the subscription status values the API returns and the webhook events ChaChing sends when automatic payments fail, when they are retried, and when a customer reaches a failed-payment outcome. For the business behavior of each outcome and the settings that control it, see [Handle failed subscription payments](../Using%20Chaching/Subscriptions/failed-payments.md).

---

# Subscription Status Values

The `status` field of a subscription returned by `GET /subscriptions` and `GET /subscriptions/{id}`, and of every subscription webhook payload, is one of these values:

| **Value** | **Meaning** |
| --- | --- |
| `active` | The subscription has started, and it is not on hold, cancelled, or expired. |
| `trial` | The subscription is in its trial period. |
| `scheduled` | The subscription starts on a future date. |
| `paused` | The subscription is on hold for a reason other than the **Blocked as unpaid** outcome, for example a pause requested through the API or the dashboard. |
| `unpaid` | The subscription is on hold because its customer is in the **Blocked as unpaid** outcome. |
| `cancelled` | The subscription is cancelled, or its cancellation is scheduled. A cancellation requested through the API or the dashboard reads `cancelled` at once and takes effect at the end of the period already billed; when nothing was billed yet, it takes effect immediately, or on the start date for a subscription that has not started. A later payment does not restore a subscription cancelled by the **Cancelled** outcome. |
| `expired` | The subscription reached its end date. |

- `paused` and `unpaid` are both holds, and a subscription on hold is not billed. A subscription on hold can read `trial`, `scheduled`, `cancelled`, or `expired` instead of `paused` or `unpaid`.
- Past due is not a status value. The **Kept past due** outcome does not change `status`.
- The status value `cancelled` is spelled with two `l`s. The webhook event name `subscription.canceled` is spelled with one.

---

# Automatic Charges and Retries

ChaChing sends `invoice.payment_failed` for every failed automatic charge, including each scheduled retry, and `invoice.payment_succeeded` when an automatic charge or a retry succeeds. An automatic charge is one ChaChing's billing engine starts on its own, such as a subscription renewal and its retries.

## Retry Timeline

- A declined automatic charge follows the account's retry schedule: while retries remain, the next retry is scheduled the configured number of days after the previous attempt.
- An automatic charge that fails for a reason other than a decline, such as a processing error at the payment gateway or a result that cannot be confirmed, can follow the retry schedule, be retried at other times, or not be retried, depending on the error.
- The outcome day is the sum of the retry intervals plus one day, and an invoice's age is counted from its invoice date. A subscription renewal invoice that is charged automatically counts toward the outcome from its invoice date. The outcome depends on invoice age, not on retries: a customer with an older unpaid renewal invoice can reach the outcome before the retries of a newer invoice end.
- An invoice created with `collection_method: "send_invoice"` does not count toward the outcome while no payment on it has been attempted, even after its `due_date` has passed; the invoice payload does not include that date, and it reports `collection_method: "charge_automatically"` for every invoice, so keep your own record of the invoices you create with `send_invoice`. After a payment on such an invoice fails, the invoice can count once its `due_date` has passed, with its age counted from its invoice date, so a failed payment on an invoice that is past its `due_date` and at least as old as the outcome day can apply the outcome right away.
- In the common case of a customer whose only unpaid invoice is a subscription renewal invoice, charged automatically on its invoice date and declined, the default schedule (1, 3, 5, 7) schedules retries on days 1, 4, 9, and 16, and the outcome is expected to be applied around day 17. It can be applied later.

## Invoice Payload Fields

- `attempt_count` is present on every invoice event. On `invoice.payment_succeeded` and `invoice.payment_failed` for an automatic charge, it is the number of attempts made on that charge. On every other invoice event, it is the number of payments recorded on the invoice.
- `next_payment_attempt` is present only on `invoice.payment_succeeded` and `invoice.payment_failed` for an automatic charge. Its value is the Unix timestamp (seconds) of the next scheduled retry, or `null` when no retry is scheduled.
- `next_payment_attempt` is absent from `invoice.payment_succeeded` and `invoice.payment_failed` for a payment started from the dashboard, the hosted payment page, or the API, from every other invoice event, and from invoice API responses.
- An `invoice.payment_failed` event whose `next_payment_attempt` is `null` means no retry is left for that charge. It does not mean an outcome was applied, because the outcome day is counted separately.

---

# Events for Failed-Payment Outcomes

The outcome is evaluated per customer, not per subscription.

Subscription events for an outcome are sent per subscription item, not per subscription. A subscription with two items produces two events of the same type with the same subscription `id`. Handle them idempotently by subscription `id`: processing the second event of the same type for the same subscription must leave the same result as processing the first.

| **Account outcome** | **Webhook events** | **Subscription `status`** |
| --- | --- | --- |
| **Cancelled** | One `subscription.canceled` for each subscription item the outcome cancels, with `reason: "dunning"` and `initiated_by: "system"`. No other event is sent for the outcome. | `cancelled` |
| **Blocked as unpaid** | One `subscription.paused` for each item of every subscription of the customer that has not ended, including a subscription whose cancellation is scheduled but not yet effective, with `reason: "dunning"` and `initiated_by: "system"`. | `unpaid`; a subscription can read `trial`, `scheduled`, `cancelled`, or `expired` instead |
| **Kept past due** | None. | Not changed by the outcome |

- The late-payment warning sends no webhook.
- `subscription.resumed` with `reason: "dunning"` and `initiated_by: "system"` can be sent when a hold from the **Blocked as unpaid** outcome ends, and only to subscription items that received `subscription.paused` for that hold. A hold can also end without any `subscription.resumed`, so do not rely on it to learn that a hold ended.
- A subscription paused through the API or the dashboard before the hold also receives `subscription.paused` for the hold. A subscription that is paused when the hold ends, whether it was paused before or during the hold, can receive `subscription.resumed`, but it stays paused and is not billed until it is resumed.
- After an `invoice.payment_succeeded` for a customer whose subscriptions are on hold, wait a short time, then re-read the subscription with `GET /subscriptions/{id}` to learn whether the hold ended. The hold is re-evaluated separately from the payment event, so a read made immediately on receipt can show the state from before the payment.
- A pause, resume, or cancellation requested through the API or the dashboard carries `reason: "requested"` and `initiated_by: "merchant"`.

---

# Subscription Changes During an Outcome

While a customer is in any of the three outcomes, do not create or change that customer's subscriptions with `POST /subscriptions` or `PATCH /subscriptions/{id}`. Such a request does not complete correctly and can disrupt the customer's existing subscriptions, including when it returns an error. Collect or void the customer's outstanding invoices first.

The API exposes no field that states whether a customer is in an outcome, and the failed-payment indicator the dashboard shows for a customer is not available through the API or in webhook payloads. Treat a customer as in an outcome while any of the customer's outstanding invoices is at least as old as the outcome day, counted from its invoice date.

---

# Emails Sent to Customers

ChaChing sends emails to the customer, with a copy to the account's first super admin user when the account has one, for failed automatic charges and retries and for applied outcomes. No email is sent for the late-payment warning, when the customer pays, or for a payment started from the dashboard, the hosted payment page, or the API. The subjects, the content, and when a customer receives no email are listed in [Handle failed subscription payments](../Using%20Chaching/Subscriptions/failed-payments.md).

---

# Detect Each State

| **State** | **How to detect it** |
| --- | --- |
| An automatic charge failed and a retry is scheduled | `invoice.payment_failed` with a non-null `next_payment_attempt` |
| No retry is left for a charge | `invoice.payment_failed` with `next_payment_attempt` set to `null`; this alone does not mean an outcome was applied |
| Subscriptions cancelled for non-payment | `subscription.canceled` with `reason: "dunning"`, per subscription item; `status` is `cancelled` |
| Subscriptions on hold for non-payment | `subscription.paused` with `reason: "dunning"`, per subscription item; `status` is `unpaid`, or can be `trial`, `scheduled`, or `cancelled` |
| Subscriptions past due | No event and no status value; the outcome does not change `status` |
| The customer paid | `invoice.payment_succeeded`; after a short delay, re-read the subscription to learn whether a hold ended. `subscription.resumed` with `reason: "dunning"` is not sent for every hold that ends; see **Events for Failed-Payment Outcomes** |
| Late-payment warning | Not exposed by the API or webhooks |
| Invoice labeled uncollectible | Not exposed by the API or webhooks |

---

# Configure Failed-Payment Settings

The retry schedule and the outcomes are configured in the dashboard, on the **Invoices & Subscriptions settings** page. A public API for these settings is in development. See [Invoices & Subscriptions](../settings/InvoicesSubscriptions.md).

For every field of the webhook payloads, see [Webhooks](./webhook.md).
