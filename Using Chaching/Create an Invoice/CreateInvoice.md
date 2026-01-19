# Create an Invoice
Use this flow to issue an invoice to a customer and either request payment or automatically charge a saved payment method.
---
## Step 1: Open Create Invoice

1. In the top-right corner of the dashboard, click **Create**.
   {% img src="../Create a Payment/images/create.svg" alt="create.svg" withLightbox=true width="" height="" /%}
2. Select **Create an invoice**.
---
## Step 2: Select Customer

| Field | Description |
| --- | --- |
| **Customer** | Select an existing customer from the dropdown. The invoice will be issued to this customer. |
---
## Step 3: Choose Payment Method

You can choose how the invoice will be paid.

### Option A: Request Payment

| Field | Description |
| --- | --- |
| **Request payment** | Creates an invoice and sends it to the customer for manual payment. |
| **Due in** | Select when the invoice is due: Today, Tomorrow, 7, 14, 30, 45, 60, or 90 days. |

### Option B: Auto-charge Customer

| Field | Description |
| --- | --- |
| **Autocharge customer** | Automatically charges a saved payment method on file. |
| **Choose payment method** | Select a saved card or payment method from the dropdown. |
---
## Step 4: Add Pricing

Add one or more products to the invoice.

| Field | Description |
| --- | --- |
| **Product** | Search and select an existing product. |
| **Amount** | Price of the selected product. |
| **Tax** | Select an existing tax rate or click **Add tax**. |
| **Total** | Automatically calculated invoice total. |

Additional actions:

- **Add another product** – include multiple line items.
- **Add tax manually** – apply a custom tax amount.
---
## Step 5: Review Invoice

Click **Review Invoice** to preview how the invoice will appear to the customer. Applicable to Option A: Request Payment.

You can preview three views:

- Invoice Preview
- Email Preview
- Payment Page Preview
---
## Step 6: Create payment

Once reviewed:

- Click **Create payment**.
- The invoice is created and:
    - Sent to the customer (if **Request payment** was selected), or
    - Charged automatically (if **Autocharge customer** was selected).
---
## What Happens Next

- The invoice appears in **Invoices** with status:
    - **Draft**, **Open**, **Paid**, or **Void**.
- Payments and failures follow your configured retry and invoice rules.
- Customers can pay via the hosted payment page powered by **ChaChing**.