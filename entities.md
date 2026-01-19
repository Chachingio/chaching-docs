# Entities Overview

This section defines the core entities used in Chaching and explains how data is organized across **Companies**, **Accounts**, **Users**. It explains the main building blocks of the Chaching system:

- **Companies** - your main organizational workspace
- **Accounts** - separate spaces for banking and financial integrations
- **Users** - the people who access and manage your company
- **Branding** - the shared visual identity applied across your company
---
## Company

A **Company** is the highest-level organizational entity in Chaching.

- A Company acts as the **tenant** of the system.
- All data inside Chaching exists **inside a Company**.
- A Company can contain multiple **Accounts**.
- The Company context is always included in the authentication token.

**Purpose of a Company**

A Company holds information and settings that apply **globally** to all Accounts under it, including:

- Company branding (logo, colors)
- Company-wide settings shared across all Accounts
- Metadata describing the tenant itself

**Important Notes**

- Companies are **always created**. Even if a user creates only one account, it still exists within a tenant Company.
- Companies are **not Accounts**; each Account belongs to exactly one Company.
- Multi-tenant functionality (e.g., Coca-Cola + Starbucks under one umbrella) is not supported (coming soon).
---
## Accounts

An **Account** represents an operational space inside a Company, similar to:

- A billing account
- A financial space
- A Stripe or Plaid account
- A segmented workspace within a tenant

**Key Characteristics**

- A single Company can have **multiple Accounts**.
- Accounts store **banking and billing information**.
- Each Account has its own:
    - Linked bank accounts (e.g., Plaid)
    - Billing configuration
    - Stripe-related data
    - API keys and tokens
- The **current Account ID** is always encoded in the user’s access token.
---
## Users

Users interact with Chaching through a Company while operating under a **current Account**.

**User Context**

- A user is authenticated into a **Company**.
- The user selects a **current Account** from a dropdown or UI selector.
- The system injects both Company ID and Account ID into the authentication token.

**Permissions**

Visibility and access to settings depend on:

- User role (super admin, admin, standard user)
- Account-level permissions included in the token
- Company-wide permissions for global settings
---
## Branding

**Branding is Company-wide**

- Logo, colors, themes, and other branding elements are stored **in the Company entity**.
- Branding does NOT vary between Accounts.
- Branding stays the same for all Accounts under the company.
---
## Settings Model

Chaching uses two categories of settings:

### Company-Level Settings

Visible to: Admins / Super Admins

Include:

- Company branding
- Company information
- Shared configurations

These settings appear **regardless of selected Account**, because they apply to the entire Company.

### Account-Level Settings

Visible only when:

- A current Account is selected
- User has sufficient permissions

Include:

- Plaid account linking
- Stripe account settings
- Account-specific billing and financial data
- Account-specific tokens
---
## Entity Summary Table

| Entity | Represents | Stores | Shared Across Accounts? | Notes |
| --- | --- | --- | --- | --- |
| **Company (Tenant)** | Top-level organization | Branding, company settings | Yes | Exists for every user; not per-Account |
| **Account** | Financial/operational workspace | Banking, billing, Plaid, Stripe, account tokens | No | Multiple Accounts per Company |
| **User** | Person accessing system | Permissions, roles | — | Operates inside a Company with a selected Account |
| **Branding** | Tenant visuals | Logo, colors | Yes | Stored in Company |