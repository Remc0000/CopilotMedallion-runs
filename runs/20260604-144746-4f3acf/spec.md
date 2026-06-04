# Run Spec 20260604-144707-e82bff

## Updated specs

### Iteration 1 — 2026-06-04 14:48:39Z — failed layer: bronze (run: 20260604-144746-4f3acf)
- **Root cause (1-line summary)**: Bronze ingestion failed with a generic Spark session cancellation, indicating the build needs stricter source-table validation and fallback handling before MERGE operations.
- **Cross-table audit**:
  - SalesLT/Address: yes — MERGE depends on AddressID being present and unique enough for key-based processing.
  - SalesLT/Customer: yes — MERGE depends on CustomerID and may contain columns referenced by incremental logic.
  - SalesLT/CustomerAddress: yes — composite-key MERGE depends on CustomerID and AddressID existing.
  - SalesLT/Product: yes — MERGE depends on ProductID and incremental logic may reference ModifiedDate.
  - SalesLT/ProductCategory: yes — key-based ingestion depends on ProductCategoryID.
  - SalesLT/ProductDescription: yes — key-based ingestion depends on ProductDescriptionID.
  - SalesLT/ProductModel: yes — key-based ingestion depends on ProductModelID.
  - SalesLT/ProductModelProductDescription: yes — composite-key ingestion depends on ProductModelID, ProductDescriptionID, and Culture.
  - SalesLT/SalesOrderDetail: yes — key-based ingestion and partitioning depend on expected columns.
  - SalesLT/SalesOrderHeader: yes — key-based ingestion and date partitioning depend on expected columns.
- **Fix approach**: GENERALIZE — the failure signal is not tied to a specific table or column, and the same validation issue could affect any source table during Bronze ingestion.
- **What was changed**:
  - Tightened Bronze ingestion requirements with mandatory schema validation before MERGE.
  - Added explicit fallback behavior when ModifiedDate is absent.
  - Added rules to avoid MERGE execution when required key columns are missing.

### Iteration 2 — 2026-06-04 14:49:39Z — failed layer: bronze (run: 20260604-144746-4f3acf)
- **Root cause (1-line summary)**: Bronze Spark session was cancelled during statement execution; MERGE execution against missing/uninitialized targets or invalid source shapes remains a likely systemic failure mode.
- **Cross-table audit**:
  - SalesLT/Address: yes — target creation and key validation requirements apply.
  - SalesLT/Customer: yes — target creation and incremental logic requirements apply.
  - SalesLT/CustomerAddress: yes — composite-key validation and target existence checks apply.
  - SalesLT/Product: yes — target creation and ModifiedDate fallback requirements apply.
  - SalesLT/ProductCategory: yes — target creation and key validation requirements apply.
  - SalesLT/ProductDescription: yes — target creation and key validation requirements apply.
  - SalesLT/ProductModel: yes — target creation and key validation requirements apply.
  - SalesLT/ProductModelProductDescription: yes — composite-key validation and target existence checks apply.
  - SalesLT/SalesOrderDetail: yes — target creation, key validation, and partition logic apply.
  - SalesLT/SalesOrderHeader: yes — target creation, key validation, and partition logic apply.
- **Fix approach**: GENERALIZE — the same Bronze write-path and validation logic is used across all source tables.
- **What was changed**:
  - Added mandatory source row-count and schema validation before any write operation.
  - Added explicit first-load behavior: create/overwrite target table when it does not already exist; do not execute MERGE against a non-existent target.
  - Added requirement to skip and log empty-source tables instead of executing MERGE statements.

## Inputs
- Workspace: `a9df7114-70df-4a8c-a234-468ba3543444`
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
- Target Lakehouse: **k**

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

Before processing any source table, validate that the source table exists and that all configured key columns for that table are present. Fail that table with a clear validation message rather than executing a MERGE with unresolved schema assumptions.

## Bronze

Land each source table into the `bronze` schema with minimal transformation.

Common approach:
- Preserve source column names and datatypes.
- Add metadata columns:
  - `bronze_ingested_at`
  - `bronze_run_id`
  - `bronze_source_table`
  - `bronze_load_date`
- Use Delta format.
- Incremental ingestion using `ModifiedDate` where available.
- If a source table does not contain a `ModifiedDate` column, ingest the full table and do not reference `ModifiedDate` in filters, watermarks, or MERGE predicates.
- Before any write or MERGE, validate that:
  - the source table exists;
  - the source dataframe schema was successfully read;
  - all configured key columns exist;
  - the source dataframe contains at least one column and can be counted successfully.
- If the source dataframe is empty, log the condition and skip processing for that table; do not execute a MERGE.
- For first-time loads where the bronze target table does not yet exist, create the Delta table using overwrite/create semantics and do not execute a MERGE.
- Only execute MERGE when both the source dataframe and target Delta table exist and all required key columns are present.
- Before any MERGE, validate that all configured key columns exist in the source dataframe.
- Only use the table-specific key columns listed below as MERGE predicates.
- If a required key column is missing, stop processing that table with an explicit validation error and continue processing other tables where possible.
- Partition large transactional tables by load date; leave small master tables unpartitioned.

Table-specific landing:
- `bronze.address`
  - Key: `AddressID`
  - Merge on `AddressID`
- `bronze.customer`
  - Key: `CustomerID`
  - Merge on `CustomerID`
  - Retain sensitive fields for lineage only; downstream silver should evaluate masking/removal of password fields.
- `bronze.customer_address`
  - Composite key: (`CustomerID`, `AddressID`)
  - Merge on composite key.
- `bronze.product`
  - Key: `ProductID`
  - Merge on `ProductID`
  - Retain binary thumbnail column.
- `bronze.product_category`
  - Key: `ProductCategoryID`
- `bronze.product_description`
  - Key: `ProductDescriptionID`
- `bronze.product_model`
  - Key: `ProductModelID`
- `bronze.product_model_product_description`
  - Composite key: (`ProductModelID`, `ProductDescriptionID`, `Culture`)
- `bronze.sales_order_header`
  - Key: `SalesOrderID`
  - Partition by order year/month derived from `OrderDate` or load date.
- `bronze.sales_order_detail`
  - Key: `SalesOrderDetailID`
  - Alternate natural key: (`SalesOrderID`, `SalesOrderDetailID`)
  - Partition by load date.

## Silver

Common transformations for all tables:
- Rename columns to snake_case.
- Standardize timestamps to UTC where applicable.
- Remove exact duplicates.
- Add:
  - `created_at`
  - `updated_at`
  - `pipeline_run_id`
- Apply Delta OPTIMIZE after writes.
- Preserve source keys.
- Track latest version using `modified_date`.

Deduplication strategy by table:
- `silver.address`
  - Dedup key: `address_id`
  - Keep latest `modified_date`.
- `silver.customer`
  - Dedup key: `customer_id`
  - Keep latest `modified_date`.
  - Exclude `password_hash` and `password_salt` from analytical outputs.
  - Standardize email casing.
- `silver.customer_address`
  - Dedup key: (`customer_id`, `address_id`)
- `silver.product`
  - Dedup key: `product_id`
  - Normalize color and size values.
  - Flag active products based on sell/discontinued dates.
- `silver.product_category`
  - Dedup key: `product_category_id`
- `silver.product_description`
  - Dedup key: `product_description_id`
- `silver.product_model`
  - Dedup key: `product_model_id`
- `silver.product_model_product_description`
  - Dedup key: (`product_model_id`, `product_description_id`, `culture`)
- `silver.sales_order_header`
  - Dedup key: `sales_order_id`
  - Derive:
    - order_year
    - order_month
    - order_date_key
    - shipment_status flag based on ship date
- `silver.sales_order_detail`
  - Dedup key: `sales_order_detail_id`
  - Derive:
    - line_sales_amount = order_qty * unit_price
    - line_discount_amount = order_qty * unit_price * unit_price_discount
    - line_net_sales_amount

Relationship validation in silver:
- `sales_order_header.customer_id` → `customer.customer_id`
- `sales_order_header.ship_to_address_id` → `address.address_id`
- `sales_order_header.bill_to_address_id` → `address.address_id`
- `sales_order_detail.sales_order_id` → `sales_order_header.sales_order_id`
- `sales_order_detail.product_id` → `product.product_id`
- `customer_address.customer_id` → `customer.customer_id`
- `customer_address.address_id` → `address.address_id`
- `product.product_category_id` → `product_category.product_category_id`
- `product.product_model_id` → `product_model.product_model_id`
- `product_model_product_description.product_model_id` → `product_model.product_model_id`
- `product_model_product_description.product_description_id` → `product_description.product_description_id`

## Gold

The source clearly contains transactional sales orders, product master data, customer master data, and address data. The preferred model is a sales star schema.

Dimensions:

- `gold.dim_customer`
  - Grain: one row per customer.
  - Source: customer.
  - Include company, salesperson, email, title, suffix.
  - PK: `customer_id`.

- `gold.dim_address`
  - Grain: one row per address.
  - Source: address.
  - Include city and postal code.
  - PK: `address_id`.

- `gold.dim_product`
  - Grain: one row per product.
  - Source:
    - product
    - product_category
    - product_model
    - product_model_product_description
    - product_description
  - Flatten available category/model/description attributes.
  - PK: `product_id`.

- `gold.dim_product_category`
  - Grain: one row per category.
  - Include parent-child hierarchy using `parent_product_category_id`.
  - PK: `product_category_id`.

- `gold.dim_date`
  - Derived calendar dimension.
  - Support order, due, and ship date analysis.

Facts:

- `gold.fact_sales_order_line`
  - Primary sales fact.
  - Grain: one row per sales order detail.
  - Sources:
    - sales_order_detail
    - sales_order_header
  - Keys:
    - sales_order_id
    - sales_order_detail_id
    - customer_id
    - product_id
    - ship_to_address_id
    - bill_to_address_id
    - order_date_key
  - Measures:
    - order_qty
    - unit_price
    - unit_price_discount
    - line_sales_amount
    - line_discount_amount
    - line_net_sales_amount

- `gold.fact_sales_order_header`
  - Optional aggregate fact.
  - Grain: one row per sales order.
  - Measures:
    - subtotal
    - tax_amt
    - freight
  - Useful for reconciliation against header totals.

Bridge/Junction:

- `gold.bridge_customer_address`
  - Source: customer_address.
  - Maintains many-to-many customer-address relationships.

Alternative modeling option:
- If users primarily require order-level reporting, `fact_sales_order_header` can become the primary fact and detail lines can be secondary.
- Current recommendation is line-level fact because product-level analysis is supported by available relationships.

## Test

All tests append a row to `test.test_results` with:
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
- Layer: bronze vs silver
- Tables:
  - address
  - customer
  - customer_address
  - product
  - product_category
  - product_description
  - product_model
  - product_model_product_description
  - sales_order_header
  - sales_order_detail
- Rule:
  - Silver row count within 1% of bronze after deduplication.

2. No-null dimension primary keys
- Tables:
  - dim_customer.customer_id
  - dim_address.address_id
  - dim_product.product_id
  - dim_product_category.product_category_id
  - dim_date.date_key
- Expected:
  - 0 null keys.

3. Unique dimension primary keys
- Tables:
  - dim_customer
  - dim_address
  - dim_product
  - dim_product_category
  - dim_date
- Expected:
  - No duplicate PK values.

4. Referential integrity
- fact_sales_order_line.customer_id exists in dim_customer
- fact_sales_order_line.product_id exists in dim_product
- fact_sales_order_line.ship_to_address_id exists in dim_address
- fact_sales_order_line.bill_to_address_id exists in dim_address
- fact_sales_order_line.order_date_key exists in dim_date
- Expected:
  - 100% match rate.

5. Business-rule sanity checks
- All `order_qty` values greater than 0.
- All `unit_price` values greater than or equal to 0.
- `line_net_sales_amount` greater than or equal to 0.
- Header monetary values (`subtotal`, `tax_amt`, `freight`) greater than or equal to 0.
- Orders with `ship_date` populated should have `ship_date >= order_date`.

## Semantic model

Mode:
- Direct Lake

Tables:
- dim_date
- dim_customer
- dim_address
- dim_product
- dim_product_category
- fact_sales_order_line
- fact_sales_order_header
- bridge_customer_address

Relationships:
- fact_sales_order_line.customer_id → dim_customer.customer_id
- fact_sales_order_line.product_id → dim_product.product_id
- fact_sales_order_line.ship_to_address_id → dim_address.address_id
- fact_sales_order_line.bill_to_address_id → dim_address.address_id
- fact_sales_order_line.order_date_key → dim_date.date_key
- dim_product.product_category_id → dim_product_category.product_category_id
- fact_sales_order_header.customer_id → dim_customer.customer_id
- bridge_customer_address.customer_id → dim_customer.customer_id
- bridge_customer_address.address_id → dim_address.address_id

Measures:
- Total Sales = SUM(fact_sales_order_line[line_net_sales_amount])
- Gross Sales = SUM(fact_sales_order_line[line_sales_amount])
- Total Discount = SUM(fact_sales_order_line[line_discount_amount])
- Total Quantity = SUM(fact_sales_order_line[order_qty])
- Order Count = DISTINCTCOUNT(fact_sales_order_line[sales_order_id])
- Average Order Value = DIVIDE(SUM(fact_sales_order_header[subtotal]), DISTINCTCOUNT(fact_sales_order_header[sales_order_id]))
- Total Freight = SUM(fact_sales_order_header[freight])
- Total Tax = SUM(fact_sales_order_header[tax_amt])

## Report

### Page 1: Sales Overview
Visuals:
- KPI cards:
  - Total Sales
  - Order Count
  - Total Quantity
  - Average Order Value
- Monthly sales trend line chart.
- Sales by product category bar chart.
- Sales by salesperson bar chart.

### Page 2: Product Performance
Visuals:
- Product sales ranking.
- Product category contribution chart.
- Quantity sold by product.
- Scatter plot:
  - Gross Sales
  - Quantity
  - Product

### Page 3: Customer & Geography
Visuals:
- Sales by customer company.
- Sales by city.
- Map visual using city/postal code.
- Customer count by salesperson.

### Page 4: Order Operations
Visuals:
- Order status distribution.
- Order-to-ship timeline analysis.
- Freight and tax trend analysis.
- Orders by ship method.

### Page 5: Data Quality
Visuals:
- Test pass/fail summary.
- Failed test details table.
- Row count reconciliation dashboard.
- Referential integrity status indicators.
- Latest pipeline execution status.

## Data Agent

Role:
- Sales Analytics Copilot for order, customer, product, and operational sales analysis using the Direct Lake semantic model.

Domain hints:
- Focus on sales orders, products, customers, addresses, shipping activity, freight, tax, and discount analysis.
- Use semantic model measures whenever available.
- Prefer aggregated business insights over raw-row outputs.

Starter questions:
- What are total sales by month?
- Which products generate the highest sales?
- Which customers contribute the most revenue?
- What is the average order value trend?
- Which cities generate the most sales?
- Which salesperson supports the most customer revenue?
- How much discount was given by product?
- What are the top product categories by sales?
- How many orders have shipped versus not shipped?
- What is the freight cost trend over time?

Guardrails:
- Only answer using data exposed through the semantic model.
- Do not expose password-related source fields.
- Do not infer customer information not present in the model.
- Prefer approved measures over ad hoc calculations.
- Clearly indicate when requested data is unavailable.
- Do not generate write-back or data-modification actions.
- Respect row-level security if configured in the semantic model.