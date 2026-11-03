# Company and Branding

The **Company and Branding** page allows workspace administrators to manage company identity and visual branding used across customer-facing pages and communications.
---
## Company Information

This section displays the current company name associated with the workspace.

| **Field** | **Description** |
| --- | --- |
| **Company Name** | The legal or display name of your company shown in the dashboard and customer-facing interfaces. |
---
## Account Time Zone

The account time zone is shown read-only on this page. ChaChing support changes it.

- Times in the dashboard, the date filters of the dashboard and reports use the account time zone.
- Payment plan dates are calendar days in the account time zone.
- Subscription billing dates and invoice dates stay UTC calendar days.
- A payment plan installment is scheduled on its due date at 10:00 UTC for a customer created since ChaChing introduced the fixed billing time, and at the time of day the customer was created for an older customer. In the account time zone that time can fall on the evening before the due date: a customer created at 02:30 UTC whose installment is due on November 3, 2026 is charged on November 2, 2026 at 9:30 p.m. EST.
- An installment is also charged at once, before its scheduled time, when another billing change on the same customer happens between 00:00 UTC and that time on its due date (for example a new subscription, including one the customer starts on a hosted checkout page, a pause, a resume, or a payment of a balance that had blocked billing), which in a US time zone can be as early as the evening before the due date.
---
## Branding

The **Branding** section controls the visual appearance of ChaChing pages that are visible to your customers, such as invoices and payment pages.

### Branding Elements

| **Element** | **Description** |
| --- | --- |
| **Logo** | The primary company logo displayed on invoices and hosted pages. |
| **Primary Brand Color** | The main color used for headers, buttons, and highlights. |
| **Accent Color** | A secondary color used for emphasis and supporting UI elements. |

### Actions

| **Action** | **Description** |
| --- | --- |
| **Edit branding** | Enables editing mode for logo and color settings. |
| **Save changes** | Saves and applies the updated branding configuration. |

### Editing Branding

When **Edit branding** is selected, you can:

- Upload or replace the company logo
- Change the primary brand color
- Adjust the accent color

Changes are applied immediately after clicking **Save changes** and will reflect across all customer-facing ChaChing pages.

### Important Notes

- Branding changes do not affect historical invoices.
- Uploaded logos should meet recommended size and format requirements for best display quality.
- Only users with appropriate permissions can modify branding settings.