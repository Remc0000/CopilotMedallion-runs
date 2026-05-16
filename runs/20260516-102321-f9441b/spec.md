# Run Spec 20260516-102013-21d323

## Inputs
- Workspace: `11a41516-e10e-401a-9eeb-4590668a9b2b`
- Source Lakehouse: **SalesLT** (`efe41f78-82b7-47ee-9780-2d78372bfdf3`)
- Tables to ingest into Bronze:
  - `Address`
  - `Customer`
  - `CustomerAddress`
  - `Product`
  - `ProductCategory`
  - `ProductDescription`
  - `ProductModel`
  - `ProductModelProductDescription`
  - `SalesOrderDetail`
  - `SalesOrderHeader`
  - `vGetAllCategories`
  - `vProductAndDescription`
  - `vProductModelCatalogDescription`
- Target Lakehouse: **CopilotMedallion_20260516_102013**

## Generic guidance
Apply these reference skills/agents at all times:
- FabricDataEngineer agent: https://github.com/microsoft/skills-for-fabric/blob/main/agents/FabricDataEngineer.agent.md
- e2e-medallion-architecture skill: https://github.com/microsoft/skills-for-fabric/tree/main/skills/e2e-medallion-architecture
- spark-authoring-cli skill: https://github.com/microsoft/skills-for-fabric/tree/main/skills/spark-authoring-cli
- powerbi-authoring-cli skill: https://github.com/microsoft/skills-for-fabric/tree/main/skills/powerbi-authoring-cli
- powerbi-consumption-cli skill: https://github.com/microsoft/skills-for-fabric/tree/main/skills/powerbi-consumption-cli
- powerbi-semantic-model-authoring: https://github.com/RuiRomano/powerbi-agentic-plugins/tree/main/plugins/powerbi/skills/powerbi-semantic-model-authoring
- powerbi-report-authoring: https://github.com/RuiRomano/powerbi-agentic-plugins/tree/main/plugins/powerbi/skills/powerbi-report-authoring

Cross-cutting code rules:
- Use defensive column references with explicit aliasing in all joins and selects.
- After every join, use alias-prefixed column references to avoid ambiguity.
- Assert column existence before groupBy or aggregation operations.
- Defensive REST handling: check if responses are None before calling `.get()`, raise errors if so.
- Avoid `saveAsTable`; use idempotent overwrite patterns with `mode("overwrite")` and partitioning.
- Parameterize all configurable values such as paths, table names, and partitions.
- Use try/except blocks that log errors with `_save_error(layer, e)` and re-raise exceptions to fail loudly.

## Bronze
- Ingest all listed source tables as-is into the `bronze` schema with no transformations.
- Add metadata columns: `_ingest_timestamp` (current timestamp), `_source_file` (if applicable).
- Use `mode("overwrite")` for idempotency on each run.
- Partition large tables by date columns where available:
  - `SalesOrderHeader` partitioned by `OrderDate` (date part only).
  - `SalesOrderDetail` partitioned by `SalesOrderID` (to cluster related details).
- Other tables ingested without partitioning due to smaller size or lack of natural partition keys.
- Preserve all columns as-is, including binary and GUID columns.

## Silver
- Clean and standardize column names to snake_case for all tables.
- Deduplicate records based on natural keys:
  - `Address`: dedup key = `address_id`
  - `Customer`: dedup key = `customer_id`
  - `CustomerAddress`: dedup key = composite (`customer_id`, `address_id`, `address_type`)
  - `Product`: dedup key = `product_id`
  - `ProductCategory`: dedup key = `product_category_id`
  - `ProductDescription`: dedup key = `product_description_id`
  - `ProductModel`: dedup key = `product_model_id`
  - `ProductModelProductDescription`: dedup key = composite (`product_model_id`, `product_description_id`, `culture`)
  - `SalesOrderHeader`: dedup key = `sales_order_id`
  - `SalesOrderDetail`: dedup key = `sales_order_detail_id`
- For views (`vGetAllCategories`, `vProductAndDescription`, `vProductModelCatalogDescription`), ingest as read-only silver tables for easier querying and joining.
- Add audit columns: `_load_timestamp` (current timestamp), `_record_source` (source table name).
- Convert timestamps to UTC if needed.
- Validate and cast decimal and numeric columns to appropriate precision.
- Optimize tables with `OPTIMIZE` command after ingestion, especially on `SalesOrderHeader` and `SalesOrderDetail`.

## Gold
- Model dimensions and facts based on actual columns and relationships:

### Dimensions
- **dim_address**: from `Address`
  - PK: `address_id`
  - Attributes: address_line1, address_line2, city, state_province, country_region, postal_code
- **dim_customer**: from `Customer`
  - PK: `customer_id`
  - Attributes: name_style, title, first_name, middle_name, last_name, suffix, company_name, salesperson, email_address, phone
- **dim_product**: from `Product`
  - PK: `product_id`
  - Attributes: name, product_number, color, size, weight, standard_cost, list_price, sell_start_date, sell_end_date, discontinued_date
  - FK: `product_category_id` → dim_product_category
  - FK: `product_model_id` → dim_product_model
- **dim_product_category**: from `ProductCategory`
  - PK: `product_category_id`
  - Attributes: parent_product_category_id, name
- **dim_product_description**: from `ProductDescription`
  - PK: `product_description_id`
  - Attribute: description
- **dim_product_model**: from `ProductModel`
  - PK: `product_model_id`
  - Attributes: name, catalog_description
- **dim_customer_address**: from `CustomerAddress`
  - Composite PK: (`customer_id`, `address_id`, `address_type`)
  - FK: `customer_id` → dim_customer
  - FK: `address_id` → dim_address
  - Attribute: address_type

### Facts
- **fact_sales_order_header**: from `SalesOrderHeader`
  - PK: `sales_order_id`
  - FK: `customer_id` → dim_customer
  - FK: `ship_to_address_id` → dim_address
  - FK: `bill_to_address_id` → dim_address
  - Measures: subtotal, tax_amt, freight, total_due
  - Attributes: revision_number, order_date, due_date, ship_date, status, online_order_flag, sales_order_number, purchase_order_number, account_number, ship_method, credit_card_approval_code, comment
- **fact_sales_order_detail**: from `SalesOrderDetail`
  - PK: `sales_order_detail_id`
  - FK: `sales_order_id` → fact_sales_order_header
  - FK: `product_id` → dim_product
  - Measures: order_qty, unit_price, unit_price_discount, line_total

### Junction / Bridge Tables
- **bridge_product_model_product_description**: from `ProductModelProductDescription`
  - Composite PK: (`product_model_id`, `product_description_id`, `culture`)
  - FK: `product_model_id` → dim_product_model
  - FK: `product_description_id` → dim_product_description
  - Attribute: culture

### Alternative modeling options
- Could merge `dim_product_description` and `dim_product_model` with their bridge table into a single denormalized product description dimension for simplicity. --> do
- Could treat `CustomerAddress` as a fact table if analyzing address usage over time, but current design treats it as a dimension bridge. --> Don't do

### Data quality tests
- Row count > 0 for all gold tables.
- No nulls in PK columns.
- Unique PK enforcement for all dimension and fact tables.
- Referential integrity checks for all FK columns.
- Value range checks on numeric columns (e.g., order_qty > 0, unit_price >= 0).

## Semantic model
- Star schema with:
  - Fact tables: `fact_sales_order_header`, `fact_sales_order_detail`
  - Dimension tables: `dim_customer`, `dim_address`, `dim_product`, `dim_product_category`, `dim_product_model`, `dim_product_description`, `dim_customer_address`
- Relationships:
  - `fact_sales_order_header.customer_id` → `dim_customer.customer_id`
  - `fact_sales_order_header.ship_to_address_id` → `dim_address.address_id`
  - `fact_sales_order_header.bill_to_address_id` → `dim_address.address_id`
  - `fact_sales_order_detail.sales_order_id` → `fact_sales_order_header.sales_order_id`
  - `fact_sales_order_detail.product_id` → `dim_product.product_id`
  - `dim_product.product_category_id` → `dim_product_category.product_category_id`
  - `dim_product.product_model_id` → `dim_product_model.product_model_id`
  - `bridge_product_model_product_description.product_model_id` → `dim_product_model.product_model_id`
  - `bridge_product_model_product_description.product_description_id` → `dim_product_description.product_description_id`
  - `dim_customer_address.customer_id` → `dim_customer.customer_id`
  - `dim_customer_address.address_id` → `dim_address.address_id`

### Measures (DAX)
- Total Sales Amount = SUM(fact_sales_order_header[total_due])
- Total Order Quantity = SUM(fact_sales_order_detail[order_qty])
- Average Unit Price = AVERAGE(fact_sales_order_detail[unit_price])
- Total Discount Amount = SUM(fact_sales_order_detail[unit_price_discount] * fact_sales_order_detail[order_qty])
- Total Freight = SUM(fact_sales_order_header[freight])
- Number of Orders = DISTINCTCOUNT(fact_sales_order_header[sales_order_id])

## Report
- **Page 1: Sales Overview**
  - Visuals: Card for Total Sales Amount, Bar chart of Sales by Product Category, Line chart of Sales over OrderDate
- **Page 2: Customer Insights**
  - Visuals: Table of Top Customers by Sales, Map of Customer Addresses by City/State, Pie chart of Sales by SalesPerson
- **Page 3: Product Performance**
  - Visuals: Table of Products with Sales and Quantity, Matrix of Sales by Product Model and Description, Bar chart of Discounts by Product
- **Page 4: Data Quality**
  - Visuals: KPI cards for row counts of gold tables, bar charts for null counts in PK columns, tables showing referential integrity violations if any

## Data Agent
- Role: Senior Sales Data Analyst assistant focused on sales order, customer, and product data insights.
- Domain hints: Sales orders, customers, products, product categories, addresses, discounts, and sales performance.
- Starter questions:
  1. What are the total sales and order quantities for the last quarter?
  2. Which customers have generated the highest revenue?
  3. How do sales vary by product category and product model?
  4. What is the average discount applied per product?
  5. Which salespersons have the highest sales volumes?
  6. How many orders were shipped late based on due and ship dates?
  7. What are the top shipping methods used?
  8. Are there any customers with multiple addresses and how does that affect sales?
  9. What is the trend of new product sales over time?
  10. Are there any data quality issues with missing or duplicate keys?
- Guardrails:
  - Only answer questions based on the modeled gold tables and semantic model.
  - Do not speculate beyond the available columns and relationships.
  - Validate all user queries against data quality tests before answering.
  - Provide clear explanations of measures and dimensions used in answers.
  - Alert users if requested data is outside the available date ranges or incomplete.
