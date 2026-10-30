# API Basics

## Environments

| **Environment** | **Base URL** |
| --- | --- |
| Staging | `https://stage-api.chaching.io` |
| Production | `https://api.chaching.io` |

Paths in this guide and in the [API Reference](/developer-guide/api) go directly after the base URL, with no version prefix: `GET https://stage-api.chaching.io/customers`. Requests and responses are JSON over HTTPS.

Staging and production are separate: an account, its keys, its customers and its data exist in one environment only.

## Authentication

Every request carries your account's secret API key in the `cc-api-key` header:

```
cc-api-key: sk_...
```

- Every endpoint of the [API Reference](/developer-guide/api) requires the **secret key** (`sk_...`). A publishable key (`pk_...`) answers `403` with `{"error":"SECRET_API_KEY_REQUIRED","message":"This route requires the account secret API key (sk_...)"}`.
- The publishable key is used only by the ChaChing hosted checkout pages. Never use it for server-side calls.
- Do not send an `Authorization` header with the API key. Some endpoints answer `409` when a request carries both.
- Keep the secret key on your server. Never put it in a browser, a mobile app or any page your customers open.

You find both keys in the dashboard under **Settings → Developer Settings**. See [Developer Settings](../settings/DeveloperSettings.md).

## Authentication Errors

| **Status** | **When** |
| --- | --- |
| `400` | The request has no `cc-api-key` header (`"message": "Missing apiKey parameter"`). |
| `401` | The key is unknown, or it belongs to a deactivated account (`Invalid apiKey`). |
| `403` | The key is a publishable key (`SECRET_API_KEY_REQUIRED`). |
