# Developer Settings

The **Developer Settings** page allows developers to manage API access credentials and configure event-driven integrations.
---
## API Keys

This section lists the API keys of the account and lets developers create, rotate and revoke them. Only users with the **Developer** or **SuperAdmin** role see it.

### Key Types

| **Type** | **Description** |
| --- | --- |
| **Publishable** | Publishable key (`pk_...`), used only by the ChaChing hosted checkout pages. It does not work on the API endpoints. It is not secret, so the table shows it in full with a copy button. |
| **Secret** | Secret key (`sk_...`), required by every API endpoint and sent in the `cc-api-key` header. The table shows only its first characters and its last four. See [API Basics](../Developer%20Guide/api-basics.md). |

The publishable key (`pk_`) is used only by the ChaChing hosted checkout pages; every API operation outside `/platform` answers it `403` with `SECRET_API_KEY_REQUIRED`. `GET /balance` also requires the secret key. A key of a deactivated account answers `401 Invalid apiKey`.

### Keys Table

| **Column** | **Description** |
| --- | --- |
| **Name** | The name given when the key was created. The keys that existed before key management are named `legacy`; the publishable key a new account starts with is named `default`. |
| **Type** | Publishable or Secret. |
| **Key** | The full publishable key, or the first characters and the last four of a secret key. |
| **Status** | **Active**, **Expiring** (rotated; it works until its expiry), **Expired** or **Revoked**. |
| **Created**, **Last used**, **Expires** | When the key was created, when it last authenticated a request (updated at most once a minute) and when it stops working. |

### Actions

| **Action** | **Description** |
| --- | --- |
| **Create key** | Creates a key of the chosen type with a name. The new key is shown once: copy it before you close the dialog. |
| **Rotate** | Creates a replacement key with the same name and sets when the current key stops working: now, in 1 hour, in 24 hours (the default) or in 7 days. Both keys work until then. The new key is shown once. |
| **Revoke** | Stops the key from the next request. This cannot be undone. Revoking the last active publishable key stops the hosted checkout pages until another publishable key is active. |
| **Copy to clipboard** | Copies a publishable key from the table, or a new key from the dialog that shows it once. |

An account holds at most 10 active publishable keys and 10 active secret keys. The limit of 10 keys per type counts active and expiring keys. Rotating a key also needs a free slot (`409 API_KEY_LIMIT_REACHED`). A rotation keeps an earlier expiry that is already set. A revoked or expired key cannot be rotated (`API_KEY_NOT_ROTATABLE`).

### Security Notes

- A secret key must never be exposed in client-side code.
- ChaChing shows a secret key only once, when it is created or rotated. If you lose it, rotate the key.
- If a secret key is compromised, revoke it, or rotate it with the current key expiring **Now**.
- A new account starts with one publishable key and no secret key: create a secret key with **Create key** before your first API call, and copy it when it is shown.
---
## Webhooks

Webhooks allow your system to receive real-time notifications when events occur in ChaChing, such as payments, invoice updates, and subscription changes.

From the Developer Settings page you can:

- Configure webhook destination endpoints
- View delivery logs with event details and statuses (`pending` / `retry` / `success` / `failure`), and click a row to see its payload and every attempt

For full webhook setup and API reference, see the [Webhook documentation](../Developer%20Guide/webhook.md).

A platform's locations have no dashboard and no account keys; a platform uses its platform key. See [Platforms](../Developer%20Guide/platforms.md).
