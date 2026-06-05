# Run Spec 20260605-091115-6334db

## Updated specs

### Iteration 1 — 2026-06-05 09:20:33Z — failed layer: gold (run: 20260605-091205-f71fda)
- **Root cause (1-line summary)**: Gold-layer failure was surfaced only as a session-cancelled error; likely caused by a downstream gold transformation referencing columns that are not guaranteed to exist after silver snake_case standardization and schema simplification.
- **Cross-table audit**:
  - Address: yes — city/postal_code fields are consumed by dim_customer and must be validated before joins.
  - Customer: yes — sales_person and customer attributes are consumed by dim_customer and dim_salesperson.
  - CustomerAddress: no — explicitly not used in the final customer dimension per requirements.
  - Product: yes — product attributes and discontinued indicators are consumed by dim_product.
  - ProductCategory: yes — category identifiers are consumed by dim_product.
  - ProductDescription: yes — description attributes are consumed by dim_product.
  - ProductModel: yes — source is known to contain only identifiers; gold must not reference a non-existent name column.
  - ProductModelProductDescription: yes — culture filtering and joins are used by dim_product.
  - SalesOrderDetail: yes — fact_sales_order depends on required detail columns.
  - SalesOrderHeader: yes — dimensions and fact tables depend on date, customer, address, and order attributes.
- **Fix approach**: GENERALIZE — the failure signature does not identify a single table/column, so gold-layer schema validation and explicit column contracts are required across all gold builds.
- **What was changed**:
  - Tightened the Gold section with required source-column contracts for every dimension and fact.
  - Added mandatory schema validation before joins, projections, and key generation.
  - Explicitly prohibited references to unavailable columns such as product_model.name and category name fields.

## Inputs
- Workspace: `423348df-cb05-4fa0-bb36-72cf92932692`
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
- Target Lakehouse: **q**

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
- Use defensive column references and validate required columns before every transformation.
- Alias-qualify all join columns inside join projections and immediately rename to flat names.
- Assert column existence before every groupBy/agg.
- For REST/API responses use `if x is None: raise` before any `.get()`.
- Create schemas with `CREATE SCHEMA IF NOT EXISTS`.
- Write all outputs with schema-qualified `saveAsTable('<schema>.<table>')`.
- Never write target lakehouse outputs using raw abfss `.save()`.
- Use parameter cells for workspace, lakehouse, schema, table names, and run_id.
- Use idempotent overwrite patterns with `overwriteSchema=true`.
- Use error-loud try/except blocks that call `_save_error(layer, e)` and re-raise.
- Process source tables independently with per-table error isolation.
- All notebooks must emit discoverable Delta tables in bronze, silver, gold, and test schemas.
- Every code cell must begin with a short explanatory comment block using `# ---`.

### Global Spark column-reference rules (apply to ALL layers: Bronze, Silver, Gold)
Retain all existing Rules A-L exactly as defined in the current specification.

ALSO REQUIRE for every generated notebook: EACH code cell must start with a short markdown comment block (Python `# ---` divider + 1-3 lines of `# ` comments) describing what the cell is doing and why.

## Bronze

Land each source table 1:1 into the `bronze` schema with minimal transformation.

Tables:
- bronze.address
- bronze.customer
- bronze.customeraddress
- bronze.product
- bronze.productcategory
- bronze.productdescription
- bronze.productmodel
- bronze.productmodelproductdescription
- bronze.salesorderdetail
- bronze.salesorderheader

Standard metadata:
- ingestion_ts
- source_table
- run_id
- source_file_or_object
- bronze_load_ts

Write pattern:
- Delta format
- overwrite mode with overwriteSchema=true
- saveAsTable into bronze schema

Partitioning:
- salesorderheader: partition by order year derived from OrderDate
- salesorderdetail: partition by SalesOrderID hash bucket or unpartitioned if volume is small
- Remaining master tables: unpartitioned

Preserve all source columns including:
- rowguid
- ModifiedDate
- binary thumbnail content in Product

## Silver

Common standards:
- Convert all columns to snake_case.
- Trim strings and normalize empty strings to null where appropriate.
- Preserve business keys.
- Add silver_load_ts and source_dt.
- Deduplicate using latest modified_date when available.
- OPTIMIZE and V-ORDER after writes.

Silver tables and deduplication:

- silver.address
  - PK: address_id
  - Dedup key: address_id

- silver.customer
  - PK: customer_id
  - Dedup key: customer_id
  - Create cleaned salesperson_source from sales_person

- silver.customeraddress
  - Composite PK: customer_id, address_id
  - Dedup key: customer_id + address_id

- silver.product
  - PK: product_id
  - Dedup key: product_id
  - Derive is_discontinued from discontinued_date
  - Derive is_active_product

- silver.productcategory
  - PK: product_category_id
  - Dedup key: product_category_id

- silver.productdescription
  - PK: product_description_id
  - Dedup key: product_description_id

- silver.productmodel
  - PK: product_model_id
  - Dedup key: product_model_id
  - NOTE: schema contains only ProductModelID, rowguid, ModifiedDate. Requested ProductModel.Name does not exist in supplied source schema. Gold layer will use ProductModelID only unless an additional name column is later provided.

- silver.productmodelproductdescription
  - Composite PK: product_model_id, product_description_id, culture
  - Filter support retained for gold layer.
  - Dedup key: product_model_id + product_description_id + culture

- silver.salesorderheader
  - PK: sales_order_id
  - Dedup key: sales_order_id

- silver.salesorderdetail
  - PK: sales_order_detail_id
  - Dedup key: sales_order_detail_id

## Gold

Target star schema aligned to requested sales-reporting solution.

Mandatory gold-layer schema validation:
- Before building each dimension or fact, assert that every required source column exists on the silver DataFrame being used.
- Fail fast with an explicit message naming the missing column, source table, and target gold table.
- After every join projection, reference only projected flat column names.
- Do not reference any column not explicitly listed below.
- Do not reference product_model.name, product_category.name, parent_category_name, or any other descriptive category/model columns not present in the supplied source schema.

Dimensions:

- gold.dim_order_date
  - Source: silver.salesorderheader
  - Required columns:
    - order_date
  - Key: date_key

- gold.dim_ship_date
  - Source: silver.salesorderheader
  - Required columns:
    - ship_date
  - Key: ship_date_key

- gold.dim_customer
  - Sources:
    - silver.customer
    - silver.salesorderheader
    - silver.address
  - Required columns:
    - customer.customer_id
    - customer.company_name
    - customer.title
    - customer.suffix
    - customer.email_address
    - salesorderheader.customer_id
    - salesorderheader.bill_to_address_id
    - address.address_id
    - address.city
    - address.postal_code
  - Join path:
    - salesorderheader.customer_id = customer.customer_id
    - salesorderheader.bill_to_address_id = address.address_id
  - Do not use customeraddress bridge.
  - Project and immediately rename joined fields to flat names before downstream use.

- gold.dim_salesperson
  - Source: silver.customer
  - Required columns:
    - customer_id
    - sales_person OR salesperson_source
  - If salesperson_source exists, use it.
  - If sales_person exists, derive salesperson_source from it.
  - Do not assume both columns exist simultaneously.

- gold.dim_order
  - Source: silver.salesorderheader
  - Required columns:
    - sales_order_id
    - revision_number
    - status
    - ship_method
    - comment

- gold.dim_product
  - Sources:
    - silver.product
    - silver.productcategory
    - silver.productmodelproductdescription
    - silver.productdescription
    - silver.productmodel
  - Required columns:
    - product.product_id
    - product.product_category_id
    - product.product_model_id
    - product.product_number
    - product.color
    - product.size
    - product.weight
    - product.standard_cost
    - product.list_price
    - product.is_discontinued
    - product.is_active_product
    - productcategory.product_category_id
    - productcategory.parent_product_category_id
    - productmodel.product_model_id
    - productmodelproductdescription.product_model_id
    - productmodelproductdescription.product_description_id
    - productmodelproductdescription.culture
    - productdescription.product_description_id
  - Filter productmodelproductdescription to culture='en'.
  - Use only identifiers from productmodel and productcategory.
  - Do not create references to non-existent model or category name columns.

Fact:

- gold.fact_sales_order
  - Grain: one sales order line item
  - Sources:
    - silver.salesorderheader
    - silver.salesorderdetail
  - Required header columns:
    - sales_order_id
    - customer_id
    - order_date
    - ship_date
    - subtotal
    - tax_amt
    - freight
  - Required detail columns:
    - sales_order_id
    - product_id
    - order_qty
    - unit_price
    - unit_price_discount
  - Join:
    - salesorderheader.sales_order_id = salesorderdetail.sales_order_id
  - Validate required columns before join and before measure calculations.
  - Foreign keys:
    - order_date_key
    - ship_date_key
    - customer_key
    - salesperson_key
    - order_key
    - product_key
  - Measures retained:
    - extended_amount = order_qty * unit_price
    - discount_amount = order_qty * unit_price * unit_price_discount
    - net_sales_amount = order_qty * unit_price * (1 - unit_price_discount)

## Test

Write all test outcomes to:
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

Required tests:

1. Row Count Reconciliation
- Compare bronze vs silver row counts per table.
- PASS if variance <= 1%.

2. Gold Dimension PK Not Null
- dim_customer.customer_id
- dim_salesperson.salesperson_key
- dim_product.product_id
- dim_order.sales_order_id
- dim_order_date.date_key
- dim_ship_date.ship_date_key

3. Gold Dimension PK Uniqueness
- Verify uniqueness of all dimension primary keys.

4. Referential Integrity
- fact_sales_order.product_key exists in dim_product
- fact_sales_order.customer_key exists in dim_customer
- fact_sales_order.salesperson_key exists in dim_salesperson
- fact_sales_order.order_key exists in dim_order
- fact_sales_order.order_date_key exists in dim_order_date
- fact_sales_order.ship_date_key exists in dim_ship_date

5. Business Rule Sanity Checks
- net_sales_amount >= 0
- discount_amount >= 0
- unit_price >= 0
- order_qty > 0
- average discount percentage between 0 and 100
- monthly sales totals should not contain null month assignments

## Semantic model

Mode:
- Direct Lake

Model tables:
- dim_order_date
- dim_ship_date
- dim_customer
- dim_salesperson
- dim_order
- dim_product
- fact_sales_order

Relationships:
- fact_sales_order → dim_order_date
- fact_sales_order → dim_ship_date
- fact_sales_order → dim_customer
- fact_sales_order → dim_salesperson
- fact_sales_order → dim_order
- fact_sales_order → dim_product

Hierarchies, measures, and relationships remain as specified.

## Report

All report requirements remain as specified.

## Data Agent

All agent requirements, starter questions, and guardrails remain as specified.