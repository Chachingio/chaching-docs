# Invoices & Subscriptions

The **Invoices & Subscriptions settings** page lets you choose how ChaChing retries declined automatic payments, the outcome applied to the subscriptions and invoices of customers with unpaid invoices, and the default content shown on invoice payment pages. To open it, go to **Settings** and select **Invoices & Subscriptions**.

For the day-by-day timeline and what each outcome does to your customers, see [Handle failed subscription payments](../Using%20Chaching/Subscriptions/failed-payments.md).

---
## Manage Failed Payments

This section defines the retry schedule for declined automatic payments and the outcome applied to customers with unpaid invoices. These settings apply to your whole account.

### Payment Retry Schedule

A retry row under **Manage failed payments** is one retry of a declined automatic payment, scheduled the selected number of days after the previous attempt.

| **Setting** | **Description** |
| --- | --- |
| **Retry** row | Schedules an automatic retry the selected number of days after the previous attempt. The options are **1 day after previous attempt**, **3 days after previous attempt**, **5 days after previous attempt**, **7 days after previous attempt**, and **10 days after previous attempt**. |
| Example below the rows | Shows how the selected intervals add up to the outcome day. |

**Default configuration:**

- Retry **1 day** after previous attempt
- Retry **3 days** after previous attempt
- Retry **5 days** after previous attempt
- Retry **7 days** after previous attempt

For a customer whose only unpaid invoice is a subscription renewal invoice charged automatically on its invoice date, the default configuration retries a charge that keeps being declined 1, 4, 9, and 16 days after the declined charge, and the outcome is expected to be applied around day 17. It can be applied later.

### Managing Retry Attempts

| **Action** | **Description** |
| --- | --- |
| **Add more** | Adds a retry row. Shown while the schedule has fewer than five rows. |
| **Minus (–) icon** | Removes the corresponding retry row. Shown while the schedule has more than one row. |

The schedule holds at least one and at most five retries. Retries are executed sequentially in the order shown.

---
## Outcome for Customers with Unpaid Invoices

Despite the labels below, the outcome is not triggered by the last retry. It depends on the age of the customer's unpaid invoices: the outcome day is the sum of the retry intervals plus one day, and an invoice's age is counted from its invoice date. For which invoices count and how the outcomes end, see [Handle failed subscription payments](../Using%20Chaching/Subscriptions/failed-payments.md). The outcome is applied to the customer, not only to the subscription whose payment was declined.

### After the last retry fails, the customer's subscriptions are

| **Option** | **Description** |
| --- | --- |
| **Cancelled** | When the outcome is applied, cancels every subscription of the customer that has not ended, immediately, and turns off automatic payment for that customer. A later payment does not restore the cancelled subscriptions. This is the default. |
| **Blocked as unpaid** | When the outcome is applied, puts every subscription of the customer that has not ended on hold: billing stops while on hold, and the subscriptions show **Unpaid**; a subscription can show Trial, Scheduled, Cancelled, or Expired instead. |
| **Kept past due** | When the outcome is applied, does not put the customer's subscriptions on hold, change their status, or stop their billing. |

### When subscriptions are cancelled, their unpaid invoices are

This field is shown only while **Cancelled** is selected, and the setting takes effect only for customers to whom the **Cancelled** outcome is applied.

| **Option** | **Description** |
| --- | --- |
| **Labelled uncollectible** | After the outcome cancels the customer's subscriptions, can label the customer's unpaid invoices as uncollectible; the dashboard shows the label **Uncollectible** next to the invoice status. The invoice status and the amount owed do not change. This is the default. |
| **Left open as past due** | Does not label the invoices uncollectible. |
---
## Invoice Payment Page Content

This section allows customization of default text shown on invoice payment pages.

| **Field** | **Description** |
| --- | --- |
| **Default memo** | A default message displayed at the top of invoices. |
| **Default footer** | A default footer message displayed at the bottom of invoices. |

These values can typically be overridden on individual invoices.
---
## Actions

| **Action** | **Description** |
| --- | --- |
| **Save changes** | Applies all updates to retry rules and invoice content settings. |
---
## Notes

- These settings apply to your whole account, and there are no per-customer settings.
- Saved settings also apply to customers who already have unpaid invoices. Shortening the retry schedule moves the outcome day earlier. Lengthening it moves the outcome day later and can release a customer from the Unpaid hold or from past due before the customer pays. Subscriptions that were already cancelled stay cancelled.
- Change the option in **After the last retry fails, the customer's subscriptions are** only when no customer has an outstanding invoice that is at least as old as your current outcome day. A customer who is already in the previous outcome when you switch is not released by later payments.
- While a customer is in an outcome, do not create or change that customer's subscriptions. Collect, write off, or void the customer's outstanding invoices first.
- An invoice created with **Request payment** does not count toward the outcome while no payment on it has been attempted. A declined payment on such an invoice can apply the outcome right away when the invoice is past its due date and at least as old as the outcome day. Follow up on invoices whose due date has passed, and collect or void them.
- The retry schedule applies to automatic payments that are declined. An automatic payment that fails because of a processing error, or whose result cannot be confirmed, can follow this schedule, be retried at other times, or not be retried.
- These settings can be changed from this page and through the ChaChing API. For the API, see [Configure Failed-Payment Settings](../Developer%20Guide/subscription-lifecycle.md) in the developer guide.