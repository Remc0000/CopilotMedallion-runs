# Run Spec 20260604-182734-cbfcb3

## Updated specs

### Iteration 1 — 2026-06-04 18:30:10Z — failed layer: bronze (run: 20260604-182917-5456ba)
- **Root cause (1-line summary)**: Bronze ingestion failed at runtime; the most likely systemic cause is assuming every source table contains a partitioning/incremental watermark column (for example `ModifiedDate`) when some SalesLT tables may not expose the expected column shape.
- **Cross-table audit**:
  - Address: yes — partition and incremental logic references `ModifiedDate`.
  - Customer: yes — partition and incremental logic references `ModifiedDate`.
  - CustomerAddress: yes — partition and incremental logic references `ModifiedDate`.
  - Product: yes — partition and incremental logic references `ModifiedDate`.
  - ProductCategory: yes — partition and incremental logic references `ModifiedDate`.
  - ProductDescription: yes — partition and incremental logic references `ModifiedDate`.
  - ProductModel: yes — partition and incremental logic references `ModifiedDate`.
  - ProductModelProductDescription: yes — partition and incremental logic references `ModifiedDate`.
  - SalesOrderDetail: yes — partition logic references `ModifiedDate` as a fallback.
  - SalesOrderHeader: yes — uses `OrderDate` instead of `ModifiedDate`; similar failure can occur if expected watermark columns are hard-coded.
- **Fix approach**: GENERALIZE — the issue is a cross-table schema-assumption risk affecting all Bronze ingestions and should be handled uniformly.
- **What was changed**:
  - Added Bronze schema-discovery requirements before applying incremental filters, MERGE keys, or partition expressions.
  - Added explicit fallback behavior when `ModifiedDate` or other expected partition columns are absent.
  - Required ingestion to preserve available source columns without failing on missing optional columns.

## Inputs
- Workspace: `e8b5ab1d-0b93-4b7d-bfb5-aaf3fbf712d4`
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
- Target Lakehouse: **l**

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

Before applying any filter, partition strategy, MERGE condition, or watermark logic, inspect the actual source schema. Do not reference a column unless it exists in the source DataFrame. When an expected column is absent, use the documented fallback behavior rather than failing the notebook.

## Bronze

Land each source table into the `bronze` schema as a Delta table with source-preserving structure.

Common metadata columns added to every bronze table:
- ingestion_timestamp
- run_id
- source_system = 'SalesLT'
- source_table
- source_file_or_object (if available)

Write strategy:
- Daily incremental load using `ModifiedDate` only when that column exists in the source table.
- If `ModifiedDate` does not exist, perform a full-table load or use an alternative available business timestamp for that table.
- MERGE/UPSERT into bronze on business key(s).
- Preserve original column names and data types.
- Store `rowguid` and `ModifiedDate` exactly as received when present.
- Prior to write, validate the existence of all key, partition, and watermark columns referenced by the ingestion logic. Missing optional columns must not cause notebook failure.

Table-specific keys and partitioning:
- bronze.address
  - Key: AddressID
  - Partition: year(ModifiedDate) when `ModifiedDate` exists; otherwise write unpartitioned.

- bronze.customer
  - Key: CustomerID
  - Partition: year(ModifiedDate) when `ModifiedDate` exists; otherwise write unpartitioned.

- bronze.customer_address
  - Key: CustomerID + AddressID
  - Partition: year(ModifiedDate) when `ModifiedDate` exists; otherwise write unpartitioned.

- bronze.product
  - Key: ProductID
  - Partition: year(ModifiedDate) when `ModifiedDate` exists; otherwise write unpartitioned.

- bronze.product_category
  - Key: ProductCategoryID
  - Partition: year(ModifiedDate) when `ModifiedDate` exists; otherwise write unpartitioned.

- bronze.product_description
  - Key: ProductDescriptionID
  - Partition: year(ModifiedDate) when `ModifiedDate` exists; otherwise write unpartitioned.

- bronze.product_model
  - Key: ProductModelID
  - Partition: year(ModifiedDate) when `ModifiedDate` exists; otherwise write unpartitioned.

- bronze.product_model_product_description
  - Key: ProductModelID + ProductDescriptionID + Culture
  - Partition: year(ModifiedDate) when `ModifiedDate` exists; otherwise write unpartitioned.

- bronze.sales_order_header
  - Key: SalesOrderID
  - Partition: year(OrderDate) when `OrderDate` exists; otherwise write unpartitioned.

- bronze.sales_order_detail
  - Key: SalesOrderDetailID
  - Partition: derived from associated order year when practical; otherwise year(ModifiedDate) only if `ModifiedDate` exists; otherwise write unpartitioned.

## Silver

General processing rules:
- Convert all column names to snake_case.
- Standardize timestamps to UTC where applicable.
- Remove exact duplicates.
- Add audit columns:
  - created_at
  - updated_at
  - pipeline_run_id
- Preserve source business keys.
- OPTIMIZE and VACUUM according to Fabric best practices.

Deduplication strategy by table:
- silver.address
  - Dedup key: address_id
  - Keep latest modified_date

- silver.customer
  - Dedup key: customer_id
  - Keep latest modified_date
  - Trim and normalize email_address
  - Retain salesperson source value for downstream dimension creation

- silver.customer_address
  - Dedup key: customer_id + address_id
  - Keep latest modified_date

- silver.product
  - Dedup key: product_id
  - Keep latest modified_date

- silver.product_category
  - Dedup key: product_category_id
  - Keep latest modified_date

- silver.product_description
  - Dedup key: product_description_id
  - Keep latest modified_date

- silver.product_model
  - Dedup key: product_model_id
  - Keep latest modified_date
  - NOTE: schema contains ProductModelID but no Name column. User requested ProductModel.Name → modelname in Gold. This is not directly possible from the provided schema. Expose ProductModelID and leave model_name null/placeholder unless a source containing the model name is later provided.

- silver.product_model_product_description
  - Dedup key: product_model_id + product_description_id + culture
  - Keep latest modified_date

- silver.sales_order_header
  - Dedup key: sales_order_id
  - Keep latest modified_date

- silver.sales_order_detail
  - Dedup key: sales_order_detail_id
  - Keep latest modified_date

Business transformations:
- Create cleaned salesperson value:
  - Extract username portion from salesperson.
  - Example: adventure-works\jillian0 → jillian0.
  - If no domain separator exists, retain original value.

- Create order-level derived fields:
  - order_year
  - order_month
  - ship_year
  - ship_month

- Create line-level sales calculations:
  - gross_line_amount = order_qty * unit_price
  - discount_amount = order_qty * unit_price * unit_price_discount
  - net_line_amount = gross_line_amount - discount_amount
  - discount_pct

## Gold

Target star schema optimized for Direct Lake sales analytics.

Dimension tables:

### dim_order_date
Source:
- SalesOrderHeader.OrderDate

Key:
- date_key

Attributes:
- calendar date attributes
- year
- quarter
- month
- month name
- week
- day

Hierarchy:
- Year → Quarter → Month → Date

### dim_ship_date
Source:
- SalesOrderHeader.ShipDate

Key:
- ship_date_key

Attributes:
- calendar date attributes
- year
- quarter
- month
- month name
- week
- day

Hierarchy:
- Year → Quarter → Month → Date

### dim_customer
User-requested design:
- Combine Customer and Address.
- Do not use CustomerAddress as an intermediate relationship.

NOTE:
- The provided schema contains no direct relationship between Customer and Address.
- The only available relationship is CustomerAddress(CustomerID, AddressID).
- Therefore the requested join is not technically possible using only Customer and Address.
- Recommended implementation: build dim_customer using Customer joined through CustomerAddress to Address.
- If the user later provides a direct customer-address key, update accordingly.

Attributes:
- customer_id
- company_name
- email_address
- title
- suffix
- city
- postal_code

Hierarchy:
- City → Customer

### dim_salesperson
Source:
- Customer.SalesPerson

Key:
- salesperson_key (surrogate)

Attributes:
- salesperson_username
- salesperson_original_value

Hierarchy:
- Salesperson

### dim_order
Source:
- SalesOrderHeader

Purpose:
- Move non-measure, non-date attributes out of fact table.

Key:
- sales_order_id

Attributes:
- revision_number
- status
- ship_method
- credit_card_approval_code
- comment

Hierarchy:
- Status → Order

### dim_product
User-requested design:
- Combine Product
- ProductCategory
- ProductDescription
- ProductModel
- ProductModelProductDescription (Culture='en')

Join strategy:
- Product.ProductCategoryID → ProductCategory.ProductCategoryID
- Product.ProductModelID → ProductModel.ProductModelID
- ProductModel → ProductModelProductDescription (Culture='en')
- ProductModelProductDescription.ProductDescriptionID → ProductDescription.ProductDescriptionID

Category handling:
- Use ProductCategory parent-child relationship.
- Expose category and subcategory levels when available.

Attributes:
- product_id
- product_number
- color
- size
- weight
- standard_cost
- list_price
- category
- subcategory
- description
- model_name (if source becomes available)
- sell_start_date
- sell_end_date
- discontinued_date

Hierarchies:
- Category → Subcategory → Product
- Color → Product

Fact table:

### fact_sales_order

Source:
- SalesOrderHeader joined to SalesOrderDetail

Grain:
- One row per sales order detail line.

Keys:
- sales_order_id
- sales_order_detail_id
- customer_id
- product_id
- salesperson_key
- date_key
- ship_date_key

Measures stored/calculated from source:
- order_qty
- unit_price
- unit_price_discount
- gross_line_amount
- discount_amount
- net_line_amount
- subtotal
- tax_amt
- freight

Fact/dimension relationships:
- fact_sales_order → dim_order_date
- fact_sales_order → dim_ship_date
- fact_sales_order → dim_customer
- fact_sales_order → dim_salesperson
- fact_sales_order → dim_order
- fact_sales_order → dim_product

## Test

Each test writes one result row into:
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
- Bronze vs Silver row counts for each table.
- Expected variance <= 1%.

2. Gold dimension PK not null
- dim_order_date.date_key
- dim_ship_date.ship_date_key
- dim_customer.customer_id
- dim_product.product_id
- dim_order.sales_order_id
- dim_salesperson.salesperson_key

3. Gold dimension PK uniqueness
- Validate uniqueness of all dimension primary keys.

4. Referential integrity
- fact_sales_order.customer_id exists in dim_customer
- fact_sales_order.product_id exists in dim_product
- fact_sales_order.sales_order_id exists in dim_order
- fact_sales_order.date_key exists in dim_order_date
- fact_sales_order.ship_date_key exists in dim_ship_date
- fact_sales_order.salesperson_key exists in dim_salesperson

5. Business-rule sanity checks
- order_qty > 0
- unit_price >= 0
- discount_pct between 0 and 100
- net_line_amount <= gross_line_amount
- ship_date >= order_date when both values exist

## Semantic model

Mode:
- Direct Lake

Tables:
- dim_order_date
- dim_ship_date
- dim_customer
- dim_salesperson
- dim_order
- dim_product
- fact_sales_order

Relationships:
- Standard one-to-many star schema relationships from dimensions to fact.

Dimension hierarchies:

- Order Date:
  - Year → Quarter → Month → Date

- Ship Date:
  - Year → Quarter → Month → Date

- Product:
  - Category → Subcategory → Product

- Customer:
  - City → Customer

Measures:

- Total Sales =
  - SUM(net_line_amount)

- Gross Sales =
  - SUM(gross_line_amount)

- Total Discount Amount =
  - SUM(discount_amount)

- Average Sale Amount =
  - AVERAGE(net_line_amount)

- Maximum Sale Amount =
  - MAX(net_line_amount)

- Discount Percentage =
  - DIVIDE(SUM(discount_amount), SUM(gross_line_amount))

- Average Discount Percentage =
  - AVERAGE(discount_pct)

- Total Orders =
  - DISTINCTCOUNT(sales_order_id)

- Average Order Quantity =
  - AVERAGE(order_qty)

- Maximum Order Quantity =
  - MAX(order_qty)

- Total Freight =
  - SUM(freight)

- Total Tax =
  - SUM(tax_amt)

Regional analytics:
- Use City from dim_customer as the available geographic attribute.
- NOTE: No state, country, latitude, or longitude fields are present in the supplied schema.

## Report

### Page 1 — Sales Overview

Visuals:
- KPI: Total Sales
- KPI: Average Sale Amount
- KPI: Maximum Sale Amount
- KPI: Total Orders
- Clustered column chart:
  - Sales by Product Category
- Bar chart:
  - Top Products by Total Sales
- Matrix:
  - Category → Subcategory → Product hierarchy

### Page 2 — Regional Performance

Visuals:
- Map visual using Customer City
  - Bubble size: Total Sales
  - Color scale: Average Sale Amount
- Map visual using Customer City
  - Bubble size: Total Orders
  - Color scale: Maximum Sale Amount
- Bar chart:
  - Sales by City
- Table:
  - Highest-performing cities
  - Lowest-performing cities

NOTE:
- Regional analysis is limited to City because no broader geographic fields exist in the provided source.

### Page 3 — Orders & Trends

Visuals:
- Line chart:
  - Monthly Total Sales by Order Date hierarchy
- Line chart:
  - Monthly Average Sale Amount
- Line chart:
  - Monthly Maximum Sale Amount
- Column chart:
  - Orders by Status
- Scatter plot:
  - Order Quantity vs Net Sales

### Page 4 — Discount Analysis

Visuals:
- Bar chart:
  - Salesperson vs Total Discount Amount
- Bar chart:
  - Salesperson vs Discount Percentage
- Table:
  - Salespeople offering largest discounts
- Scatter plot:
  - Discount Percentage vs Total Sales

### Page 5 — Data Quality

Visuals:
- PASS/FAIL summary card
- Test results table from test.test_results
- Failed tests by layer
- Historical test trend by execution date

## Data Agent

Role:
- AI Sales Performance Analyst for the SalesLT reporting solution.
- Answer questions using only the Direct Lake semantic model.
- Specialize in sales trends, regional performance, product performance, customer behavior, order activity, discounts, and salesperson effectiveness.

Domain hints:
- Sales are represented by fact_sales_order.
- Regional analysis is based on customer city.
- Product rollups use Category → Subcategory → Product.
- Order analysis can use both Order Date and Ship Date dimensions.
- Discount metrics are available through discount amount and discount percentage measures.
- Salesperson analysis uses the cleaned username extracted from Customer.SalesPerson.

Starter questions:
- Which cities generate the highest total sales?
- Which cities generate the lowest total sales?
- What are the monthly sales trends over time?
- What is the average sale amount by month?
- What is the maximum sale amount by month?
- Which products generate the most revenue?
- Which product categories are performing best?
- Which salespeople provide the largest discounts?
- What is the average discount percentage by salesperson?
- How do sales compare between order date and ship date views?
- Which customers contribute the most revenue?
- What orders have the highest sales value?

Guardrails:
- Use only fields and measures exposed by the semantic model.
- Do not invent geography beyond available customer city data.
- Clearly state when requested attributes are unavailable in the model.
- Prefer business measures over raw column aggregation.
- Use defined hierarchies when drilling through dimensions.
- Explain calculation logic when discussing discount percentages or averages.
- Do not expose sensitive source fields such as PasswordHash or PasswordSalt.
- If a requested analysis requires unavailable data, identify the missing field and suggest a model enhancement.