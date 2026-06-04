# Run Spec 20260604-141036-017051

## Updated specs

### Iteration 1 — 2026-06-04 14:17:20Z — failed layer: silver (run: 20260604-141241-d794fd)
- **Root cause (1-line summary)**: Silver-layer Spark execution failed without a column-level traceback; likely caused by schema-assumption or join-shape issues during enrichment view creation.
- **Cross-table audit**:
  - SalesLT/Address: yes — participates in customer-address enrichment joins and may contribute duplicate column names after snake_case conversion.
  - SalesLT/Customer: yes — participates in customer-address enrichment joins and may contribute duplicate column names after snake_case conversion.
  - SalesLT/CustomerAddress: yes — junction table used in enrichment joins and commonly introduces overlapping key columns.
  - SalesLT/Product: yes — participates in product enrichment joins and may contribute overlapping keys.
  - SalesLT/ProductCategory: yes — may be joined into product enrichment and introduce duplicate attribute names.
  - SalesLT/ProductDescription: yes — participates in product enrichment joins and may introduce overlapping description columns.
  - SalesLT/ProductModel: yes — participates in product enrichment joins and may introduce overlapping model attributes.
  - SalesLT/ProductModelProductDescription: yes — junction table used in enrichment joins and commonly introduces overlapping keys.
  - SalesLT/SalesOrderDetail: no — not part of the specified enrichment views in Silver.
  - SalesLT/SalesOrderHeader: no — not part of the specified enrichment views in Silver.
- **Fix approach**: GENERALIZE — the risk is systemic across multiple Silver enrichment joins, so a single defensive join-and-projection rule should be applied to all Silver enrichment datasets.
- **What was changed**:
  - Added explicit Silver join rules requiring table aliases and fully qualified join keys.
  - Added a post-join projection requirement to eliminate duplicate column names before writing Silver outputs.
  - Added validation that all declared business keys exist after snake_case conversion before deduplication logic runs.

## Inputs
- Workspace: `ad938c56-0933-47f8-9f87-c862248e89e7`
- Source Lakehouse: **SalesLT** (`47f5fdf7-1902-471b-958f-5a1e9430070e`)
- Tables to ingest into Bronze:
  - `SalesLT/Address`
  - `SalesLT/Customer`
  - `SalesLT/CustomerAddress`
  - `SalesLT/Product`
  - `SalesLT/ProductCategory`
  - `SalesLT/ProductDescription`
  - `SalesLT/ProductModel`
  - `SalesLT/ProductModelProductDescription`
  - `SalesLT/SalesOrderDetail`
  - `SalesLT/SalesOrderHeader`
- Target Lakehouse: **i**

## Generic guidance

Apply these reference skills/agents at all times:
- FabricDataEngineer agent: https://github.com/microsoft/skills-for-fabric/blob/main/agents/FabricDataEngineer.agent.md
- e2e-medallion-architecture skill: https://github.com/microsoft/skills-for-fabric/tree/main/skills/e2e-medallion-architecture
- spark-authoring-cli skill: https://github.com/microsoft/skills-for-fabric/tree/main/skills/spark-authoring-cli
- powerbi-authoring-cli skill: https://github.com/microsoft/skills-for-fabric/tree/main/skills/powerbi-authoring-cli
- powerbi-consumption-cli skill: https://github.com/microsoft/skills-for-fabric/tree/main/skills/powerbi-consumption-cli
- powerbi-semantic-model-authoring: https://github.com/RuiRomano/powerbi-agentic-plugins/tree/main/plugins/powerbi/skills/powerbi-semantic-model-authoring
- powerbi-report-authoring: https://github.com/RuiRomano/powerbi-agentic-plugins/tree/main/plugins/powerbi/skills/powerbi-report-authoring

All generated notebooks must be designed to run incrementally on a daily schedule unless the spec explicitly states otherwise. Use idempotent write patterns (mode='overwrite' with overwriteSchema=true, or merge/upsert where appropriate) so that re-running the same notebook the next day produces correct, up-to-date results without duplicates or failures.

For all Silver and downstream joins:
- Use explicit DataFrame aliases.
- Use fully qualified join references (for example, `c.customer_id = ca.customer_id`).
- Never write a joined DataFrame containing duplicate column names.
- Immediately after every join, project a curated column list with unique output column names.
- After snake_case conversion and before deduplication, validate that each table's declared business key columns are present; fail fast with a descriptive error if a required key is missing.

## Bronze

Land each source table into the `bronze` schema as Delta tables preserving source column names and datatypes.

Common metadata columns added to every bronze table:
- ingestion_timestamp
- ingestion_date
- source_system = 'SalesLT'
- source_table
- run_id
- source_file_or_object (if available)

Write strategy:
- Daily incremental ingestion.
- Merge/upsert based on business key where practical; otherwise overwrite refreshed snapshots.
- Retain all source attributes including rowguid and ModifiedDate.
- Enable schema evolution.

Partitioning:
- Large transactional tables:
  - bronze.sales_order_header partition by year/month derived from OrderDate.
  - bronze.sales_order_detail partition by year/month derived from ModifiedDate.
- Master/reference tables:
  - Address
  - Customer
  - CustomerAddress
  - Product
  - ProductCategory
  - ProductDescription
  - ProductModel
  - ProductModelProductDescription
  - Use unpartitioned Delta tables unless data volume grows significantly.

Source-to-bronze mappings:
- SalesLT/Address → bronze.address
- SalesLT/Customer → bronze.customer
- SalesLT/CustomerAddress → bronze.customer_address
- SalesLT/Product → bronze.product
- SalesLT/ProductCategory → bronze.product_category
- SalesLT/ProductDescription → bronze.product_description
- SalesLT/ProductModel → bronze.product_model
- SalesLT/ProductModelProductDescription → bronze.product_model_product_description
- SalesLT/SalesOrderHeader → bronze.sales_order_header
- SalesLT/SalesOrderDetail → bronze.sales_order_detail

## Silver

Standard transformations:
- Convert all column names to snake_case.
- Standardize timestamp fields to UTC.
- Trim string fields.
- Normalize empty strings to null where appropriate.
- Preserve source keys.
- Add audit columns:
  - created_at
  - updated_at
  - run_id
  - record_source
- Remove exact duplicates.
- OPTIMIZE and VACUUM according to Fabric best practices.

Deduplication strategy by table:
- silver.address
  - Key: address_id
  - Keep latest modified_date.
- silver.customer
  - Key: customer_id
  - Keep latest modified_date.
  - Mask or exclude password_hash and password_salt from downstream Gold models.
- silver.customer_address
  - Key: (customer_id, address_id)
  - Keep latest modified_date.
- silver.product
  - Key: product_id
  - Keep latest modified_date.
- silver.product_category
  - Key: product_category_id
  - Keep latest modified_date.
- silver.product_description
  - Key: product_description_id
  - Keep latest modified_date.
- silver.product_model
  - Key: product_model_id
  - Keep latest modified_date.
- silver.product_model_product_description
  - Key: (product_model_id, product_description_id, culture)
  - Keep latest modified_date.
- silver.sales_order_header
  - Key: sales_order_id
  - Keep latest modified_date.
- silver.sales_order_detail
  - Key: sales_order_detail_id
  - Alternative key: (sales_order_id, sales_order_detail_id).
  - Keep latest modified_date.

Silver business enrichment:
- Build product descriptive view by joining:
  - product
  - product_model
  - product_model_product_description
  - product_description
- Required join keys:
  - product.product_model_id = product_model.product_model_id
  - product_model.product_model_id = product_model_product_description.product_model_id
  - product_model_product_description.product_description_id = product_description.product_description_id
- Use aliases for all joined tables and project a unique output schema with no duplicate column names.

- Build customer address enrichment view by joining:
  - customer
  - customer_address
  - address
- Required join keys:
  - customer.customer_id = customer_address.customer_id
  - customer_address.address_id = address.address_id
- Use aliases for all joined tables and project a unique output schema with no duplicate column names.

- Derive order lifecycle attributes:
  - order_year
  - order_month
  - ship_delay_days
  - order_status_description (based on status values if business definitions become available)

## Gold

Data pattern assessment:
- Fact-like tables:
  - SalesOrderHeader
  - SalesOrderDetail
- Dimension-like tables:
  - Customer
  - Address
  - Product
  - ProductCategory
  - ProductDescription
  - ProductModel
- Junction tables:
  - CustomerAddress
  - ProductModelProductDescription

Recommended star schema:

Dimensions:

- gold.dim_customer
  - Grain: one row per customer_id.
  - Source: silver.customer.
  - Include customer business attributes.
  - Exclude password_hash and password_salt.
  - Optional address rollup through customer_address.

- gold.dim_address
  - Grain: one row per address_id.
  - Source: silver.address.
  - Include address_line1, city, postal_code and related attributes.

- gold.dim_product
  - Grain: one row per product_id.
  - Source: silver.product plus product/category/model/description enrichments.
  - Include product_number, color, size, standard_cost, list_price.
  - Include category and description attributes when available.

- gold.dim_product_category
  - Grain: one row per product_category_id.
  - Source: silver.product_category.
  - Include parent_product_category_id hierarchy support.

- gold.dim_date
  - Grain: one row per calendar date.
  - Used for order, due, and ship date analysis.

Facts:

- gold.fact_sales_order_line
  - Grain: one row per sales_order_detail_id.
  - Source:
    - silver.sales_order_detail
    - silver.sales_order_header
  - Keys:
    - customer_id
    - product_id
    - order_date_key
    - due_date_key
    - ship_date_key
    - ship_to_address_id
    - bill_to_address_id
  - Measures:
    - order_qty
    - unit_price
    - unit_price_discount
    - line_sales_amount = order_qty * unit_price
    - line_discount_amount
    - net_line_sales

- gold.fact_sales_order_header
  - Grain: one row per sales_order_id.
  - Source: silver.sales_order_header.
  - Measures:
    - subtotal
    - tax_amt
    - freight
  - Useful for order-level operational reporting.

Modeling alternatives:
- Option A (recommended): Use fact_sales_order_line as primary reporting fact because it supports product-level analysis.
- Option B: Use only a single consolidated sales fact combining header and detail attributes if simpler reporting is preferred.
- User may edit the spec to select either approach.

## Test

All tests append one row to:
- test.test_results

Schema:
- run_id
- test_name
- layer
- table_name
- status
- actual
- expected
- details
- checked_at

Standard tests:

1. Row count reconciliation
- Compare bronze vs silver row counts.
- Threshold: within 1%.
- Execute for all ingested tables.

2. Gold dimension primary key not null
- dim_customer.customer_id
- dim_address.address_id
- dim_product.product_id
- dim_product_category.product_category_id
- dim_date.date_key

3. Gold dimension primary key uniqueness
- Validate uniqueness of:
  - customer_id
  - address_id
  - product_id
  - product_category_id
  - date_key

4. Referential integrity
- fact_sales_order_line.customer_id exists in dim_customer.
- fact_sales_order_line.product_id exists in dim_product.
- fact_sales_order_line.order_date_key exists in dim_date.
- fact_sales_order_line.ship_to_address_id exists in dim_address.
- fact_sales_order_line.bill_to_address_id exists in dim_address.
- dim_product.product_category_id exists in dim_product_category when populated.

5. Business-rule sanity checks
- order_qty > 0 for all sales fact rows.
- unit_price >= 0.
- subtotal >= 0.
- tax_amt >= 0.
- freight >= 0.
- order_date <= due_date when both values exist.
- ship_date >= order_date when both values exist.

6. Optional enrichment quality checks
- Percentage of fact rows matched to customer dimension > 99%.
- Percentage of fact rows matched to product dimension > 99%.

## Semantic model

Mode:
- Direct Lake

Tables:
- fact_sales_order_line
- fact_sales_order_header
- dim_customer
- dim_product
- dim_product_category
- dim_address
- dim_date

Relationships:
- fact_sales_order_line.customer_id → dim_customer.customer_id
- fact_sales_order_line.product_id → dim_product.product_id
- dim_product.product_category_id → dim_product_category.product_category_id
- fact_sales_order_line.order_date_key → dim_date.date_key
- fact_sales_order_line.ship_to_address_id → dim_address.address_id
- fact_sales_order_line.bill_to_address_id → dim_address.address_id
- fact_sales_order_header.customer_id → dim_customer.customer_id

Explicit measures:
- Total Sales = SUM(net_line_sales)
- Gross Sales = SUM(line_sales_amount)
- Total Discount Amount = SUM(line_discount_amount)
- Total Quantity Sold = SUM(order_qty)
- Total Orders = DISTINCTCOUNT(sales_order_id)
- Average Order Value = DIVIDE([Total Sales],[Total Orders])
- Total Freight = SUM(freight)
- Total Tax = SUM(tax_amt)
- Average Unit Price = AVERAGE(unit_price)

Hierarchies:
- Date: Year → Quarter → Month → Day
- Product Category: Parent Category → Category (where hierarchy data is available)
- Geography: City → Postal Code

## Report

Page 1: Sales Overview
- KPI cards:
  - Total Sales
  - Total Orders
  - Total Quantity Sold
  - Average Order Value
- Line chart:
  - Total Sales by Month
- Bar chart:
  - Total Sales by Product Category
- Top N products visual:
  - Top products by Total Sales

Page 2: Customer and Geography
- Bar chart:
  - Sales by Customer
- Map or filled map:
  - Sales by City and Postal Code
- Matrix:
  - Customer by Product Category
- Trend visual:
  - Orders over time by customer

Page 3: Product Performance
- Scatter plot:
  - List Price vs Total Sales
- Bar chart:
  - Quantity Sold by Product
- Decomposition tree:
  - Sales by Category → Product → Customer
- Product attribute slicers:
  - Color
  - Size

Page 4: Operational Orders
- Orders by Status
- Order fulfillment timing analysis
- Ship delay distribution
- Freight and tax trend analysis

Page 5: Data Quality
- Test pass/fail summary
- Failed test details table
- Row count reconciliation metrics
- Referential integrity status
- Data freshness indicators using latest modified_date and ingestion timestamps

## Data Agent

Role:
- Sales Analytics Copilot for order, customer, product, pricing, and fulfillment analysis based on the Direct Lake semantic model.

Domain hints:
- Analyze sales orders and order lines.
- Compare product performance.
- Analyze customer purchasing patterns.
- Evaluate pricing, discounts, freight, and tax impacts.
- Explore geographic sales distribution using address information.

Starter questions:
- What are the top selling products by revenue?
- Which customers generated the most sales?
- How has sales revenue changed month over month?
- Which product categories contribute the most revenue?
- What is the average order value over time?
- Which cities generate the highest sales?
- What products have the highest quantities sold?
- How much revenue was reduced through discounts?
- What are the trends in freight and tax charges?
- Which customers have the largest number of orders?

Guardrails:
- Use only data exposed through the semantic model.
- Do not expose password_hash or password_salt.
- Do not infer customer information not present in the model.
- Clearly distinguish calculated metrics from stored values.
- Return aggregated results whenever possible.
- Identify missing or incomplete dimension matches when encountered.
- Do not generate write-back actions or source-system updates.
- Respect semantic model security and permissions.