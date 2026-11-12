# Processor Credentials

Processor credentials are the payment gateway keys of an account: Dejavoo (iPOSpays), NMI or Stripe. You set and rotate them through the API with the account's secret key, and you read back which fields are set. ChaChing never returns a credential value, not even the ones that are public by nature, such as a TPN or a Stripe publishable key.

## Set Up an Account in Order

1. **Set the processor credentials** with `PUT /dejavoo/credentials`, `PUT /nmi/credentials` or `PUT /stripe/credentials`. The first save sets the account's payment gateway. On Dejavoo and NMI, include the signing key and give the gateway the account's webhook URL; see [The Dejavoo FEED Webhook URL](#the-dejavoo-feed-webhook-url) and [The NMI Webhook URL](#the-nmi-webhook-url).
2. **Register the terminals** (Dejavoo only) with `POST /dejavoo/terminals`. A Dejavoo account without an e-commerce TPN answers `409 DEJAVOO_NOT_CONFIGURED`; a successful `PUT /dejavoo/credentials` sets the TPN and clears that error.
3. **Take the first charge** once the save has answered `200`.

## The Endpoints

| **Request** | **What it does** |
| --- | --- |
| `GET /dejavoo/credentials` | Returns the Dejavoo credential state. |
| `PUT /dejavoo/credentials` | Sets or rotates the Dejavoo credentials. |
| `GET /nmi/credentials` | Returns the NMI credential state. |
| `PUT /nmi/credentials` | Sets or rotates the NMI credentials. |
| `GET /stripe/credentials` | Returns the Stripe credential state. |
| `PUT /stripe/credentials` | Sets or rotates the Stripe credentials. |

A `GET` of another gateway's credentials answers `200` with `active: false`, every `*_set` `false` and `updated_at: null`.

Every request carries the account's secret key in the `cc-api-key` header. A publishable key answers `403 SECRET_API_KEY_REQUIRED`. Send no `Authorization` header (see [API Basics](./api-basics.md#authentication)).

## Request Bodies

Every `PUT` sends the gateway's full set of required fields, every time. Optional fields can travel in the same request: leave a field out to keep its stored value.

| **Gateway** | **Required** | **Optional** |
| --- | --- | --- |
| Dejavoo | `tpn` (12 digits), `api_key`, `secret_key`, `ecom_token` | `feed_signing_key` (16 to 256 printable characters), `google_pay_merchant_id` (letters and digits, at most 64; `null` removes it) |
| NMI | `security_key`, `tokenization_key`, `checkout_public_key` | `webhook_signing_key` (16 to 256 printable characters) |
| Stripe | `publishable_key`, `secret_key` | none |

API keys, secret keys and security keys are trimmed and must be printable ASCII without a backslash.

The TPN is the location's e-commerce TPN, not a terminal TPN.

The `feed_signing_key` is optional. It verifies the iPOSpays FEED deliveries ChaChing receives for this account.

A Stripe publishable key starts with `pk_test_` or `pk_live_`. A Stripe secret key starts with `sk_test_`, `sk_live_`, `rk_test_` or `rk_live_`, and must be in the same mode as the publishable key: both test keys or both live keys.

## The State Object

`GET` and a successful `PUT` answer the same object. It tells you which fields are set, never their values.

```json
{
  "object": "dejavoo_credentials",
  "active": true,
  "tpn_set": true,
  "api_key_set": true,
  "secret_key_set": true,
  "secret_key_last4": "8888",
  "ecom_token_set": true,
  "feed_signing_key_set": false,
  "google_pay_merchant_id_set": false,
  "updated_at": 1762214400
}
```

| **Field** | **Meaning** |
| --- | --- |
| `object` | `dejavoo_credentials`, `nmi_credentials` or `stripe_credentials`. |
| `active` | `true` when this gateway is the account's payment gateway. A new account with no credentials reports Stripe as active until its first save. |
| `*_set` | `true` when the field holds a value. |
| `secret_key_last4`, `security_key_last4` | The last four characters of the secret key (Dejavoo, Stripe) or the security key (NMI); `null` before the first save. A Stripe account saved before this field existed reports `null` until its next save. |
| `updated_at` | Unix seconds of the last `PUT` (or dashboard save) of the required fields; `null` before the first save. |

The NMI object carries `security_key_set`, `security_key_last4`, `tokenization_key_set`, `checkout_public_key_set` and `webhook_signing_key_set`. The Stripe object carries `publishable_key_set`, `secret_key_set` and `secret_key_last4`.

## Credentials Are Checked Before They Are Saved

For Dejavoo, ChaChing asks iPOSpays to accept the API key and secret key. For Stripe, ChaChing reads the account balance with the secret key. A set the gateway rejects answers `422` and is not saved; when the gateway cannot be reached, the request answers `503` and nothing is saved, so send it again. The Dejavoo TPN and e-commerce token, the Stripe publishable key and the NMI keys are checked for format only.

A Stripe restricted key (`rk_`) needs read access to Balance; without it the request answers `422 STRIPE_CREDENTIALS_REJECTED`.

## The Payment Gateway Is Locked After the First Save

The first save sets the account's payment gateway. After that, only that gateway's credentials can be saved: a `PUT` for another gateway answers `409 PAYMENT_GATEWAY_LOCKED` and changes nothing. No request changes or removes the gateway, and no request deletes credentials; send a new set with `PUT` to rotate them.

## Saving Takes a Few Seconds

A `PUT` answers after the new credentials are in use. That usually takes a few seconds; it can take up to about a minute when the gateway check and the configuration step both run to their limits. When the answer is `500 KILL_BILL_PLUGIN_CONFIG_UNCONFIRMED`, the message starts with `Dejavoo configuration was saved, but` (or `NMI` or `Stripe`): the credentials were saved, but ChaChing could not confirm that they are in use. Send the same `PUT` again. When your client times out before the answer, read `GET` and compare `updated_at`.

## The Dejavoo FEED Webhook URL

iPOSpays FEED notifies ChaChing of every transaction of a Dejavoo account, sales taken on its terminals included. Each account has its own FEED webhook URL: `https://stage-api.chaching.io/dejavoo/webhook/{account_id}`.

`{account_id}` is the account's `id` (`acc_...`), returned by `POST /platform/accounts` and `GET /platform/accounts`. Replace `{account_id}` with the account's `id` (the `acc_...` value) when you give the URL to Dejavoo.

Turn FEED on for an account in this order:

1. Choose a signing key of 16 to 256 printable characters and save it as `feed_signing_key` with `PUT /dejavoo/credentials`, in the same request as the four required fields. Until the key is saved, ChaChing answers `401` to every FEED delivery for the account.
2. Ask Dejavoo, through your ISO, to configure merchant-level FEED for the merchant. Send the merchant's TPNs (the e-commerce TPN and every terminal TPN), the account's FEED webhook URL, HMAC as the authentication type and the signing key, and ask for the configuration to use the current signature version. Dejavoo's team makes this configuration; it is not a setting in the iPOSpays portal. One URL and one signing key cover every TPN of the merchant.
3. For each terminal TPN you register later, ask Dejavoo to enable FEED for that TPN.

Do not use the FEED setting under Settings in the iPOSpays portal for a ChaChing account. That setting belongs to the ISO account and sends the transactions of every merchant under the ISO to one URL, which ChaChing cannot match to an account. It also does not replace a merchant-level configuration, so every transaction would be delivered twice.

ChaChing checks each delivery's `signature` field (HMAC-SHA512 with the account's signing key) and answers:

| **Status** | **When** |
| --- | --- |
| `200` | ChaChing accepted the delivery. |
| `401` | The delivery is not a signed FEED notification, the URL names no active account, the account has no signing key, or the signature of a Transaction notification does not match the account's signing key. The body is empty. |
| Any other status | ChaChing could not process the delivery. Dejavoo retries a delivery that is not answered with a `2xx` status, up to three attempts. |

A Settlement notification whose signature does not match is answered `200` and not processed: ChaChing does not use Settlement notifications.

When a delivery is rejected, give ChaChing support the account id and the delivery time.

To change the signing key, save the new `feed_signing_key` and send it to Dejavoo the same day: until both sides hold the same key, every Transaction notification answers `401`.

## The NMI Webhook URL

NMI webhooks tell ChaChing the outcome of the account's NMI transactions. Each account has its own NMI webhook URL: `https://stage-api.chaching.io/nmi/webhook/{account_id}`.

`{account_id}` is the account's `id` (`acc_...`), as in the FEED webhook URL. Replace `{account_id}` with the account's `id` (the `acc_...` value) when you enter the URL in NMI.

Register it in this order:

1. In the NMI Merchant Portal, open Settings, then Webhooks, and copy the signing key the page shows. Save it as `webhook_signing_key` with `PUT /nmi/credentials`, in the same request as the three required fields. Until the key is saved, ChaChing answers `401` to every NMI webhook for the account.
2. On the same page, click Create, enter the account's NMI webhook URL and select the transaction events (`transaction.sale.success` and the other `transaction.*` events). NMI starts sending as soon as the webhook is saved.

ChaChing checks each delivery's `Webhook-Signature` header (HMAC-SHA256 with the signing key) and answers:

| **Status** | **When** |
| --- | --- |
| `200` | ChaChing accepted the delivery. |
| `401` | The `Webhook-Signature` header is missing, malformed or does not match the account's signing key, the account has no signing key, or the URL names no active account. The body is empty. |
| Any other status | ChaChing could not process the delivery. NMI retries a delivery that is not answered with `200`. |

When a delivery is rejected, give ChaChing support the account id and the delivery time.

## Errors

| **Status** | **Error** | **When** |
| --- | --- | --- |
| `400` | no code | A field fails validation, or the body carries an unknown field. |
| `401` | no code | The key is unknown or belongs to a deactivated account, or the request carries an `Authorization` header with a signed-in token whose session has ended or that belongs to another account. Send no `Authorization` header (see [API Basics](./api-basics.md#authentication)). |
| `403` | `SECRET_API_KEY_REQUIRED` | The key is a publishable key. |
| `404` | `ACCOUNT_NOT_FOUND` | With a platform key: `Chaching-Account` names no account of your platform. |
| `409` | `PAYMENT_GATEWAY_LOCKED` | The account is set up on another gateway. |
| `409` | no code | The request carries any other `Authorization` header. Send no `Authorization` header (see [API Basics](./api-basics.md#authentication)). |
| `422` | `DEJAVOO_CREDENTIALS_REJECTED` | iPOSpays rejected the API key and secret key. Nothing was saved. |
| `422` | `STRIPE_CREDENTIALS_REJECTED` | Stripe rejected the secret key, or the key cannot read the account balance. Nothing was saved. |
| `429` | `PROCESSOR_CREDENTIALS_RATE_LIMITED` | More than 10 `PUT` requests in one minute with the same key. Wait for the `Retry-After` seconds. |
| `500` | `KILL_BILL_PLUGIN_CONFIG_NOT_WRITTEN` | Nothing was saved. Send the request again. |
| `500` | `KILL_BILL_PLUGIN_CONFIG_UNCONFIRMED` | The credentials were saved but not confirmed in use. Send the same request again. |
| `503` | `DEJAVOO_VERIFICATION_UNAVAILABLE` | iPOSpays could not be reached. Nothing was saved. |
| `503` | `STRIPE_VERIFICATION_UNAVAILABLE` | Stripe could not be reached. Nothing was saved. |
