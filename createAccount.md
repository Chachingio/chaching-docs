# Adding Account

## Overview

Chaching supports multiple Live and Sandbox accounts. Each account is fully isolated and has its own Stripe integration, customers, subscriptions, products, invoices, and billing data.

There is **no limit** to the number of accounts you can create.
---
## When to Add a New Account

You should create a new account if you need to:

* Manage a **separate Stripe account**
* Keep **test and production data isolated**
* Operate **multiple brands or business units**
* Manage **client accounts** (agency or reseller model)

---
## How to Add a New Account

1. Go to **Your Account**
2. Select one of the following options:
{% img src="./images/Workspace.svg" alt="Workspace.svg" withLightbox=true width="" height="" /%}
  Add a Live Account:
   * Switch to workspace → Add an account
   * Creates a new Live account
   * Starts the standard onboarding flow
  
  Add a Sandbox Account:
   * Switch to sandbox → Add new sandbox
   * Creates a new Sandbox (Test Mode) account
   * Starts the same onboarding flow

The full step-by-step onboarding process (account details, Stripe connection, migration, bank setup) is described in [User Registration & Onboarding Guide](./onboarding.md).

---
## Sandbox vs Live Accounts

* Sandbox accounts follow the same onboarding steps
* Once completed, the Dashboard clearly shows **Test Mode**
* Sandbox and live accounts are fully isolated
---
## Note

**Incomplete setup deletes the account**

* If you exit onboarding after creating an account but before completion, the account is **automatically deleted**
* Deleted accounts **cannot be recovered**
* You must restart the account creation process

**No account limit**

* You can create **unlimited accounts**
* Each account requires its own valid Stripe credentials
---
## Switching Between Accounts

Use the **Account Selector** to switch accounts.

When switching accounts:

* Customers, subscriptions, invoices, and metrics change
* Active Stripe connection changes

What stays the same:

* Your user profile
* Login credentials
* Personal settings
---
## Data Isolation

Each account is completely isolated:

* Data from one account is never visible in another
* Stripe operations affect only the active account
* Reports and analytics are account-specific





