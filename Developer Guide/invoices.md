# Invoices

Three endpoints act on an existing invoice: send it to the customer by email, get the link to its hosted page, and download its PDF. Each takes the invoice `id` (the `id` of the Invoice object) and requires the account **secret** API key (`sk_…`) in the `cc-api-key` header. A publishable key (`pk_…`) answers `403` with the error code `SECRET_API_KEY_REQUIRED`. Send no `Authorization` header (see [API Basics](./api-basics.md#authentication)). An `id` that is not one of your invoices answers `404`.

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
  "invoice": "a80baa67-1341-4184-b712-b6f9b85b5a2b",
  "url": "https://rosen-dental.stage.chaching.io/checkout/a80baa67-1341-4184-b712-b6f9b85b5a2b"
}
```

`url` is `https://<account slug>.stage.chaching.io/checkout/<invoice id>` on staging and `https://<account slug>.chaching.io/checkout/<invoice id>` in production; an account without a slug gets the dashboard host (`https://stage.chaching.io/checkout/<invoice id>` on staging). The invoice id is a UUID. The link is the permanent link of the invoice's hosted page, the link the **Pay this invoice** button of the invoice email opens. It does not expire and is the same on every call. The page shows the invoice as payable, paid or cancelled according to its status.

---

# Download the Invoice PDF

`GET /invoices/{id}/pdf` answers `200` with the PDF (`Content-Type: application/pdf`) as an attachment named after the invoice number, for example `INVOICE-1234.pdf`. The PDF is generated on each request and is not stored. On an account that charges through iPOSpays (Dejavoo), an invoice whose card payments carried a surcharge shows the `Credit card surcharge` and `Total charged` rows (see [Card Surcharges](./card-surcharges.md)).

---

# Rate Limits

`POST /invoices/{id}/send` and `GET /invoices/{id}/pdf` each accept at most 60 requests per minute from one IP address; the next request in that minute answers `429`. Every endpoint also counts toward the general limit of 200 requests per minute per IP address.

With a platform API key, the 60-per-minute budget of each of these two routes applies per key and `Chaching-Account`, and the `RateLimit-*` headers describe that budget. One platform key also has an aggregate of 120 requests per minute across both routes together and all its accounts; past it the API answers `429` with `Retry-After`. Platform-key requests count toward the platform quotas instead of the general per-IP limit; see [Rate limits](./api-basics.md#rate-limits).

---

# Error Codes

| HTTP | Code | When |
| --- | --- | --- |
| `400` | — | the `cc-api-key` header is missing or malformed |
| `400` | no code | `POST /invoices/{id}/send`: the email provider refused the recipient (`Failed to send email. Please verify recipient and sender configuration.`). |
| `401` | — | the key is unknown |
| `401` | `MALFORMED_AUTHORIZATION_HEADER` | The `Authorization` header is malformed. |
| `403` | `SECRET_API_KEY_REQUIRED` | a publishable key was sent |
| `404` | — | the invoice is not one of yours |
| `409` | — | the request also carries an `Authorization` header |
| `409` | `INVOICE_CUSTOMER_EMAIL_MISSING` | send: the invoice's customer has no email address |
| `429` | — | send and PDF: more than 60 requests in one minute from one IP address |
| `500` | no code | `POST /invoices/{id}/send` or `GET /invoices/{id}/pdf`: the email or the PDF could not be produced. |

The PDF file name keeps letters, digits, `_` and `-` of the invoice number.

---

# Create an Invoice

`POST /invoices` takes a `collection_method`:

- `charge_automatically` charges the invoice at creation. `default_payment_method` is required: the `id` of one of the customer's saved payment methods. A missing value answers `400`, and an unknown one answers `400`. There is no fallback to the customer's default payment method. Creating the invoice makes that payment method the customer's default, so later automatic charges of that customer use it.
- `send_invoice` creates an open invoice for the customer to pay. `due_date` is required and must be a Unix timestamp in the future; a past `due_date` answers `400`. `default_payment_method` is ignored.

To charge an open invoice later, use `PUT /invoices/{id}`. It answers an `invoice_payment` object with the amount in cents and its `transactions`. A `502` with `error: PAYMENT_OUTCOME_UNCONFIRMED` means the charge may have succeeded. Check the invoice before you try again, or you can charge the customer twice.

`PUT /invoices/{id}` outcomes: `402` the charge was declined; `409` `INVOICE_ALREADY_PAID`, `PAYMENT_NOT_ATTEMPTED` (the customer has no payment method, or the payment was not attempted), `PAYMENT_BLOCKED_PENDING_CONFIRMATION` (an earlier payment is still being confirmed), `PAYMENT_PLAN_INVOICE_NOT_PAYABLE` (a guarded payment plan invoice). `DELETE /invoices/{id}`: `409` `INVOICE_ALREADY_VOID`, `INVOICE_PAID_NOT_VOIDABLE`, `INVOICE_ALREADY_WRITTEN_OFF`, `INVOICE_CHANGE_IN_PROGRESS` (another change to the invoice, such as a payment, is in progress; retry), `PAYMENT_PLAN_INVOICE_NOT_PAYABLE`; `429` `INVOICE_WRITE_OFF_BUSY` (retry after `Retry-After` seconds).

The invoice's `collection_method` is the one you created it with. An invoice created with `send_invoice` before this value was recorded reads `charge_automatically`.

The Error Codes table above covers only the three routes that act on an existing invoice. The refusals of `POST /invoices` are the ones named in this section. It also answers `404 Customer not found` and `404 Price not found` for an unknown `customer` or `price` (`404 Customer not found` when both are unknown), a validation `400` with `message` as an array, and `500 Invoice could not be created`.

---

# Written-off Invoices

`DELETE /invoices/{id}` on an open invoice that bills a subscription or payment plan period writes the invoice off instead of voiding it, and the dashboard can write off any open invoice. A written-off invoice keeps `status: open` and its amounts (`amount_due`, `amount_remaining`), reads `collection_status: written_off`, and ChaChing no longer collects it automatically. ChaChing sends `invoice.written_off` once, with the Invoice object, when the write-off is applied; it sends no `invoice.deleted` for it.

A written-off invoice can still be paid through `PUT /invoices/{id}` or its hosted page. A successful payment removes the write-off: the invoice reads `paid` and `collection_status: null`. `collection_status` is always `null` on a paid or void invoice. A payment that is declined or not attempted leaves the invoice written off. When the outcome of a payment cannot be confirmed (`502 PAYMENT_OUTCOME_UNCONFIRMED`), the write-off is removed; once that payment has failed, write the invoice off again. When another payment attempt of the invoice is still pending, or the write-off cannot be put back, the invoice is left without its write-off: read the invoice, and write it off again only when it is still owed and no attempt is settling.

A webhook destination subscribed to `*` receives `invoice.written_off`; a destination with an explicit event list receives it only after `invoice.written_off` is added to the list.
