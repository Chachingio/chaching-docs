# Products

The **Products** section allows you to manage billable items, define pricing models, and configure tax behavior for subscriptions and one-time charges.
---
## Filter Tabs

| **Tab** | **Description** |
| --- | --- |
| **All products** | Displays all products regardless of tax configuration. |
| **Tax Rates** | Displays products grouped or filtered by tax-related settings. |
---
## Additional Filters

### Created Filter

| **Field** | **Description** |
| --- | --- |
| **Date filter type** | Dropdown options: *In the last, Is equal to, Between, On or After, Before or on*. |
| **Dynamic date fields** | Input fields adjust based on selected filter type. |
| **Apply button** | Applies the selected creation-date filter. |

### Tax Category Filter

| **Field** | **Description** |
| --- | --- |
| **Tax category selector** | Dropdown list of available tax categories. |
| **Apply button** | Filters products by the selected tax category. |
---
## Create New Product

### Create New Product Button

Opens a side panel to define a new product and its billing configuration.
{% img src="./images/Create new product.svg" alt="Create new product.svg" withLightbox=true width="" height="" /%}

### Product Details

| **Name** | **Description** | **Required / Optional** |
| --- | --- | --- |
| **Product name** | Display name of the product shown in checkout and billing flows. | Required |
| **Product description** | Optional description shown internally or at checkout. | Optional |
| **Product tax code** | Tax classification applied to the product. | Required |
| **Product image** | Image shown at checkout. Supports PNG/JPEG formats under 2 MB. | Optional |
| **Product type** | Defines billing behavior: *Recurring* or *One-off*. | Required |
| **Amount** | Price of the product before tax adjustments. | Required |
| **Currency** | Currency used for pricing (default: USD). | Required |
| **Include tax in price** | Determines whether tax is included in the displayed price. | Required |
| **Billing period** | Billing frequency for recurring products (e.g., monthly). | Required (Recurring only) |
| **Create product** | Saves the product and makes it available for pricing and subscriptions. | — |
---
## Products Table
{% img src="./images/Products.svg" alt="Products.svg" withLightbox=true width="" height="" /%}

| **Column** | **Description** |
| --- | --- |
| **Product Name** | Name of the product. |
| **Amount** | Base price of the product. |
| **Tax Category** | Applied tax classification. |
| **Created** | Product creation date. |
| **Updated** | Date of last modification. |

Clicking a product name in the table row opens the **Product Details** page.
{% img src="./images/Products.svg" alt="Products.svg" withLightbox=true width="" height="" /%}

## Product Details Page
{% img src="./images/Product details.svg" alt="Product details.svg" withLightbox=true width="" height="" /%}

### Header

| **Element** | **Description** |
| --- | --- |
| **Product name** | Display name of the product. |
| **Status** | Indicates whether the product is active. |
| **Delete** | Permanently deletes the product. |
| **Edit** | Opens the product for editing. |

### Pricing Section

| **Field** | **Description** |
| --- | --- |
| **Price** | Configured price for the product. |
| **Frequency** | Billing interval (e.g., 30-day period). |
| **Subscriptions** | Number of active subscriptions using this price. |
| **Created** | Price creation date. |
| **Description** | Optional pricing description. |

### Product Metadata (Details Section)

| **Field** | **Description** |
| --- | --- |
| **Created** | Date the product was created. |
| **Last modified** | Date the product was last updated. |
| **Tax category** | Assigned tax category. |
| **Product ID** | Unique system identifier for the product. |

### Create New Pricing
{% img src="./images/Product details - Create new pricing.svg" alt="Product details - Create new pricing.svg" withLightbox=true width="" height="" /%}

Opens a side panel to add a new price to the product.

| **Name** | **Description** | **Required / Optional** |
| --- | --- | --- |
| **Pricing type** | Select *Recurring* or *One-off*. | Required |
| **Amount** | Price value for this pricing option. | Required |
| **Currency** | Currency for pricing (default: USD). | Required |
| **Price description** | Optional description for internal reference. | Optional |
| **Include tax in price** | Determines whether tax is included in the price. | Required |
| **Billing period** | Frequency for recurring pricing. | Required (Recurring only) |
| **Create price** | Saves the new pricing option. | — |