# Invoices

Three endpoints act on an existing invoice: send it to the customer by email, get the link to its hosted page, and download its PDF. Each takes the invoice `id` (the `id` of the Invoice object) and requires the account **secret** API key (`sk_…`) in the `cc-api-key` header. A publishable key (`pk_…`) answers `403` with the error code `SECRET_API_KEY_REQUIRED`, and a request that also carries an `Authorization` header answers `409`. An `id` that is not one of your invoices answers `404`.

None of the three refuses an invoice because of its status: a draft or open invoice is presented as payable, a paid invoice as paid and a void invoice as cancelled.

For every request and response schema, see the [API Reference](./api.json).

---

# Send an Invoice by Email

`POST /invoices/{id}/send` takes no request body. ChaChing renders the invoice PDF and sends the invoice email:

- **To:** the email address of the invoice's customer. No other recipient can be chosen.
- **Cc:** the owner of your ChaChing account (its first Super Admin user). When the owner's address is the customer's address, the email is sent to it once.
- **Content:** the invoice summary, a link to download the PDF and, while the invoice is payable, a **Pay this invoice** button that opens the invoice's hosted page.

The endpoint answers `200` with the Invoice object and sends the `invoice.sent` webhook once. Every call sends a new email; the endpoint takes no `Idempotency-Key`.

When the invoice's customer has no email address, the endpoint answers `409` with `INVOICE_CUSTOMER_EMAIL_MISSING` and sends nothing.

---

# Get the Hosted Invoice Link

`GET /invoices/{id}/hosted-link` answers:

```json
{
  "object": "invoice_hosted_link",
  "invoice": "84c413b1-cda9-4637-a75d-352e36274bf2",
  "url": "https://your-business.example.com/checkout/84c413b1-cda9-4637-a75d-352e36274bf2"
}
```

`url` is the permanent link of the invoice's hosted page, the link the **Pay this invoice** button of the invoice email opens. It does not expire and is the same on every call. The page shows the invoice as payable, paid or cancelled according to its status.

---

# Download the Invoice PDF

`GET /invoices/{id}/pdf` answers `200` with the PDF (`Content-Type: application/pdf`) as an attachment named after the invoice number, for example `INVOICE-1234.pdf`. The PDF is generated on each request and is not stored. On an account that charges through iPOSpays (Dejavoo), an invoice whose card payments carried a surcharge shows the `Credit card surcharge` and `Total charged` rows (see [Card Surcharges](./card-surcharges.md)).

---

# Rate Limits

`POST /invoices/{id}/send` and `GET /invoices/{id}/pdf` each accept at most 60 requests per minute from one IP address; the next request in that minute answers `429`. Every endpoint also counts toward the general limit of 200 requests per minute per IP address.

---

# Error Codes

| HTTP | Code | When |
| --- | --- | --- |
| `400` | — | the `cc-api-key` header is missing or malformed |
| `401` | — | the key is unknown |
| `403` | `SECRET_API_KEY_REQUIRED` | a publishable key was sent |
| `404` | — | the invoice is not one of yours |
| `409` | — | the request also carries an `Authorization` header |
| `409` | `INVOICE_CUSTOMER_EMAIL_MISSING` | send: the invoice's customer has no email address |
| `429` | — | send and PDF: more than 60 requests in one minute from one IP address |

---

# Create an Invoice

`POST /invoices` takes a `collection_method`:

- `charge_automatically` charges the invoice at creation. `default_payment_method` is required: the `id` of one of the customer's saved payment methods. A missing value answers `400`, and an unknown one answers `400`. There is no fallback to the customer's default payment method.
- `send_invoice` creates an open invoice for the customer to pay. `due_date` is required, and `default_payment_method` is ignored.

To charge an open invoice later, use `PUT /invoices/{id}`. It answers an `invoice_payment` object with the amount in cents and its `transactions`.

The invoice's `collection_method` is the one you created it with.
