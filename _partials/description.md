# Overview

The Chaching API provides a unified way to manage subscriptions, invoices, and payments with branded checkout experiences.

It is designed for developers building billing flows similar to Stripe, but with lower transaction fees and full subscription management.

- **Base URL**: `https://api.chaching.io/v1/`
- **Format**: JSON over HTTPS
- **Versioning**: Current version: `v1`
---
# Authentication

Chaching API uses **Bearer tokens** for authentication.

Include your API key in the `Authorization` header with each request.

**Example:**

```
Authorization: Bearer sk_test_123456789
```

Token Lifecycle

- Tokens are project-specific.
- Expiration rules: TBD (fill in).
- Refresh/rotation mechanism: TBD.