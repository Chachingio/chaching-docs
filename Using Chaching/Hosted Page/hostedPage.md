## Using Hosted Checkout

ChaChing provides **hosted checkout pages** that allow you to redirect customers from your website to a secure ChaChing payment page.

This is typically used when:

- A customer selects a subscription plan on your website
- You want to redirect them to checkout
- You do not want to build your own payment UI

---

## Hosted Page URL Format

To redirect a customer to a hosted checkout page, use the following URL structure:

```
https://{chaching-url}/checkout/{id}
```

Where:

- `{id}` must be replaced with either:
    - **price_id** (for subscription checkout)
    - **invoice_id** (for unpaid invoice checkout)

---

## When to Use price_id vs invoice_id

### Subscription Checkout (Using `price_id`)

Use this when:

- A customer selects a plan
- You want to create a new subscription
- No invoice exists yet

Example:

```
https://chaching-url/checkout/price_12345
```

In this case:

- The hosted page will create the subscription
- An invoice will be generated after payment

---

### Invoice Payment (Using `invoice_id`)

Use this when:

- An invoice already exists
- The invoice is unpaid
- You want the customer to pay it

Example:

```
https://chaching-url/checkout/invoice_67890
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
window.location.href = "https://chaching-url/checkout/price_12345";
```

No additional API call is required for basic hosted page usage.

---

## Important Notes

ID Must Be Replaced

The placeholder shown during onboarding:

```
https://test.chaching.io/checkout/{priceId | invoiceId}
```

must be replaced with a real:

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