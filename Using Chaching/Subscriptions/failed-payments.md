# Handle failed subscription payments

When an automatic subscription payment is declined, ChaChing retries it on the schedule you set in **Invoices & Subscriptions** settings. ChaChing also tracks how old a customer's unpaid invoices are, and the outcome you chose depends on that age, whether or not a retry is still scheduled. This page explains the subscription statuses, how the dashboard marks customers with failed payments, the retry timeline, how the days are counted and which invoices count, what the outcomes do to your customer, how to write off or void an invoice, and how to change your settings safely.

For the API values and webhook events behind this behavior, see [Track subscription lifecycle and failed payments](../../Developer%20Guide/subscription-lifecycle.md). For a field-by-field description of the settings page, see [Invoices & Subscriptions](../../settings/InvoicesSubscriptions.md).

---

## Subscription statuses

The **Status** column of the subscriptions table and the status badge on the subscription details page show one of these statuses.

| **Status** | **Meaning** |
| --- | --- |
| **Active** | The subscription has started, and it is not on hold, cancelled, or expired. |
| **Scheduled** | The subscription starts on a future date. |
| **Trial** | The subscription is in its trial period. |
| **Paused** | The subscription is on hold for a reason other than the **Blocked as unpaid** outcome, for example because you paused it. |
| **Unpaid** | The subscription is on hold because its customer is in the **Blocked as unpaid** outcome. |
| **Cancelled** | The subscription is cancelled, or its cancellation is scheduled. A cancellation you request shows Cancelled at once and takes effect at the end of the period the customer was already billed for. When nothing was billed yet, it takes effect immediately, or on the start date for a subscription that has not started. |
| **Expired** | The subscription reached its end date. |

Paused and Unpaid are both holds, and a subscription on hold is not billed. A subscription on hold can show Trial, Scheduled, Cancelled, or Expired instead of Paused or Unpaid.

Past due is not a subscription status. The **Kept past due** outcome does not change the status of the customer's subscriptions.

The status filter tabs above the subscriptions table are **Active**, **Scheduled**, **Trial**, **Canceled**, **Paused**, **Expired**, **Unpaid**, and **All**.

---

## Failed-payment indicators in the dashboard

The dashboard can show one of these indicators for a customer:

| **Indicator** | **Meaning** |
| --- | --- |
| **Payment failed** | The customer has the late-payment warning described in **What happens when an automatic payment is declined**. |
| **Unpaid** | The customer is in the **Blocked as unpaid** outcome. |
| **Past due** | The customer is in the **Kept past due** outcome. |
| **Cancelled for non-payment** | The customer is in the **Cancelled** outcome. |

The indicator belongs to the customer, not to a single subscription:

- In a subscriptions table, including the one on the customer details page, the indicator shows next to the status of the customer's subscriptions.
- On the subscription details page, the indicator shows next to the status together with the date it started, for example **Payment failed since** followed by the date.
- On the customer details page, a banner shows the indicator with the date it started, an **Outstanding balance** amount, and a **View invoices** link.

The **Unpaid** indicator is not repeated next to a subscription whose status is **Unpaid**.

---

## What happens when an automatic payment is declined

The timeline below shows the most common case: a customer whose only unpaid invoice is a subscription renewal, charged automatically on its invoice date, with the default retry schedule (retries 1, 3, 5, and 7 days after the previous attempt).

| **Day** | **What happens** |
| --- | --- |
| **Day 0** | The renewal invoice is created and the automatic charge is declined. |
| **Day 1** | First retry. ChaChing also records a late-payment warning for the customer, and the dashboard shows the **Payment failed** indicator for that customer. The warning does not change the subscription status, and no email or webhook is sent for it. |
| **Day 4** | Second retry. |
| **Day 9** | Third retry. |
| **Day 16** | Fourth and last retry. |
| **Around day 17** | If the invoice is still unpaid, the outcome you chose is expected to be applied around this day. It can be applied later. |

When the customer has an email address, the declined charge and every failed retry send the customer a payment-failed email, and the email for the last failed retry states that no further automatic retries are scheduled. When a retry succeeds and the customer has no other unpaid invoice, no outcome is applied.

### How the days are counted

- Retries count from the declined charge. A retry is scheduled the configured number of days after the previous attempt, so with the default schedule, a charge that keeps being declined is retried 1, 4, 9, and 16 days after the declined charge.
- The outcome day is the sum of your retry intervals plus one day: day 17 with the default schedule, because 1 + 3 + 5 + 7 = 16. An invoice's age is counted in days from its invoice date.
- A subscription renewal invoice that is charged automatically counts toward the outcome from its invoice date. The customer reaches the outcome day when the oldest unpaid invoice that counts is as old as the outcome day, so the outcome can be applied before the retries of a newer invoice end. For example, with the default schedule, when a customer has two unpaid renewal invoices dated 10 days apart and the older one stays unpaid, the older invoice reaches the outcome day 7 days after the newer invoice's date, before the newer invoice's last scheduled retry day, its day 16.
- The outcome can also be applied to a customer who has no subscription. That customer can then receive the outcome email described in **Emails sent to your customer**.

### Invoices you create with Request payment

An invoice you create with **Request payment** does not count toward the outcome while no payment on it has been attempted, even after its due date, shown in the **Due** column of the invoices table, has passed.

After a payment on such an invoice fails, the invoice can count toward the outcome once its due date has passed, with its age counted from its invoice date. A declined payment on an invoice that is past its due date and at least as old as the outcome day can therefore apply the outcome to that customer right away. With the default settings, that outcome cancels every subscription of the customer that has not ended.

To protect customers you invoice for later payment, follow up on every invoice whose due date has passed, and collect it or void it. After a payment on such an invoice is declined, write off or void the invoice when you do not expect the customer to pay it soon: a voided invoice no longer counts toward the outcome, and neither does a written-off invoice while no later payment on it is attempted. See **Write off or void an invoice**.

### Declined payments and processing errors

The retry schedule applies to automatic payments that are declined. An automatic payment that fails for another reason, such as a processing error at the payment gateway or a result that cannot be confirmed, can follow your retry schedule, be retried at other times, or not be retried, depending on the error. The outcome day is counted the same way in every case.

---

## Choose the outcome

You choose the outcome in **After the last retry fails, the customer's subscriptions are**. Despite the label, the outcome is not triggered by the last retry: it depends on the age of the customer's unpaid invoices, as described in **How the days are counted**. The outcome is applied to the customer, not to a single subscription.

| | **Cancelled** | **Blocked as unpaid** | **Kept past due** |
| --- | --- | --- | --- |
| **Subscription status** | Cancelled | Unpaid; a subscription can show Trial, Scheduled, Cancelled, or Expired instead | Not changed by the outcome |
| **Indicator in the dashboard** | Cancelled for non-payment | Unpaid | Past due |
| **Billing** | Ends, because the subscriptions are cancelled | Stops while the subscriptions are on hold | Not stopped by the outcome |
| **Automatic payment for the customer** | Turned off | Not changed | Not changed |
| **Email to the customer** | "Your subscriptions have been cancelled" | "Your access is on hold pending payment" | "Your account is past due" |

While a customer is in any of the three outcomes, do not create or change that customer's subscriptions from the dashboard, the hosted payment page, or the API: such a request does not complete correctly and can disrupt the customer's existing subscriptions. Collect, write off, or void the customer's outstanding invoices first.

### Cancelled

When ChaChing applies this outcome, it cancels every subscription of the customer that has not ended, immediately. The unpaid balance stays owed: no invoice is voided or written off. ChaChing turns off automatic payment for the customer, so no further automatic charge is made to the customer's payment method. The customer can still pay the unpaid invoices through their invoice payment links, and the cancellation email includes a **Pay now** button for one of them.

A later payment never restores the cancelled subscriptions. After every outstanding invoice of the customer is paid in full, written off, or voided, ChaChing turns back on the automatic payment it turned off, for a customer who has a default payment method, unless you changed the outcome option while the customer was in this outcome (see **Change the settings**). To bill the customer again, create new subscriptions after every outstanding invoice of the customer is paid in full, written off, or voided.

### Blocked as unpaid

When ChaChing applies this outcome, every subscription of the customer that has not ended is put on hold. The subscriptions are not cancelled, and they are not billed while on hold. While on hold they show **Unpaid**, and a subscription can show Trial, Scheduled, Cancelled, or Expired instead, for example while it is still in its trial period.

After every outstanding invoice of the customer is paid in full, written off, or voided, the hold ends, unless you changed the outcome option while the customer was on hold (see **Change the settings**). The hold can also end before that, for example when the customer pays the oldest unpaid invoices. When the hold ends, billing resumes, except for a subscription that is paused at that time, including one paused during the hold: that subscription stays paused and is not billed until it is resumed.

### Kept past due

When ChaChing applies this outcome, it does not put the customer's subscriptions on hold, change their status, or stop their billing. After every outstanding invoice of the customer is paid in full, written off, or voided, the customer is no longer past due, unless you changed the outcome option while the customer was past due (see **Change the settings**).

---

## Choose what happens to the unpaid invoices

You choose the invoice outcome in **When subscriptions are cancelled, their unpaid invoices are**. The settings page shows this field only while **Cancelled** is selected, and the setting takes effect only for customers to whom the **Cancelled** outcome is applied. For customers in the **Blocked as unpaid** or **Kept past due** outcome, the invoice setting has no effect.

| **Option** | **What it does** |
| --- | --- |
| **Labelled uncollectible** | After the outcome cancels the customer's subscriptions, ChaChing can label the customer's unpaid invoices as uncollectible. Some unpaid invoices of the customer can stay unlabeled. |
| **Left open as past due** | ChaChing does not label the invoices uncollectible. |

The uncollectible label does not change the invoice status, does not reduce the amount owed, and does not stop the customer from paying the invoice. The label is not included in API responses or webhook payloads.

### Invoice labels in the dashboard

The invoices tables and the invoice details page can show one of these labels next to the invoice status:

| **Label** | **Meaning** |
| --- | --- |
| **Past due** | The invoice has an amount still owed, and its customer has one of the indicators described in **Failed-payment indicators in the dashboard**. |
| **Uncollectible** | ChaChing labeled the invoice uncollectible because of the **Labelled uncollectible** setting, and the invoice has an amount still owed. |
| **Written off** | You wrote off the invoice. |

An invoice shows at most one of these labels. **Written off** takes precedence over **Uncollectible**, and **Uncollectible** takes precedence over **Past due**.

---

## Write off or void an invoice

Writing off or voiding an invoice is how you forgive its balance.

To write off an **Open** invoice, open the invoice details page, open the menu next to the page actions, select **Write off**, and confirm with **Write off invoice**. The message "Invoice written off successfully!" confirms the write-off, and the invoice then shows the **Written off** label.

- A written-off invoice does not count toward the outcome while no later payment on it is attempted.
- A later payment attempt on a written-off invoice, for example through its payment link, can remove the write-off, and when that payment fails, the invoice can count toward the outcome.
- An invoice you create with **Request payment** cannot be written off while no payment on it has been attempted. Such an invoice does not count toward the outcome (see **Invoices you create with Request payment**).

In the dashboard, **Mark as void** is available only for draft invoices. You can void an open invoice through the API (`DELETE /invoices/{id}`).

---

## Emails sent to your customer

ChaChing sends these emails to the customer's email address, with a copy to the email address of your account's first super admin user. No email is sent when the customer has no email address.

| **When** | **Subject** | **Content** |
| --- | --- | --- |
| An automatic charge or a retry fails | "Payment failed for invoice" followed by the invoice number | The amount and invoice number, the date of the next retry or the sentence "No further automatic retries are scheduled.", and a **Pay this invoice** button. |
| Outcome: **Cancelled** | "Your subscriptions have been cancelled" | A **Pay now** button that links to one of the customer's unpaid invoices. |
| Outcome: **Blocked as unpaid** | "Your access is on hold pending payment" | A **Pay now** button that links to one of the customer's unpaid invoices. |
| Outcome: **Kept past due** | "Your account is past due" | The sentence "Your subscriptions are still active." and a **Pay now** button that links to one of the customer's unpaid invoices. |

No email is sent for the late-payment warning or when the customer pays. Charges you start from the dashboard, the hosted payment page, or the API do not send the payment-failed email.

---

## Default settings

A new ChaChing account starts with these settings:

| **Setting** | **Default** |
| --- | --- |
| Retry schedule under **Manage failed payments** | Retries 1, 3, 5, and 7 days after the previous attempt |
| **After the last retry fails, the customer's subscriptions are** | **Cancelled** |
| **When subscriptions are cancelled, their unpaid invoices are** | **Labelled uncollectible** |

With the default settings, for a customer whose only unpaid invoice is a subscription renewal invoice charged automatically on its invoice date, the **Cancelled** outcome is expected to be applied around day 17 after that invoice's date if the invoice stays unpaid, and it can be applied later. When that outcome is applied, it cancels every subscription of the customer that has not ended. For invoices you create with **Request payment**, see **Invoices you create with Request payment**.

---

## Change the settings

1. Go to **Settings** and select **Invoices & Subscriptions**. The **Invoices & Subscriptions settings** page opens.
2. Under **Manage failed payments**, set the retry schedule. Choose an interval for each row. Select **Add more** to add a row, and select the minus icon next to a row to remove it. Below the rows, an example shows how your intervals add up to the outcome day.
3. In **After the last retry fails, the customer's subscriptions are**, select **Cancelled**, **Blocked as unpaid**, or **Kept past due**.
4. When **Cancelled** is selected, the field **When subscriptions are cancelled, their unpaid invoices are** appears. Select **Labelled uncollectible** or **Left open as past due**.
5. Select **Save changes**. The message "Settings updated successfully" confirms that the settings are saved.

The retry schedule follows these rules:

- The interval options for a row are 1, 3, 5, 7, and 10 days after the previous attempt.
- The schedule holds at least one retry and at most five, and every row needs an interval.
- **Save changes** stays disabled until every field is valid.
- The longest schedule is five retries at 10-day intervals, which puts the outcome day on day 51.

### Before you save

The settings apply to your whole account, and there are no per-customer settings. Saved settings also apply to customers who already have unpaid invoices.

- **Changing the outcome option.** Change the option in **After the last retry fails, the customer's subscriptions are** only when no customer has an outstanding invoice that is at least as old as your current outcome day. If you lengthened the retry schedule earlier, check against the outcome day from before that change. A customer who is already in the previous outcome when you switch is not released by later payments and is not moved to the new outcome: a customer on hold stays on hold, a past-due customer stays past due, and a customer whose subscriptions were cancelled does not get automatic payment turned back on. If you already switched while customers were in an outcome, contact ChaChing support.
- **Shortening the retry schedule.** The outcome day moves earlier, and a customer with an unpaid invoice that is already at least as old as the new outcome day can reach the outcome soon after you save.
- **Lengthening the retry schedule.** The outcome day moves later, and a customer who is already on hold or past due can be released before paying. Subscriptions that were already cancelled stay cancelled.

These settings can be changed only from the dashboard. A public API for these settings is in development.
