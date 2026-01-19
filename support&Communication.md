# Support & Communication

This section explains how to get help, find answers, and communicate with the ChaChing team.
---
## Documentation

The ChaChing documentation is the primary source of guidance for both business users and developers.

Use the documentation to:

* Learn how to use the ChaChing dashboard
* Understand customer, product, subscription, and invoice workflows
* Explore API capabilities and integration details
* Review onboarding, migration, and billing behavior

Documentation is continuously updated as new features are released.

---
## Contact Support (Email)

If you need direct assistance or encounter an issue that cannot be resolved through documentation, you can contact the ChaChing support team via email.

**Support email:**
**[support@chaching.io](mailto:support@chaching.io)**

Use email support for:

* Account or access issues
* Migration or onboarding questions
* Payment, invoice, or subscription problems
* Unexpected system behavior or errors

**When contacting support, include:**

* Your account or tenant name
* Relevant customer, subscription, invoice, or payment IDs
* Screenshots or error messages (if applicable)
* A short description of what you were trying to do and what happened instead

This helps the support team investigate and respond faster.

---
## Frequently Asked Questions (FAQ)

### Getting Started & Onboarding

**What happens to my existing Stripe subscriptions after migration?**

After migration is confirmed, ChaChing takes over billing while preserving all historical Stripe data. Subscriptions continue billing on their normal renewal dates.

**Can I onboard without connecting a Stripe account?**

Yes. If you don’t have existing Stripe subscriptions, you can skip the migration step and start using ChaChing for new customers and subscriptions.

**How long does the migration process take?**

Migration time depends on the number of subscriptions and related data. During migration, ChaChing continues monitoring Stripe and imports any new subscriptions created.



### Customers

**Can I add customers manually?**

Yes. You can create customers directly from the Customers section and optionally add billing and shipping details.

**Can I update customer information after creation?**

Yes. Go to **Customers**, hover over a customer, select **View details**, and click **Edit customer**.

**Can I delete a customer?**

Yes. Customers can be deleted from the customer details page. Historical invoices and payments remain available for reporting and audit purposes.

**Can I export customer data?**

Yes. Customers can be exported in CSV or XLSX format with configurable date ranges and selected columns.



### Products & Pricing

**What pricing models are supported?**

Currently, ChaChing supports flat-rate pricing for recurring and one-off products. Additional pricing models will be introduced in future releases.

**Can I create multiple prices for a single product?**

Yes. Each product can have multiple pricing configurations with different billing periods.

**Can I include tax in the product price?**

Yes. You can choose whether tax is included or calculated separately when creating products or prices.



### Subscriptions

**What subscription statuses are available?**

Subscriptions can be active, trialing, scheduled, paused, cancelled, expired, or past due, depending on billing and payment state.

**Can I pause or cancel a subscription?**

Yes. From the subscription details page, you can pause or cancel an active subscription at any time.

**Can I change a customer’s subscription plan?**

Yes. You can modify subscription pricing or products depending on your current subscription configuration.



### Payments

**How are payments processed in ChaChing?**

Payments are processed using the customer’s default payment method or manually entered card details, depending on the workflow.

**What happens when a payment fails?**

Failed payments follow the retry rules configured in **Invoices & Subscriptions settings**. Depending on your configuration, subscriptions or invoices may be marked as past due, unpaid, or cancelled.

**Can I charge a customer manually?**

Yes. You can create one-time payments or charge open invoices directly from the dashboard.


### Invoices

**What invoice statuses are supported?**

Invoices can be draft, open, paid, void, or uncollectible, depending on payment and retry outcomes.

**Can I send invoices by email?**

Yes. Invoices can be emailed directly to customers, including a hosted payment page.

**Can customers pay invoices online?**

Yes. Customers can pay invoices using a secure hosted payment page.


### Transactions & Reporting

**Where can I view transaction history?**

All transactions are available in the **Transactions** section, with filters for status, customer, date, and amount.

**Can I export transactions and invoices?**

Yes. Both transactions and invoices can be exported in CSV or XLSX format.


### Users & Access Management

**Can I invite team members to my account?**

Yes. Owners and admins can invite users and assign roles with specific permissions.

**What roles are supported?**

ChaChing supports Owner, Admin, Developer, and Support Agent roles, each with predefined access levels.



### Security & Accounts

**Is my payment data stored in ChaChing?**

ChaChing does not store raw card details. Payment information is securely handled by connected payment providers.

**Can I change my password settings?**

Yes. They can be updated from the **Personal Details** section in Settings.

**Can I change my language settings?**

Chaching currently supports English only. Support for additional languages is planned for future releases.

### API & Developer Questions

**Does ChaChing provide a REST API?**

Yes. ChaChing provides a REST API for managing customers, subscriptions, invoices, and payments.

**Are webhooks available?**

Webhook support is planned and currently under development.

