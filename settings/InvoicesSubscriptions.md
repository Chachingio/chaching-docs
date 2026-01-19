# Invoices & Subscriptions

The **Invoices & Subscriptions** settings page allows administrators to configure how failed payments are handled for subscriptions and invoices, as well as customize default content shown on invoice payment pages.
---
## Manage Failed Payments

This section defines automated retry behavior and actions taken when payments fail.

### Payment Retry Schedule

You can configure multiple retry attempts for failed payments. Each retry is scheduled relative to the previous attempt.

| **Setting** | **Description** |
| --- | --- |
| **Retry (N days after previous attempt)** | Schedules an automatic retry after the specified number of days. |

**Default configuration example:**

- Retry **1 day** after previous attempt
- Retry **3 days** after previous attempt
- Retry **5 days** after previous attempt
- Retry **7 days** after previous attempt
- Retry 10 **days** after previous attempt

### Managing Retry Attempts

| **Action** | **Description** |
| --- | --- |
| **Add more** | Adds an additional retry attempt to the schedule. |
| **Minus (–) icon** | Removes the corresponding retry attempt from the list. |

Retries are executed sequentially in the order shown.
---
## Actions After All Retries Fail

These settings determine what happens after all configured retry attempts have been exhausted.

### Subscription Payment Failure

If all retries for a **subscription payment** fail, one of the following actions can be selected:

| **Option** | **Description** |
| --- | --- |
| **Cancel the subscription** | Terminates the subscription immediately. |
| **Mark the subscription as unpaid** | Keeps the subscription active but flags it as unpaid. |
| **Leave the subscription past due** | Keeps the subscription in a past-due state until manual resolution. |

### Invoice Payment Failure

If all retries for an **invoice payment** fail, one of the following actions can be selected:

| **Option** | **Description** |
| --- | --- |
| **Mark the invoice as uncollectible** | Flags the invoice as uncollectible and stops further attempts. |
| **Leave the invoice past due** | Leaves the invoice open and past due for manual follow-up. |
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

- Retry schedules apply to both subscription and invoice payments unless otherwise specified.
- Changes affect future payment attempts only.
- Ensure retry timing aligns with your customer communication strategy.