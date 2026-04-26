# Developer Settings

The **Developer Settings** page allows developers to manage API access credentials and configure event-driven integrations.
---
## API Keys

This section provides access to standard API keys used to authenticate requests to the ChaChing API.

### Keys

| **Field** | **Description** |
| --- | --- |
| **Published key** | Public API key used in client-side integrations (e.g., `pk_3NKx7UOLI3MKgGwLqAJmKYDMSyXaK...`). |
| **Secret key** | Private API key used for server-side requests. Displayed in a masked format (e.g., `sk_Ew2Ww****C9JwORcw...`). |

### Actions

| **Action** | **Description** |
| --- | --- |
| **Copy to clipboard** | Copies the selected API key (published or secret) to the clipboard. |

### Security Notes

- The **Secret key** must never be exposed in client-side code.
- If a secret key is compromised, it should be rotated immediately.
- Access to API keys should be restricted to authorized users only.
---
## Webhooks

Webhooks allow your system to receive real-time notifications when events occur in ChaChing, such as payments, invoice updates, and subscription changes.

From the Developer Settings page you can:

- Configure webhook destination endpoints
- View delivery logs with event details and statuses (`success` / `failed` / `retry`)

For full webhook setup and API reference, see the [Webhook documentation](../Developer%20Guide/webhook.md).
