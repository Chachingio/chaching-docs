## Using Hosted Checkout

ChaChing provides **hosted checkout pages** that allow you to redirect customers from your website to a secure ChaChing payment page.

This is typically used when:

- A customer selects a subscription plan on your website
- You want to redirect them to checkout
- You do not want to build your own payment UI

---

## Hosted Page URL Format

The hosted page address is `https://<your-slug>.stage.chaching.io/checkout/<id>` on staging and `https://<your-slug>.chaching.io/checkout/<id>` in production; the same path on the dashboard host redirects there when your account has a slug. For an invoice, read the exact link from `GET /invoices/{id}/hosted-link`.

Where:

- `{id}` must be replaced with either:
    - **price_id** (for subscription checkout)
    - **invoice_id** (for unpaid invoice checkout)

V23_OBSERVATION

---

## When to Use price_id vs invoice_id

### Subscription Checkout (Using `price_id`)

Use this when:

- A customer selects a plan
- You want to create a new subscription
- No invoice exists yet

Example:

```
https://rosen-dental.stage.chaching.io/checkout/price_1RzK8sN3vQ5mT7wY9xA2bC4d
```

A price checkout address is the account host plus `/checkout/` and the price id; no API returns it.

In this case:

- The hosted page will create the subscription
- An invoice will be generated after payment

---

### Invoice Payment (Using `invoice_id`)

Use this when:

- An invoice already exists
- The invoice is unpaid
- You want the customer to pay it

V23_OBSERVATION

Example:

```
https://rosen-dental.stage.chaching.io/checkout/a80baa67-1341-4184-b712-b6f9b85b5a2b
```

In this case:

- The hosted page loads the existing invoice
- The customer completes payment

---

## How to Redirect from Your Website

When a customer selects a plan:

1. Store the corresponding **price_id**
2. Redirect the user to:

```
https://{chaching-url}/checkout/{price_id}
```

Example (JavaScript):

```jsx
window.location.href = "https://rosen-dental.stage.chaching.io/checkout/price_1RzK8sN3vQ5mT7wY9xA2bC4d";
```

No additional API call is required for basic hosted page usage.

---

## What Customers See on the Hosted Page

The hosted checkout page displays the **workspace name** of the workspace selected in your account. The company name is not shown on hosted pages.

V23_OBSERVATION

---

## Applying Tax Rates

ChaChing supports applying a tax rate during checkout. Tax rates are not calculated automatically — you determine the appropriate rate externally and pass it when creating the checkout.

Append `?taxRate=` and the `txr_...` id of one tax rate: `https://<your-slug>.stage.chaching.io/checkout/<price id>?taxRate=txr_...`.

You can retrieve available tax rates for your account via the API.

---

## Important Notes

ID Must Be Replaced

The `<id>` of the hosted page address must be replaced with a real:

- `price_id`
- `invoice_id`

---

## Developer Reference

For full API details related to:

- Creating products
- Creating prices
- Creating invoices

Refer to:
- [**Developer Guide → API**](../../Developer%20Guide/api.json)