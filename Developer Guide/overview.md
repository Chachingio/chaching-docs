# Developer Overview

ChaChing is a billing API for recurring payments and payment plans. These pages describe the **staging** environment, where every feature lands first; production receives each release in a later promotion. Operations that exist only on staging carry a **Staging only** badge in the [API Reference](./api.json).

{% cards columns=2 %}
{% card title="Quickstart: your first payment plan" to="./quickstart-payment-plan.md" %}
Create an account, save its Dejavoo credentials, create a customer and a plan, and collect the deposit on a terminal, in eleven requests.
{% /card %}
{% card title="Integrating against staging" to="./integrating-staging.md" %}
Base URLs, how to get keys, test cards and amounts, idempotency, rate limits and how to receive webhooks on your laptop.
{% /card %}
{% card title="API Basics" to="./api-basics.md" %}
Authentication, errors, pagination, search filters, nullable fields, identifiers and every contract change.
{% /card %}
{% card title="Platforms" to="./platforms.md" %}
Operate many practice locations from one platform key: accounts, the Chaching-Account header and platform webhooks.
{% /card %}
{% card title="Payment Plans" to="./payment-plans.md" %}
Create plans, record payments and follow every status change.
{% /card %}
{% card title="Payment Plans on a Terminal" to="./payment-plans-terminal.md" %}
Register Dejavoo terminals, collect deposits and take payments at the counter.
{% /card %}
{% card title="Webhooks" to="./webhook.md" %}
Receive signed events, verify them and recover missed deliveries.
{% /card %}
{% card title="API Reference" to="./api.json" %}
Every operation, request and response schema, with a curl sample.
{% /card %}
{% /cards %}

## How the pages fit together

1. Read [Integrating against staging](./integrating-staging.md) to get a staging key.
2. Run the [Quickstart](./quickstart-payment-plan.md) end to end.
3. Read [API Basics](./api-basics.md) before you write error handling.
4. A partner that manages many locations reads [Platforms](./platforms.md); a merchant that integrates its own account uses its secret key and skips it.
