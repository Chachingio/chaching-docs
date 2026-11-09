# Quickstart: Your First Payment Plan

This quickstart creates a practice location, saves its Dejavoo credentials, registers a terminal, creates a patient as a customer, creates a payment plan with a down payment and collects that down payment on the terminal. It uses a **platform API key** (`psk_...`) and the `Chaching-Account` header, which is how a platform such as a practice-management integration works. A merchant that integrates its own account follows the same steps from step 5 with its secret key and without `Chaching-Account`; see [Account-level variant](#account-level-variant).

Every request goes to staging, `https://stage-api.chaching.io`. Amounts are in cents and are small on purpose: on staging the iPOSpays UAT host approves `100` ($1.00). See [Integrating against staging](./integrating-staging.md#test-cards-and-amounts).

## Before you start

- A staging platform key from Chaching, stored in `CHACHING_PLATFORM_KEY`. See [Get access](./integrating-staging.md#get-access).
- Your iPOSpays UAT e-commerce credentials: TPN, API key, secret key and e-commerce token.
- A Dejavoo terminal set up for SPIn Cloud, with its TPN and its 10-character SPIn Auth Key.
- A public HTTPS URL that receives webhooks. See [Receive webhooks on staging](./integrating-staging.md#receive-webhooks-on-staging).

```bash
export CHACHING_PLATFORM_KEY="psk_..."
```

## 1. Register a webhook endpoint

Register one endpoint for your platform before you create accounts: it receives the events of every account of your platform, including accounts created later.

```bash
curl -X POST https://stage-api.chaching.io/platform/webhook-endpoints \
  -H "cc-api-key: $CHACHING_PLATFORM_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "endpoint": "https://example.com/chaching/webhooks",
    "enabled_events": ["payment_plan.created", "payment_plan.activated", "payment_plan.installment_paid", "payment_plan.installment_failed", "payment_plan.payment_succeeded", "payment_plan.completed", "payment_plan.canceled"]
  }'
```

<!-- captured: POST /platform/webhook-endpoints on stage, redacted -->
```json
CAPTURE WebhookEndpointWithSecretPublicDto
```

Returns the webhook endpoint with its signing secret in `secret`. The secret is shown only in this response and in a rotation: store it now. Verify every delivery with it; see [Webhooks](./webhook.md).

| Status | Error code | When | What to do |
| --- | --- | --- | --- |
| `400` | `WEBHOOK_EVENT_TYPE_UNKNOWN` | An event type is not in the catalog. | Use the names of [Webhooks](./webhook.md). |
| `400` | no code | `endpoint` is not a public `http` or `https` URL. | Use a public URL; see [Receive webhooks on staging](./integrating-staging.md#receive-webhooks-on-staging). |
| `409` | `WEBHOOK_ENDPOINTS_LIMIT_REACHED` | Your platform already has 16 endpoints. | Delete one first. |

## 2. Create an account for the location

```bash
curl -X POST https://stage-api.chaching.io/platform/accounts \
  -H "cc-api-key: $CHACHING_PLATFORM_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: create-account-office-1042-location-7" \
  -d '{
    "external_id": "location-7",
    "name": "Rosen Family Dental - Main Street",
    "company": {
      "external_id": "office-1042",
      "name": "Rosen Family Dental"
    },
    "time_zone": "America/New_York"
  }'
```

<!-- captured: POST /platform/accounts on stage, redacted -->
```json
CAPTURE PlatformAccountPublicDto
```

Returns the Account object with status `201`. Store its `id` (`acc_...`): every later request for this location sends it in `Chaching-Account`. Sending the same `external_id` again returns the same account with status `200`, unchanged, so a lost id is recovered by repeating this request.

```bash
export CHACHING_ACCOUNT_ID="acc_..."
```

| Status | Error code | When | What to do |
| --- | --- | --- | --- |
| `400` | `ACCOUNT_TIME_ZONE_INVALID` | `time_zone` is not an accepted IANA zone. | Use a US zone such as `America/New_York`. |
| `422` | `IDEMPOTENCY_KEY_REUSED` | The key was used with a different body. | Use a new key. |
| `500` | no code | The account could not be created; nothing was kept. | Retry with a new `Idempotency-Key`, or none: the same `external_id` is safe to send again. |

## 3. Save the location's Dejavoo credentials

```bash
curl -X PUT https://stage-api.chaching.io/dejavoo/credentials \
  -H "cc-api-key: $CHACHING_PLATFORM_KEY" \
  -H "Chaching-Account: $CHACHING_ACCOUNT_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "tpn": "123456789012",
    "api_key": "YOUR_IPOSPAYS_API_KEY",
    "secret_key": "YOUR_IPOSPAYS_SECRET_KEY",
    "ecom_token": "YOUR_IPOSPAYS_ECOM_TOKEN"
  }'
```

<!-- captured: PUT /dejavoo/credentials on stage, redacted -->
```json
CAPTURE DejavooCredentialsStatePublicDto
```

Returns which credential fields are set; it never returns a value. ChaChing checks the API key and secret key with iPOSpays before saving anything. The first save sets the account's payment gateway to Dejavoo.

| Status | Error code | When | What to do |
| --- | --- | --- | --- |
| `422` | `DEJAVOO_CREDENTIALS_REJECTED` | iPOSpays refused the API key and secret key. | Check them; on staging they are UAT credentials. |
| `503` | `DEJAVOO_VERIFICATION_UNAVAILABLE` | iPOSpays did not answer. | Nothing was saved; retry. |
| `429` | `PROCESSOR_CREDENTIALS_RATE_LIMITED` | More than 10 saves in a minute for this location. | Wait a minute. |

## 4. Register the terminal

```bash
curl -X POST https://stage-api.chaching.io/dejavoo/terminals \
  -H "cc-api-key: $CHACHING_PLATFORM_KEY" \
  -H "Chaching-Account: $CHACHING_ACCOUNT_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "tpn": "123456789013",
    "label": "Front desk",
    "spin_auth_key": "SPINAUTH10"
  }'
```

<!-- captured: GET /dejavoo/terminals on stage, redacted -->
```json
CAPTURE DejavooTerminalPublicDto @ /data/0
```

Returns the terminal with status `201`. Store its `id`: you name it when you collect a deposit. The SPIn Auth Key is never returned.

| Status | Error code | When | What to do |
| --- | --- | --- | --- |
| `409` | `DEJAVOO_NOT_CONFIGURED` | Step 3 was not done. | Save the credentials first. |
| `409` | `TERMINAL_TPN_EXISTS` | The TPN is already registered on this account. | List the terminals with `GET /dejavoo/terminals`. |
| `422` | `SPIN_CREDENTIALS_REJECTED` | SPIn refused the TPN and Auth Key pair. | Check both values. |
| `503` | `SPIN_UNREACHABLE` | SPIn did not answer. | Nothing was saved; retry. |

## 5. Create the customer

```bash
curl -X POST https://stage-api.chaching.io/customers \
  -H "cc-api-key: $CHACHING_PLATFORM_KEY" \
  -H "Chaching-Account: $CHACHING_ACCOUNT_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Jenny Rosen",
    "email": "jenny.rosen@example.com",
    "external_id": "patient-55821"
  }'
```

<!-- captured: POST /customers on stage, redacted -->
```json
CAPTURE CustomersPublicDto
```

Returns the Customer object with status `201`. `external_id` is your own id for the patient; find the customer later with `GET /customers?external_id=patient-55821`.

| Status | Error code | When | What to do |
| --- | --- | --- | --- |
| `409` | `PAYMENT_GATEWAY_NOT_CONFIGURED` | The account has no credentials yet. | Do step 3 first. |
| `409` | `CUSTOMER_EXTERNAL_ID_EXISTS` | An active customer already has this `external_id`. | Use the customer the message names. |
| `400` | no code | `name` is missing, or `email` is not an email. | Fix the body. |

## 6. Preview the schedule

```bash
curl -X POST https://stage-api.chaching.io/payment-plans/preview \
  -H "cc-api-key: $CHACHING_PLATFORM_KEY" \
  -H "Chaching-Account: $CHACHING_ACCOUNT_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "customer": "CUSTOMER_ID",
    "total_amount": 1200,
    "down_payment_amount": 100,
    "installment_count": 11,
    "interval": "monthly",
    "start_date": 1811808000
  }'
```

<!-- captured: POST /payment-plans/preview on stage, redacted -->
```json
CAPTURE PaymentPlanPreviewDto
```

Returns the schedule the plan would have, with nothing created. Replace `CUSTOMER_ID` with the id from step 5, and `start_date` (here 1 June 2027, 00:00 UTC) with a day after today.

## 7. Create the plan

Leave out `payment_method`: the plan waits for its down payment on the terminal.

```bash
curl -X POST https://stage-api.chaching.io/payment-plans \
  -H "cc-api-key: $CHACHING_PLATFORM_KEY" \
  -H "Chaching-Account: $CHACHING_ACCOUNT_ID" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: plan-patient-55821-treatment-1" \
  -d '{
    "customer": "CUSTOMER_ID",
    "external_id": "treatment-1",
    "description": "Orthodontic treatment",
    "total_amount": 1200,
    "down_payment_amount": 100,
    "installment_count": 11,
    "interval": "monthly",
    "start_date": 1811808000
  }'
```

<!-- captured: POST /payment-plans on stage, redacted -->
```json
CAPTURE PaymentPlanPublicDto
```

Returns the Payment Plan object with status `201` in status `awaiting_deposit`. Its `deposit` object names the `amount` the terminal collects. Nothing is charged until the deposit is collected. Store the plan's `id`:

```bash
export PLAN_ID="pp_..."
```

| Status | Error code | When | What to do |
| --- | --- | --- | --- |
| `400` | `PAYMENT_PLAN_START_DATE_INVALID` | `start_date` is not after today. | The message names the earliest valid date. |
| `409` | `PAYMENT_PLAN_EXTERNAL_ID_EXISTS` | A plan of this account already has this `external_id`. | Read that plan. |
| `409` | `PAYMENT_PLAN_CUSTOMER_DELINQUENT` | The customer has an invoice unpaid for one day or more, or is in the late-payment warning. | Collect the balance first. |

## 8. Collect the down payment on the terminal

```bash
curl -X POST https://stage-api.chaching.io/payment-plans/$PLAN_ID/deposit/terminal \
  -H "cc-api-key: $CHACHING_PLATFORM_KEY" \
  -H "Chaching-Account: $CHACHING_ACCOUNT_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "terminal_id": "TERMINAL_ID"
  }'
```

<!-- captured: POST /payment-plans/{id}/deposit/terminal on stage, redacted -->
```json
CAPTURE PaymentPlanPublicDto
```

Replace `TERMINAL_ID` with the terminal `id` from step 4. Returns the plan with status `202`; `deposit.attempt.state` is `in_flight` and the terminal shows the amount. The patient presents the card.

## 9. Wait for the answer

```bash
curl https://stage-api.chaching.io/payment-plans/$PLAN_ID \
  -H "cc-api-key: $CHACHING_PLATFORM_KEY" \
  -H "Chaching-Account: $CHACHING_ACCOUNT_ID"
```

<!-- captured: GET /payment-plans/{id} on stage, redacted -->
```json
CAPTURE PaymentPlanPublicDto
```

Poll every 2 to 3 seconds until `deposit.attempt.state` is no longer `in_flight`; it leaves that state at most 240 seconds after it started. `approved` saves the card and the plan becomes `active`. Every other state is described in [Deposit Attempt States](./payment-plans-terminal.md#deposit-attempt-states).

## 10. Read the delivery of an event

```bash
curl -G https://stage-api.chaching.io/webhook-deliveries \
  -H "cc-api-key: $CHACHING_PLATFORM_KEY" \
  -H "Chaching-Account: $CHACHING_ACCOUNT_ID" \
  --data-urlencode 'search={"event_type":"payment_plan.created"}'
```

<!-- captured: GET /webhook-deliveries on stage, redacted -->
```json
CAPTURE page WebhookDeliveryPublicDto
```

Returns one page of the account's deliveries of that event type. Store the `id` of the first one (`whd_...`) in `DELIVERY_ID` and read it:

```bash
curl https://stage-api.chaching.io/webhook-deliveries/$DELIVERY_ID \
  -H "cc-api-key: $CHACHING_PLATFORM_KEY" \
  -H "Chaching-Account: $CHACHING_ACCOUNT_ID"
```

<!-- captured: GET /webhook-deliveries/{id} on stage, redacted -->
```json
CAPTURE WebhookDeliveryDetailPublicDto
```

Returns the delivery with the payload sent and every attempt. Your endpoint received the same payload; its envelope `account.external_id` is `location-7` and `account.company.external_id` is `office-1042`.

## 11. List the installments

```bash
curl -G https://stage-api.chaching.io/payment-plans/$PLAN_ID/installments \
  -H "cc-api-key: $CHACHING_PLATFORM_KEY" \
  -H "Chaching-Account: $CHACHING_ACCOUNT_ID" \
  -d take=50
```

<!-- captured: GET /payment-plans/{id}/installments on stage, redacted -->
```json
CAPTURE page PaymentPlanInstallmentPublicDto
```

Returns one page of installments; installment `0` is the down payment. ChaChing charges the next installment on its `due_date` to the card the terminal saved.

## Account-level variant

A merchant that integrates its own account sends its secret key in `cc-api-key`, sends no `Chaching-Account` header, and starts at step 5 (its credentials and terminals are set up in the dashboard). Register its webhook endpoint with `POST /webhook-endpoints`. Every other request, body and response is the same.

```bash
curl https://stage-api.chaching.io/payment-plans/$PLAN_ID \
  -H "cc-api-key: $CHACHING_SECRET_KEY"
```
