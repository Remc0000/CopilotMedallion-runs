# Run Spec 20260604-102523-3841e4

## Updated specs

### Iteration 1 — 2026-06-04 10:32:22Z — failed layer: bronze (run: 20260604-102614-505b55)
- **Root cause (1-line summary)**: Bronze completed without producing discoverable Delta tables under `Tables/bronze/`, so Silver could not start.
- **Cross-table audit**: Address: yes — output path discoverability applies; Customer: yes — output path discoverability applies; CustomerAddress: yes — output path discoverability applies; Product: yes — output path discoverability applies; ProductCategory: yes — output path discoverability applies; ProductDescription: yes — output path discoverability applies; ProductModel: yes — output path discoverability applies; ProductModelProductDescription: yes — output path discoverability applies; SalesOrderDetail: yes — output path discoverability applies; SalesOrderHeader: yes — output path discoverability applies.
- **Fix approach**: GENERALIZE — the failure is systemic and can affect every Bronze source table equally because discoverability depends on write location and registration behavior, not table-specific schema.
- **What was changed**:
  - Tightened Bronze output requirements to mandate one Delta write per source table under `Tables/bronze/<exact_lowercase_table_name>`.
  - Added explicit mapping between source tables and required Bronze folder names.
  - Required Bronze to fail if fewer than 10 Bronze Delta outputs are written and discoverable.

### Iteration 2 — 2026-06-04 10:39:44Z — failed layer: bronze (run: 20260604-102614-505b55)
- **Root cause (1-line summary)**: Silver could not discover any Bronze tables because Bronze outputs were not materialized as readable Delta tables under the required Lakehouse `Tables/bronze/*` locations.
- **Cross-table audit**: Address: yes — discoverability depends on physical Delta creation; Customer: yes — same; CustomerAddress: yes — same; Product: yes — same; ProductCategory: yes — same; ProductDescription: yes — same; ProductModel: yes — same; ProductModelProductDescription: yes — same; SalesOrderDetail: yes — same; SalesOrderHeader: yes — same.
- **Fix approach**: GENERALIZE — the issue is location/materialization related and applies uniformly to every Bronze source table.
- **What was changed**:
  - Added mandatory post-write validation using Delta reads from every required Bronze path.
  - Required Bronze completion to verify exactly 10 readable Delta outputs before reporting success.
  - Prohibited notebook success when any source table is skipped, empty due to read failure, or missing from the final validation manifest.

### Iteration 3 — 2026-06-04 10:41:45Z — failed layer: bronze (run: 20260604-102614-505b55)
- **Root cause (1-line summary)**: Build was interrupted by a server restart during Bronze execution, creating risk of partial writes and non-resumable state.
- **Cross-table audit**: Address: yes — interruption can occur mid-write; Customer: yes — same; CustomerAddress: yes — same; Product: yes — same; ProductCategory: yes — same; ProductDescription: yes — same; ProductModel: yes — same; ProductModelProductDescription: yes — same; SalesOrderDetail: yes — same; SalesOrderHeader: yes — same.
- **Fix approach**: GENERALIZE — restart/interruption handling must be applied consistently to every Bronze table.
- **What was changed**:
  - Added resumable Bronze processing rules that validate existing Delta outputs before reprocessing.
  - Required per-table write/validation completion tracking so already-valid tables are not treated as failed after a restart.
  - Added final reconciliation that confirms all 10 required Bronze outputs are readable regardless of whether they were written in the current session or a resumed session.

## Inputs
- Workspace: `d547615a-511c-436c-b6e0-c95f688a7ead`
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
- Target Lakehouse: **a**

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
- Use defensive column references and validate required columns before every join, filter, aggregation, window, and derived-column operation.
- Apply alias-prefixed joins only within the join statement; materialize flat column names immediately after joins.
- Assert groupBy and aggregation columns exist before execution.
- For REST/API responses use `if x is None: raise RuntimeError(...)` before any `.get(...)`.
- Do not use `saveAsTable`; write Delta directly to Lakehouse paths.
- All notebooks must begin with parameter cells for workspace, lakehouse, paths, run_id, and layer.
- Use idempotent overwrite patterns with `mode("overwrite")` and `overwriteSchema=true`.
- Use explicit try/except blocks that call `_save_error(layer, e)` and re-raise after logging.
- Process source tables independently and persist outputs per table.
- Emit discoverable Delta outputs under Tables/bronze, Tables/silver, Tables/gold, and Tables/test.
- Every notebook cell must start with a short comment block describing intent.

### Global Spark column-reference rules (apply to ALL layers: Bronze, Silver, Gold)
- Retain all existing rules A–K from the current specification without modification.

## Bronze

Land each source table unchanged into `Tables/bronze/<table_name_lower>`.

Source-to-bronze tables and REQUIRED output folder names:
- `SalesLT/Address` → `Tables/bronze/address`
- `SalesLT/Customer` → `Tables/bronze/customer`
- `SalesLT/CustomerAddress` → `Tables/bronze/customeraddress`
- `SalesLT/Product` → `Tables/bronze/product`
- `SalesLT/ProductCategory` → `Tables/bronze/productcategory`
- `SalesLT/ProductDescription` → `Tables/bronze/productdescription`
- `SalesLT/ProductModel` → `Tables/bronze/productmodel`
- `SalesLT/ProductModelProductDescription` → `Tables/bronze/productmodelproductdescription`
- `SalesLT/SalesOrderDetail` → `Tables/bronze/salesorderdetail`
- `SalesLT/SalesOrderHeader` → `Tables/bronze/salesorderheader`

Bronze standards:
- Preserve source schema and datatypes.
- Add metadata columns:
  - `_run_id`
  - `_ingested_at`
  - `_source_table`
  - `_bronze_ts`
- Overwrite write pattern with schema evolution enabled.
- Partition large transactional tables:
  - salesorderheader: partition by order year derived from OrderDate.
  - salesorderdetail: partition by SalesOrderID hash bucket or order year if available after enrichment.
- Small master tables remain unpartitioned.
- Persist all tables as Delta format.
- Capture row counts in notebook summary output.
- After each write, validate the target path exists and contains a readable Delta table.
- Immediately after writing each table, perform `spark.read.format("delta").load(<target_path>)` and verify the read succeeds and returns the expected row count.
- Maintain a results collection containing table name, row count, and output path for every successful write.
- Print a final JSON summary listing all successfully written Bronze tables.
- Build a final validation manifest containing all 10 required Bronze table names and paths.
- Before processing a source table, check whether the required target path already contains a readable Delta table; if it does, capture its row count and treat it as a valid completed table for resume/rebuild scenarios.
- Processing must be table-by-table with validation immediately after each table so a server restart cannot invalidate previously completed tables.
- On rebuild after interruption, revalidate existing Bronze outputs and only rewrite tables that are missing or unreadable.
- Raise an error if fewer than 10 Bronze tables are successfully written and discoverable.
- Raise an error if any required Bronze output path listed above is missing at notebook completion.
- Raise an error if the final validation manifest does not contain exactly these 10 table names: address, customer, customeraddress, product, productcategory, productdescription, productmodel, productmodelproductdescription, salesorderdetail, salesorderheader.
- Bronze success is defined as: all 10 source tables written as readable Delta outputs under their required `Tables/bronze/<name>` locations and successfully re-read from those locations during notebook validation.

## Silver

Standardize all tables:
- Rename columns to snake_case.
- Add:
  - `_silver_ts`
  - `_source_dt`
  - `_is_current`
- Remove duplicate records using latest modified_date where available.
- Retain rowguid only where useful for lineage.

Deduplication keys:
- address: address_id
- customer: customer_id
- customeraddress: (customer_id, address_id)
- product: product_id
- productcategory: product_category_id
- productdescription: product_description_id
- productmodel: product_model_id
- productmodelproductdescription: (product_model_id, product_description_id, culture)
- salesorderheader: sales_order_id
- salesorderdetail: sales_order_detail_id

Silver business preparation:
- Customer:
  - Standardize email casing.
  - Trim text fields.
- Address:
  - Normalize city values.
- Product:
  - Derive is_discontinued from discontinued_date.
  - Derive is_active_product.
- SalesOrderHeader:
  - Derive order_year, order_month, order_date_key.
  - Derive ship_date_key when ship_date exists.
- SalesOrderDetail:
  - Derive line_discount_amount = order_qty * unit_price * unit_price_discount.
  - Derive gross_line_amount = order_qty * unit_price.
  - Derive net_line_amount = order_qty * unit_price * (1 - unit_price_discount).

Performance:
- OPTIMIZE silver transactional tables.
- ZORDER:
  - sales_order_id
  - customer_id
  - product_id

## Gold

User-requested target schema.

Dimension: dim_order_date
- Source: sales_order_header.order_date.
- Grain: one row per calendar date.

Dimension: dim_ship_date
- Source: sales_order_header.ship_date.
- Grain: one row per calendar date.

Dimension: dim_customer
- Build primarily from Customer.
- Include:
  - customer_id
  - company_name
  - title
  - suffix
  - email_address

Dimension: dim_salesperson
- Source: customer.sales_person.

Dimension: dim_order
- Source: sales_order_header.

Dimension: dim_product
- Source:
  - product
  - productcategory
  - productmodel
  - productmodelproductdescription
  - productdescription
- Filter ProductModelProductDescription to culture='en'.

Fact: fact_sales_order
- Grain: one row per sales order detail line.
- Join sales_order_detail to sales_order_header on sales_order_id.

## Test

Write all results to:
- `Tables/test/test_results`

Required tests:
1. Row count reconciliation.
2. Gold dimension PK not null.
3. Gold dimension PK uniqueness.
4. Referential integrity.
5. Business-rule sanity checks.

## Semantic model

Mode:
- Direct Lake

Tables:
- Fact Sales Order
- Customer
- Product
- SalesPerson
- Order
- OrderDate
- ShipDate

Relationships:
- Fact Sales Order → Customer
- Fact Sales Order → Product
- Fact Sales Order → SalesPerson
- Fact Sales Order → Order
- Fact Sales Order → OrderDate
- Fact Sales Order → ShipDate

## Report

Page 1: Executive Sales Overview
- KPI cards.
- Monthly sales trend.
- Category analysis.
- Top products.

Page 2: Regional Performance
- Regional visuals only when valid geography enrichment is available through approved modeling.

Page 3: Salesperson Discount Analysis

Page 4: Orders and Fulfillment

Page 5: Data Quality

## Data Agent

Role:
- Enterprise Sales Performance Analyst for the SalesLT reporting platform.

Guardrails:
- Do not invent geographic relationships not present in the model.
- Do not infer customer addresses unless they exist in the published dimensions.
- Clearly state when requested attributes are unavailable.
- Do not expose PasswordHash or PasswordSalt fields.
- Use only semantic-model entities and measures.