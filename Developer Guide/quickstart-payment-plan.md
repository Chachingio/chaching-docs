# Quickstart: Your First Payment Plan

This quickstart creates a practice location, saves its Dejavoo credentials, registers a terminal, creates a patient as a customer, creates a payment plan with a down payment and collects that down payment on the terminal. It uses a **platform API key** (`psk_...`) and the `Chaching-Account` header, which is how a platform such as a practice-management integration works. A merchant that integrates its own account follows the same steps from step 5 with its secret key and without `Chaching-Account`; see [Account-level variant](#account-level-variant).

Every request goes to staging, `https://stage-api.chaching.io`. Amounts are in cents and are small on purpose: on staging the iPOSpays UAT host approves `100` ($1.00). See [Integrating against staging](./integrating-staging.md#test-cards-and-amounts).

## Before you start

- A staging platform key, created by your administrator in the platform console and stored in `CHACHING_PLATFORM_KEY`. See [Get access](./integrating-staging.md#get-access).
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
{
  "id": "4a19c6f9-c4c8-4b79-b9a1-379260670c3d",
  "object": "webhook_endpoint",
  "endpoint": "https://doctor-cycling-likewise-brother.trycloudflare.com/chaching/webhooks",
  "enabled_events": [
    "payment_plan.created",
    "payment_plan.activated",
    "payment_plan.installment_paid",
    "payment_plan.installment_failed",
    "payment_plan.completed",
    "payment_plan.canceled",
    "payment_plan.payment_succeeded"
  ],
  "status": "active",
  "signature_key": "endpoint_secret",
  "created": 1794269584,
  "updated": 1794269584,
  "secret": "<redacted len=49>"
}
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
{
  "object": "account",
  "id": "acc_ac3a743a00d0db71b2332f78",
  "external_id": "location-7",
  "name": "Rosen Family Dental",
  "company": {
    "id": "com_8d2abb56a643c585514c5aa1",
    "external_id": "office-1042",
    "name": "Rosen Family Dental"
  },
  "time_zone": "America/New_York",
  "status": "active",
  "deactivated_at": null,
  "created": 1794269584
}
```

Returns the Account object with status `201`. Store its `id` (`acc_...`): every later request for this location sends it in `Chaching-Account`. Sending the same `external_id` again returns the same account with status `200`, unchanged, so a lost id is recovered by repeating this request.

```bash
export CHACHING_ACCOUNT_ID="acc_..."
```

| Status | Error code | When | What to do |
| --- | --- | --- | --- |
| `400` | `ACCOUNT_TIME_ZONE_INVALID` | `time_zone` is not an accepted IANA zone. | Use a US zone such as `America/New_York`. |
| `422` | `IDEMPOTENCY_KEY_REUSED` | The key was used with a different body. | Use a new key. |
| `500` | no code | The outcome is unknown: the account may or may not exist. Repeat the request with the same external_id values (company and account) and a new Idempotency-Key: it returns the existing account or creates it. A billing-engine record created during a failed attempt can remain behind. | Repeat with the same `external_id` values and a NEW `Idempotency-Key`: the create is idempotent on those ids (`200` with the existing account, or `201`). |

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
{
  "object": "dejavoo_credentials",
  "active": true,
  "tpn_set": true,
  "api_key_set": true,
  "secret_key_set": true,
  "secret_key_last4": "t0He",
  "ecom_token_set": true,
  "feed_signing_key_set": false,
  "google_pay_merchant_id_set": false,
  "updated_at": 1794269586
}
```

Returns which credential fields are set; it never returns a value. ChaChing checks the API key and secret key with iPOSpays before saving anything. The first save sets the account's payment gateway to Dejavoo.

| Status | Error code | When | What to do |
| --- | --- | --- | --- |
| `422` | `DEJAVOO_CREDENTIALS_REJECTED` | iPOSpays refused the API key and secret key. | Check them; on staging they are UAT credentials. |
| `503` | `DEJAVOO_VERIFICATION_UNAVAILABLE` | iPOSpays did not answer. | Nothing was saved; retry. |
| `429` | `PROCESSOR_CREDENTIALS_RATE_LIMITED` | More than 10 saves in a minute for this location. | Wait a minute. |

To receive this location's FEED notifications, add `feed_signing_key` to this request and give Dejavoo the location's FEED webhook URL; see [The Dejavoo FEED Webhook URL](./processor-credentials.md#the-dejavoo-feed-webhook-url).

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
{
  "id": "393c386a-6c4c-41af-8645-8a1b2b262473",
  "object": "dejavoo_terminal",
  "tpn": "123456789013",
  "label": "UAT P1",
  "enabled": true,
  "register_id": null,
  "isv_id": null,
  "spin_auth_key_set": true,
  "credential_rotated_at": 1791760446,
  "last_probe_at": 1794009709,
  "last_probe_status": "Online",
  "build": null,
  "serial": "P1260428000529",
  "feed_enable_requested_at": 1791760425,
  "created": 1791760425
}
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
{
  "id": "cus_06dc7b45aabd042f600aec0c",
  "object": "customer",
  "name": "Jenny Rosen",
  "email": "jenny.rosen@example.com",
  "external_id": "patient-55821",
  "phone": null,
  "description": null,
  "created": 1794269591,
  "currency": "usd",
  "address": {
    "line1": null,
    "line2": null,
    "city": null,
    "state": null,
    "postal_code": null,
    "country": null
  },
  "shipping": {
    "name": "Jenny Rosen",
    "phone": "",
    "address": {
      "line1": null,
      "line2": null,
      "city": null,
      "state": null,
      "postal_code": null,
      "country": null
    }
  }
}
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
{
  "object": "payment_plan_preview",
  "mode": "terminal_deposit",
  "customer": "cus_06dc7b45aabd042f600aec0c",
  "total_amount": 1200,
  "down_payment_amount": 100,
  "financed_amount": 1100,
  "installment_amount": 100,
  "installment_count": 11,
  "interval": "monthly",
  "remainder_placement": "first",
  "start_date": 1811808000,
  "installments": [
    {
      "sequence": 1,
      "due_date": 1811808000,
      "amount": 100
    },
    {
      "sequence": 2,
      "due_date": 1814400000,
      "amount": 100
    },
    {
      "sequence": 3,
      "due_date": 1817078400,
      "amount": 100
    },
    {
      "sequence": 4,
      "due_date": 1819756800,
      "amount": 100
    },
    {
      "sequence": 5,
      "due_date": 1822348800,
      "amount": 100
    },
    {
      "sequence": 6,
      "due_date": 1825027200,
      "amount": 100
    },
    {
      "sequence": 7,
      "due_date": 1827619200,
      "amount": 100
    },
    {
      "sequence": 8,
      "due_date": 1830297600,
      "amount": 100
    },
    {
      "sequence": 9,
      "due_date": 1832976000,
      "amount": 100
    },
    {
      "sequence": 10,
      "due_date": 1835481600,
      "amount": 100
    },
    {
      "sequence": 11,
      "due_date": 1838160000,
      "amount": 100
    }
  ]
}
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
{
  "id": "pp_fb761610ab8be59e112221d7",
  "object": "payment_plan",
  "external_id": "treatment-1",
  "customer": "cus_06dc7b45aabd042f600aec0c",
  "customer_external_id": "patient-55821",
  "description": "Orthodontic treatment",
  "status": "awaiting_deposit",
  "currency": "USD",
  "total_amount": 1200,
  "down_payment_amount": 100,
  "installment_amount": 100,
  "installment_count": 11,
  "interval": "monthly",
  "start_date": 1811808000,
  "remainder_placement": "first",
  "remaining_balance": 1200,
  "installments_paid": 0,
  "installments_remaining": 12,
  "next_due_date": 1794182400,
  "payment_method": null,
  "gateway": null,
  "reference": "EZS0S6S8",
  "deposit": {
    "reference": "EZS0S6S8",
    "kind": "down_payment",
    "amount": 100,
    "status": "pending",
    "transaction_reference": null,
    "collected_at": null,
    "attempt": null
  },
  "created": 1794269592,
  "ended_at": null,
  "canceled_at": null
}
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
{
  "id": "pp_dfb674a07acfc1882b7e046e",
  "object": "payment_plan",
  "external_id": "treatment-1",
  "customer": "cus_848bff42aa106cf8815dcb4f",
  "customer_external_id": "patient-55821",
  "description": "Orthodontic treatment",
  "status": "awaiting_deposit",
  "currency": "USD",
  "total_amount": 1200,
  "down_payment_amount": 100,
  "installment_amount": 100,
  "installment_count": 11,
  "interval": "monthly",
  "start_date": 1811808000,
  "remainder_placement": "first",
  "remaining_balance": 1200,
  "installments_paid": 0,
  "installments_remaining": 12,
  "next_due_date": 1794182400,
  "payment_method": null,
  "gateway": null,
  "reference": "9DRT5BKW",
  "deposit": {
    "reference": "9DRT5BKW",
    "kind": "down_payment",
    "amount": 100,
    "status": "pending",
    "transaction_reference": null,
    "collected_at": null,
    "attempt": {
      "id": "ppda_72e785c65610da55a48bb496",
      "object": "payment_plan_deposit_attempt",
      "terminal": "393c386a-6c4c-41af-8645-8a1b2b262473",
      "reference_id": "9DRT5BKW02",
      "state": "in_flight",
      "started": 1794350505,
      "resolved": null,
      "decline_code": null,
      "decline_message": null,
      "surcharge_amount": 0,
      "amount_charged": 0
    }
  },
  "created": 1794269623,
  "ended_at": null,
  "canceled_at": null
}
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
{
  "id": "pp_dfb674a07acfc1882b7e046e",
  "object": "payment_plan",
  "external_id": "treatment-1",
  "customer": "cus_848bff42aa106cf8815dcb4f",
  "customer_external_id": "patient-55821",
  "description": "Orthodontic treatment",
  "status": "active",
  "currency": "USD",
  "total_amount": 1200,
  "down_payment_amount": 100,
  "installment_amount": 100,
  "installment_count": 11,
  "interval": "monthly",
  "start_date": 1811808000,
  "remainder_placement": "first",
  "remaining_balance": 1100,
  "installments_paid": 0,
  "installments_remaining": 11,
  "next_due_date": 1811808000,
  "payment_method": "35e7da7e-e108-4530-923f-cd8fe38ff107",
  "gateway": "dejavoo",
  "reference": "9DRT5BKW",
  "deposit": {
    "reference": "9DRT5BKW",
    "kind": "down_payment",
    "amount": 100,
    "status": "collected",
    "transaction_reference": "00006212345678901320261110174200",
    "collected_at": 1794350522,
    "attempt": {
      "id": "ppda_72e785c65610da55a48bb496",
      "object": "payment_plan_deposit_attempt",
      "terminal": "393c386a-6c4c-41af-8645-8a1b2b262473",
      "reference_id": "9DRT5BKW02",
      "state": "approved",
      "started": 1794350505,
      "resolved": 1794350522,
      "decline_code": null,
      "decline_message": null,
      "surcharge_amount": 3,
      "amount_charged": 103
    }
  },
  "created": 1794269623,
  "ended_at": null,
  "canceled_at": null
}
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
{
  "data": [
    {
      "id": "whd_8ed91230db0f6eca61fe73b5",
      "object": "webhook_delivery",
      "event": "evt_3b12bdf0ed369e47f07b3e8d",
      "event_type": "payment_plan.created",
      "webhook_endpoint": "4a19c6f9-c4c8-4b79-b9a1-379260670c3d",
      "endpoint": "https://doctor-cycling-likewise-brother.trycloudflare.com/chaching/webhooks",
      "status": "pending",
      "response_status_code": null,
      "created": 1794269592,
      "updated": 1794269592
    },
    {
      "id": "whd_ab34f5ced45354becd9a304e",
      "object": "webhook_delivery",
      "event": "evt_3b12bdf0ed369e47f07b3e8d",
      "event_type": "payment_plan.created",
      "webhook_endpoint": "535c6441-e0b3-49c2-b51a-04d771d87ea4",
      "endpoint": "https://doctor-cycling-likewise-brother.trycloudflare.com/chaching/webhooks",
      "status": "pending",
      "response_status_code": null,
      "created": 1794269592,
      "updated": 1794269592
    }
  ],
  "meta": {
    "page": 1,
    "take": 10,
    "items_total": 2,
    "pages_total": 1
  }
}
```

Returns one page of the account's deliveries of that event type. Store the `id` of the first one (`whd_...`) in `DELIVERY_ID` and read it:

```bash
curl https://stage-api.chaching.io/webhook-deliveries/$DELIVERY_ID \
  -H "cc-api-key: $CHACHING_PLATFORM_KEY" \
  -H "Chaching-Account: $CHACHING_ACCOUNT_ID"
```

<!-- captured: GET /webhook-deliveries/{id} on stage, redacted -->
```json
{
  "id": "whd_8ed91230db0f6eca61fe73b5",
  "object": "webhook_delivery",
  "event": "evt_3b12bdf0ed369e47f07b3e8d",
  "event_type": "payment_plan.created",
  "webhook_endpoint": "4a19c6f9-c4c8-4b79-b9a1-379260670c3d",
  "endpoint": "https://doctor-cycling-likewise-brother.trycloudflare.com/chaching/webhooks",
  "status": "success",
  "response_status_code": 200,
  "created": 1794269592,
  "updated": 1794269593,
  "payload": {
    "id": "evt_3b12bdf0ed369e47f07b3e8d",
    "account": {
      "id": "acc_ac3a743a00d0db71b2332f78",
      "external_id": "location-7",
      "company": {
        "id": "com_8d2abb56a643c585514c5aa1",
        "external_id": "office-1042"
      }
    },
    "created": 1794269592,
    "createdAt": "2026-11-10T00:13:12.918Z",
    "data": {
      "id": "pp_fb761610ab8be59e112221d7",
      "object": "payment_plan",
      "status": "awaiting_deposit",
      "created": 1794269592,
      "deposit": {
        "kind": "down_payment",
        "amount": 100,
        "status": "pending",
        "attempt": null,
        "reference": "EZS0S6S8",
        "collected_at": null,
        "transaction_reference": null
      },
      "gateway": null,
      "currency": "USD",
      "customer": "cus_06dc7b45aabd042f600aec0c",
      "ended_at": null,
      "interval": "monthly",
      "reference": "EZS0S6S8",
      "start_date": 1811808000,
      "canceled_at": null,
      "description": "Orthodontic treatment",
      "external_id": "treatment-1",
      "total_amount": 1200,
      "next_due_date": 1794182400,
      "payment_method": null,
      "installment_count": 11,
      "installments_paid": 0,
      "remaining_balance": 1200,
      "installment_amount": 100,
      "down_payment_amount": 100,
      "remainder_placement": "first",
      "customer_external_id": "patient-55821",
      "installments_remaining": 12
    },
    "event": "payment_plan.created"
  },
  "attempts": [
    {
      "attempt": 1,
      "created": 1794269593,
      "response_status_code": 200,
      "response_body": "{\"ok\":true}",
      "response_body_truncated": false,
      "error": null
    }
  ]
}
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
{
  "data": [
    {
      "id": "ppi_9cc4eaba578f5bfb64d1504c",
      "object": "payment_plan_installment",
      "payment_plan": "pp_dfb674a07acfc1882b7e046e",
      "payment_plan_external_id": "treatment-1",
      "customer": "cus_848bff42aa106cf8815dcb4f",
      "customer_external_id": "patient-55821",
      "sequence": 0,
      "due_date": 1794182400,
      "amount": 100,
      "amount_paid": 100,
      "surcharge_amount": 3,
      "amount_charged": 103,
      "surcharge_status": "final",
      "status": "paid",
      "attempt_count": 1,
      "next_payment_attempt": null,
      "paid_at": 1794350522,
      "first_failed_at": null,
      "failure_code": null,
      "failure_message": null,
      "card": {
        "brand": "visa",
        "last4": "3165"
      },
      "invoice": "9d23261f-698d-4fd9-8e4e-7c49064a0e12",
      "hosted_invoice_url": "https://stage.chaching.io/checkout/9d23261f-698d-4fd9-8e4e-7c49064a0e12",
      "payment_plan_summary": {
        "installment_count": 11,
        "currency": "USD",
        "description": "Orthodontic treatment",
        "remaining_balance": 1100,
        "installments_paid": 0,
        "installments_remaining": 11,
        "next_due_date": 1811808000,
        "status": "active"
      }
    },
    {
      "id": "ppi_bc5124a70b4056e9b719923f",
      "object": "payment_plan_installment",
      "payment_plan": "pp_dfb674a07acfc1882b7e046e",
      "payment_plan_external_id": "treatment-1",
      "customer": "cus_848bff42aa106cf8815dcb4f",
      "customer_external_id": "patient-55821",
      "sequence": 1,
      "due_date": 1811808000,
      "amount": 100,
      "amount_paid": 0,
      "surcharge_amount": 0,
      "amount_charged": 0,
      "surcharge_status": "final",
      "status": "scheduled",
      "attempt_count": 0,
      "next_payment_attempt": null,
      "paid_at": null,
      "first_failed_at": null,
      "failure_code": null,
      "failure_message": null,
      "card": null,
      "invoice": null,
      "hosted_invoice_url": null,
      "payment_plan_summary": {
        "installment_count": 11,
        "currency": "USD",
        "description": "Orthodontic treatment",
        "remaining_balance": 1100,
        "installments_paid": 0,
        "installments_remaining": 11,
        "next_due_date": 1811808000,
        "status": "active"
      }
    },
    {
      "id": "ppi_91bfa0a42c488cc5e5705c95",
      "object": "payment_plan_installment",
      "payment_plan": "pp_dfb674a07acfc1882b7e046e",
      "payment_plan_external_id": "treatment-1",
      "customer": "cus_848bff42aa106cf8815dcb4f",
      "customer_external_id": "patient-55821",
      "sequence": 2,
      "due_date": 1814400000,
      "amount": 100,
      "amount_paid": 0,
      "surcharge_amount": 0,
      "amount_charged": 0,
      "surcharge_status": "final",
      "status": "scheduled",
      "attempt_count": 0,
      "next_payment_attempt": null,
      "paid_at": null,
      "first_failed_at": null,
      "failure_code": null,
      "failure_message": null,
      "card": null,
      "invoice": null,
      "hosted_invoice_url": null,
      "payment_plan_summary": {
        "installment_count": 11,
        "currency": "USD",
        "description": "Orthodontic treatment",
        "remaining_balance": 1100,
        "installments_paid": 0,
        "installments_remaining": 11,
        "next_due_date": 1811808000,
        "status": "active"
      }
    },
    {
      "id": "ppi_6000c0ae6e9bfec2c0ffda24",
      "object": "payment_plan_installment",
      "payment_plan": "pp_dfb674a07acfc1882b7e046e",
      "payment_plan_external_id": "treatment-1",
      "customer": "cus_848bff42aa106cf8815dcb4f",
      "customer_external_id": "patient-55821",
      "sequence": 3,
      "due_date": 1817078400,
      "amount": 100,
      "amount_paid": 0,
      "surcharge_amount": 0,
      "amount_charged": 0,
      "surcharge_status": "final",
      "status": "scheduled",
      "attempt_count": 0,
      "next_payment_attempt": null,
      "paid_at": null,
      "first_failed_at": null,
      "failure_code": null,
      "failure_message": null,
      "card": null,
      "invoice": null,
      "hosted_invoice_url": null,
      "payment_plan_summary": {
        "installment_count": 11,
        "currency": "USD",
        "description": "Orthodontic treatment",
        "remaining_balance": 1100,
        "installments_paid": 0,
        "installments_remaining": 11,
        "next_due_date": 1811808000,
        "status": "active"
      }
    },
    {
      "id": "ppi_4e3dc46e021b52c8e1dc7596",
      "object": "payment_plan_installment",
      "payment_plan": "pp_dfb674a07acfc1882b7e046e",
      "payment_plan_external_id": "treatment-1",
      "customer": "cus_848bff42aa106cf8815dcb4f",
      "customer_external_id": "patient-55821",
      "sequence": 4,
      "due_date": 1819756800,
      "amount": 100,
      "amount_paid": 0,
      "surcharge_amount": 0,
      "amount_charged": 0,
      "surcharge_status": "final",
      "status": "scheduled",
      "attempt_count": 0,
      "next_payment_attempt": null,
      "paid_at": null,
      "first_failed_at": null,
      "failure_code": null,
      "failure_message": null,
      "card": null,
      "invoice": null,
      "hosted_invoice_url": null,
      "payment_plan_summary": {
        "installment_count": 11,
        "currency": "USD",
        "description": "Orthodontic treatment",
        "remaining_balance": 1100,
        "installments_paid": 0,
        "installments_remaining": 11,
        "next_due_date": 1811808000,
        "status": "active"
      }
    },
    {
      "id": "ppi_78d853845c0fdf4beecd8e1c",
      "object": "payment_plan_installment",
      "payment_plan": "pp_dfb674a07acfc1882b7e046e",
      "payment_plan_external_id": "treatment-1",
      "customer": "cus_848bff42aa106cf8815dcb4f",
      "customer_external_id": "patient-55821",
      "sequence": 5,
      "due_date": 1822348800,
      "amount": 100,
      "amount_paid": 0,
      "surcharge_amount": 0,
      "amount_charged": 0,
      "surcharge_status": "final",
      "status": "scheduled",
      "attempt_count": 0,
      "next_payment_attempt": null,
      "paid_at": null,
      "first_failed_at": null,
      "failure_code": null,
      "failure_message": null,
      "card": null,
      "invoice": null,
      "hosted_invoice_url": null,
      "payment_plan_summary": {
        "installment_count": 11,
        "currency": "USD",
        "description": "Orthodontic treatment",
        "remaining_balance": 1100,
        "installments_paid": 0,
        "installments_remaining": 11,
        "next_due_date": 1811808000,
        "status": "active"
      }
    },
    {
      "id": "ppi_947f2747fc18efbfecfbc30d",
      "object": "payment_plan_installment",
      "payment_plan": "pp_dfb674a07acfc1882b7e046e",
      "payment_plan_external_id": "treatment-1",
      "customer": "cus_848bff42aa106cf8815dcb4f",
      "customer_external_id": "patient-55821",
      "sequence": 6,
      "due_date": 1825027200,
      "amount": 100,
      "amount_paid": 0,
      "surcharge_amount": 0,
      "amount_charged": 0,
      "surcharge_status": "final",
      "status": "scheduled",
      "attempt_count": 0,
      "next_payment_attempt": null,
      "paid_at": null,
      "first_failed_at": null,
      "failure_code": null,
      "failure_message": null,
      "card": null,
      "invoice": null,
      "hosted_invoice_url": null,
      "payment_plan_summary": {
        "installment_count": 11,
        "currency": "USD",
        "description": "Orthodontic treatment",
        "remaining_balance": 1100,
        "installments_paid": 0,
        "installments_remaining": 11,
        "next_due_date": 1811808000,
        "status": "active"
      }
    },
    {
      "id": "ppi_104d73a72eb16a5f742d8c5f",
      "object": "payment_plan_installment",
      "payment_plan": "pp_dfb674a07acfc1882b7e046e",
      "payment_plan_external_id": "treatment-1",
      "customer": "cus_848bff42aa106cf8815dcb4f",
      "customer_external_id": "patient-55821",
      "sequence": 7,
      "due_date": 1827619200,
      "amount": 100,
      "amount_paid": 0,
      "surcharge_amount": 0,
      "amount_charged": 0,
      "surcharge_status": "final",
      "status": "scheduled",
      "attempt_count": 0,
      "next_payment_attempt": null,
      "paid_at": null,
      "first_failed_at": null,
      "failure_code": null,
      "failure_message": null,
      "card": null,
      "invoice": null,
      "hosted_invoice_url": null,
      "payment_plan_summary": {
        "installment_count": 11,
        "currency": "USD",
        "description": "Orthodontic treatment",
        "remaining_balance": 1100,
        "installments_paid": 0,
        "installments_remaining": 11,
        "next_due_date": 1811808000,
        "status": "active"
      }
    },
    {
      "id": "ppi_7027045468e7f7b00d0682b1",
      "object": "payment_plan_installment",
      "payment_plan": "pp_dfb674a07acfc1882b7e046e",
      "payment_plan_external_id": "treatment-1",
      "customer": "cus_848bff42aa106cf8815dcb4f",
      "customer_external_id": "patient-55821",
      "sequence": 8,
      "due_date": 1830297600,
      "amount": 100,
      "amount_paid": 0,
      "surcharge_amount": 0,
      "amount_charged": 0,
      "surcharge_status": "final",
      "status": "scheduled",
      "attempt_count": 0,
      "next_payment_attempt": null,
      "paid_at": null,
      "first_failed_at": null,
      "failure_code": null,
      "failure_message": null,
      "card": null,
      "invoice": null,
      "hosted_invoice_url": null,
      "payment_plan_summary": {
        "installment_count": 11,
        "currency": "USD",
        "description": "Orthodontic treatment",
        "remaining_balance": 1100,
        "installments_paid": 0,
        "installments_remaining": 11,
        "next_due_date": 1811808000,
        "status": "active"
      }
    },
    {
      "id": "ppi_f214d8b0894db50e0cd75215",
      "object": "payment_plan_installment",
      "payment_plan": "pp_dfb674a07acfc1882b7e046e",
      "payment_plan_external_id": "treatment-1",
      "customer": "cus_848bff42aa106cf8815dcb4f",
      "customer_external_id": "patient-55821",
      "sequence": 9,
      "due_date": 1832976000,
      "amount": 100,
      "amount_paid": 0,
      "surcharge_amount": 0,
      "amount_charged": 0,
      "surcharge_status": "final",
      "status": "scheduled",
      "attempt_count": 0,
      "next_payment_attempt": null,
      "paid_at": null,
      "first_failed_at": null,
      "failure_code": null,
      "failure_message": null,
      "card": null,
      "invoice": null,
      "hosted_invoice_url": null,
      "payment_plan_summary": {
        "installment_count": 11,
        "currency": "USD",
        "description": "Orthodontic treatment",
        "remaining_balance": 1100,
        "installments_paid": 0,
        "installments_remaining": 11,
        "next_due_date": 1811808000,
        "status": "active"
      }
    },
    {
      "id": "ppi_a6ddcc6ac65618f07e15f787",
      "object": "payment_plan_installment",
      "payment_plan": "pp_dfb674a07acfc1882b7e046e",
      "payment_plan_external_id": "treatment-1",
      "customer": "cus_848bff42aa106cf8815dcb4f",
      "customer_external_id": "patient-55821",
      "sequence": 10,
      "due_date": 1835481600,
      "amount": 100,
      "amount_paid": 0,
      "surcharge_amount": 0,
      "amount_charged": 0,
      "surcharge_status": "final",
      "status": "scheduled",
      "attempt_count": 0,
      "next_payment_attempt": null,
      "paid_at": null,
      "first_failed_at": null,
      "failure_code": null,
      "failure_message": null,
      "card": null,
      "invoice": null,
      "hosted_invoice_url": null,
      "payment_plan_summary": {
        "installment_count": 11,
        "currency": "USD",
        "description": "Orthodontic treatment",
        "remaining_balance": 1100,
        "installments_paid": 0,
        "installments_remaining": 11,
        "next_due_date": 1811808000,
        "status": "active"
      }
    },
    {
      "id": "ppi_dea5f41e89cd45f8154e0c70",
      "object": "payment_plan_installment",
      "payment_plan": "pp_dfb674a07acfc1882b7e046e",
      "payment_plan_external_id": "treatment-1",
      "customer": "cus_848bff42aa106cf8815dcb4f",
      "customer_external_id": "patient-55821",
      "sequence": 11,
      "due_date": 1838160000,
      "amount": 100,
      "amount_paid": 0,
      "surcharge_amount": 0,
      "amount_charged": 0,
      "surcharge_status": "final",
      "status": "scheduled",
      "attempt_count": 0,
      "next_payment_attempt": null,
      "paid_at": null,
      "first_failed_at": null,
      "failure_code": null,
      "failure_message": null,
      "card": null,
      "invoice": null,
      "hosted_invoice_url": null,
      "payment_plan_summary": {
        "installment_count": 11,
        "currency": "USD",
        "description": "Orthodontic treatment",
        "remaining_balance": 1100,
        "installments_paid": 0,
        "installments_remaining": 11,
        "next_due_date": 1811808000,
        "status": "active"
      }
    }
  ],
  "meta": {
    "page": 1,
    "take": 50,
    "items_total": 12,
    "pages_total": 1
  }
}
```

Returns one page of installments; installment `0` is the down payment. ChaChing charges the next installment on its `due_date` to the card the terminal saved.

## Account-level variant

A merchant that integrates its own account sends its secret key in `cc-api-key`, sends no `Chaching-Account` header, and starts at step 5 (its credentials and terminals are set up in the dashboard). Register its webhook endpoint with `POST /webhook-endpoints`. Every other request, body and response is the same.

```bash
curl https://stage-api.chaching.io/payment-plans/$PLAN_ID \
  -H "cc-api-key: $CHACHING_SECRET_KEY"
```
