# Processor Credentials

Processor credentials are the payment gateway keys of an account: Dejavoo (iPOSpays), NMI or Stripe. You set and rotate them through the API with the account's secret key, and you read back which fields are set. ChaChing never returns a credential value, not even the ones that are public by nature, such as a TPN or a Stripe publishable key.

## Set Up an Account in Order

1. **Set the processor credentials** with `PUT /dejavoo/credentials`, `PUT /nmi/credentials` or `PUT /stripe/credentials`. The first save sets the account's payment gateway.
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

Every request carries the account's secret key in the `cc-api-key` header. A publishable key answers `403 SECRET_API_KEY_REQUIRED`. A request that also carries an `Authorization` header is refused: `409`, or `401` when the header carries a signed-in token whose session has ended or that belongs to another account.

## Request Bodies

Every `PUT` sends the gateway's full set of required fields, every time. Optional fields can travel in the same request: leave a field out to keep its stored value.

| **Gateway** | **Required** | **Optional** |
| --- | --- | --- |
| Dejavoo | `tpn` (12 digits), `api_key`, `secret_key`, `ecom_token` | `feed_signing_key` (16 to 256 printable characters), `google_pay_merchant_id` (letters and digits, at most 64; `null` removes it) |
| NMI | `security_key`, `tokenization_key`, `checkout_public_key` | `webhook_signing_key` (16 to 256 printable characters) |
| Stripe | `publishable_key`, `secret_key` | none |

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

## Errors

| **Status** | **Error** | **When** |
| --- | --- | --- |
| `400` | no code | A field fails validation, or the body carries an unknown field. |
| `401` | no code | The key is unknown or belongs to a deactivated account, or the request carries an `Authorization` header with a signed-in token whose session has ended or that belongs to another account. |
| `403` | `SECRET_API_KEY_REQUIRED` | The key is a publishable key. |
| `409` | `PAYMENT_GATEWAY_LOCKED` | The account is set up on another gateway. |
| `409` | no code | The request carries any other `Authorization` header. |
| `422` | `DEJAVOO_CREDENTIALS_REJECTED` | iPOSpays rejected the API key and secret key. Nothing was saved. |
| `422` | `STRIPE_CREDENTIALS_REJECTED` | Stripe rejected the secret key, or the key cannot read the account balance. Nothing was saved. |
| `429` | `PROCESSOR_CREDENTIALS_RATE_LIMITED` | More than 10 `PUT` requests in one minute with the same key. Wait for the `Retry-After` seconds. |
| `500` | `KILL_BILL_PLUGIN_CONFIG_NOT_WRITTEN` | Nothing was saved. Send the request again. |
| `500` | `KILL_BILL_PLUGIN_CONFIG_UNCONFIRMED` | The credentials were saved but not confirmed in use. Send the same request again. |
| `503` | `DEJAVOO_VERIFICATION_UNAVAILABLE` | iPOSpays could not be reached. Nothing was saved. |
| `503` | `STRIPE_VERIFICATION_UNAVAILABLE` | Stripe could not be reached. Nothing was saved. |
