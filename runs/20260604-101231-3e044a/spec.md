# Run Spec 20260604-100109-de5088

## Updated specs

### Iteration 1 — 2026-06-04 10:18:30Z — failed layer: bronze (run: 20260604-101231-3e044a)
- **Root cause (1-line summary)**: Bronze completed without producing discoverable Delta tables under `Tables/bronze/`, causing Silver generation to stop.
- **Cross-table audit**:
  - SalesLT/Address: yes — Bronze output path/discoverability rules apply.
  - SalesLT/Customer: yes — Bronze output path/discoverability rules apply.
  - SalesLT/CustomerAddress: yes — Bronze output path/discoverability rules apply.
  - SalesLT/Product: yes — Bronze output path/discoverability rules apply.
  - SalesLT/ProductCategory: yes — Bronze output path/discoverability rules apply.
  - SalesLT/ProductDescription: yes — Bronze output path/discoverability rules apply.
  - SalesLT/ProductModel: yes — Bronze output path/discoverability rules apply.
  - SalesLT/ProductModelProductDescription: yes — Bronze output path/discoverability rules apply.
  - SalesLT/SalesOrderDetail: yes — Bronze output path/discoverability rules apply.
  - SalesLT/SalesOrderHeader: yes — Bronze output path/discoverability rules apply.
- **Fix approach**: GENERALIZE — the failure is not table-specific; every Bronze source table must be written to a discoverable Delta location using a deterministic naming convention and validated after write.
- **What was changed**:
  - Tightened Bronze output-path requirements and table-name mapping.
  - Added mandatory post-write validation that each Delta path exists and contains data.
  - Added explicit build-failure conditions if any required Bronze table is missing or if zero discoverable tables are produced.

### Iteration 2 — 2026-06-04 10:22:56Z — failed layer: bronze (run: 20260604-101231-3e044a)
- **Root cause (1-line summary)**: Build was interrupted by a server restart mid-Bronze execution, creating a risk of partially written or partially validated Bronze outputs.
- **Cross-table audit**:
  - SalesLT/Address: yes — interruption could occur before or after write completion.
  - SalesLT/Customer: yes — interruption could occur before or after write completion.
  - SalesLT/CustomerAddress: yes — interruption could occur before or after write completion.
  - SalesLT/Product: yes — interruption could occur before or after write completion.
  - SalesLT/ProductCategory: yes — interruption could occur before or after write completion.
  - SalesLT/ProductDescription: yes — interruption could occur before or after write completion.
  - SalesLT/ProductModel: yes — interruption could occur before or after write completion.
  - SalesLT/ProductModelProductDescription: yes — interruption could occur before or after write completion.
  - SalesLT/SalesOrderDetail: yes — interruption could occur before or after write completion.
  - SalesLT/SalesOrderHeader: yes — interruption could occur before or after write completion.
- **Fix approach**: GENERALIZE — restart resilience and idempotent validation requirements apply uniformly to every Bronze table.
- **What was changed**:
  - Added mandatory restart-safe, idempotent Bronze write behavior.
  - Added pre-run validation of existing Bronze outputs and selective rewrites for missing or unreadable tables.
  - Added completion manifest requirements so resumed builds can determine which Bronze tables are already valid.

## Inputs
- Workspace: `1779f71c-1dd7-4707-af35-94419229e9ac`
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
- Target Lakehouse: **SalesAnalytics**

## Generic guidance

Apply these reference skills/agents at all times:
- FabricDataEngineer agent: https://github.com/microsoft/skills-for-fabric/blob/main/agents/FabricDataEngineer.agent.md
- e2e-medallion-architecture skill: https://github.com/microsoft/skills-for-fabric/tree/main/skills/e2e-medallion-architecture
- spark-authoring-cli skill: https://github.com/microsoft/skills-for-fabric/tree/main/skills/spark-authoring-cli
- powerbi-authoring-cli skill: https://github.com/microsoft/skills-for-fabric/tree/main/skills/powerbi-authoring-cli
- powerbi-consumption-cli skill: https://github.com/microsoft/skills-for-fabric/tree/main/skills/powerbi-consumption-cli
- powerbi-semantic-model-authoring: https://github.com/RuiRomano/powerbi-agentic-plugins/tree/main/plugins/powerbi/skills/powerbi-semantic-model-authoring
- powerbi-report-authoring: https://github.com/RuiRomano/powerbi-agentic-plugins/tree/main/plugins/powerbi/skills/powerbi-report-authoring

All existing Generic guidance rules remain in force, including Rule K (Resilience to partial output). In addition:

- Treat discoverability as a required acceptance criterion for every layer output.
- After every write, immediately validate that the target Delta path exists and can be read back successfully.
- Record the exact output path written for each table and include it in the final results summary.
- A layer is considered successful only if all required output tables for that layer are discoverable at their expected locations.
- All writes must be idempotent and safe to rerun after notebook interruption, cluster restart, or server restart.
- When resuming a build, validate existing outputs before rewriting them; do not assume prior completion based solely on notebook execution history.

## Bronze

Land each source table 1:1 into `Tables/bronze/<table_name>`.

For every table:
- Preserve source schema exactly.
- Add metadata columns:
  - `_run_id`
  - `_ingested_at`
  - `_source_table`
  - `_bronze_ts`
- Write as Delta overwrite with schema evolution enabled.

Mandatory output-path mapping (exact names):
- `SalesLT/Address` → `Tables/bronze/address`
- `SalesLT/Customer` → `Tables/bronze/customer`
- `SalesLT/CustomerAddress` → `Tables/bronze/customeraddress`
- `SalesLT/Product` → `Tables/bronze/product`
- `SalesLT/ProductCategory` → `Tables/bronze/productcategory`
- `SalesLT/ProductDescription` → `Tables/bronze/productdescription`
- `SalesLT/ProductModel` → `Tables/bronze/productmodel`
- `SalesLT/ProductModelProductDescription` → `Tables/bronze/productmodelproductdescription`
- `SalesLT/SalesOrderHeader` → `Tables/bronze/salesorderheader`
- `SalesLT/SalesOrderDetail` → `Tables/bronze/salesorderdetail`

Mandatory write and validation requirements:
- Write each source table independently in a loop.
- Use Delta format and overwrite mode.
- Before writing a table, check whether the target Delta path already exists and is readable.
- If a readable Delta table exists and passes validation (schema readable and row count obtainable), it may be reused during a resumed run instead of being rewritten.
- If a target path is missing, unreadable, or fails validation, rewrite that table from source.
- After writing each table:
  - Read the Delta path back immediately.
  - Verify row count > 0 unless the source itself is empty.
  - Record `{table_name, rows_written, output_path}` in results.
- Maintain a completion manifest containing one entry per successfully validated Bronze table.
- At notebook completion:
  - Assert all 10 expected Bronze tables exist at the exact paths listed above.
  - Verify all 10 tables are readable as Delta tables.
  - Print a JSON summary of all written or reused tables and paths.
  - Raise an error if any expected Bronze table is missing.
  - Raise an error if fewer than 10 discoverable Bronze tables were produced.
  - Raise an error if the completion manifest does not contain all 10 required Bronze tables.

Partition large transactional tables by year extracted from business date:
- salesorderheader: orderdate year
- salesorderdetail: salesorderid hash bucket or unpartitioned if volume is small

Dimension-style tables may remain unpartitioned.

Bronze outputs:
- bronze.address
- bronze.customer
- bronze.customeraddress
- bronze.product
- bronze.productcategory
- bronze.productdescription
- bronze.productmodel
- bronze.productmodelproductdescription
- bronze.salesorderheader
- bronze.salesorderdetail

## Silver

Standardize all tables:
- Convert column names to snake_case.
- Add `_silver_ts`, `_source_dt`, `_run_id`.
- Trim strings and normalize empty strings to null where appropriate.
- Remove exact duplicate rows.
- Preserve rowguid for lineage but exclude from Gold unless needed.

Deduplication keys:
- address: `address_id`
- customer: `customer_id`
- customeraddress: (`customer_id`, `address_id`)
- product: `product_id`
- productcategory: `product_category_id`
- productdescription: `product_description_id`
- productmodel: `product_model_id`
- productmodelproductdescription: (`product_model_id`, `product_description_id`, `culture`)
- salesorderheader: `sales_order_id`
- salesorderdetail: (`sales_order_id`, `sales_order_detail_id`)

Business standardization:
- Derive `sales_person_username` from `sales_person`.
- When value contains `\`, retain only username portion and remove numeric suffix when present (example: `adventure-works\jillian0` → `jillian`).
- Create product lifecycle flags:
  - `is_discontinued`
  - `is_currently_sellable`
- Create order timing metrics:
  - `days_to_ship`
  - `days_until_due`

NOTE:
- User requested ProductModel Name as model name, but ProductModel schema only contains `product_model_id`, `rowguid`, and `modified_date`.
- Gold product dimension will therefore not contain a model name unless the source schema is expanded. Retain `product_model_id` as the model reference.

Optimize:
- OPTIMIZE silver.salesorderheader
- OPTIMIZE silver.salesorderdetail
- OPTIMIZE major Gold source dimensions after write

## Gold

Target star schema required by user.

Dimensions

1. dim_order_date
- Source: salesorderheader.order_date
- Surrogate key: order_date_key

2. dim_ship_date
- Source: salesorderheader.ship_date
- Surrogate key: ship_date_key

3. dim_customer
- Source: customer + address
- User explicitly requested direct combination without CustomerAddress.

4. dim_sales_person
- Source: customer.sales_person

5. dim_order
- Source: salesorderheader

6. dim_product
- Source:
  - product
  - productcategory
  - productmodel
  - productmodelproductdescription (culture='en')
  - productdescription

Fact

fact_sales_order
- Grain: one row per SalesOrderDetail line.
- Join: salesorderdetail.sales_order_id = salesorderheader.sales_order_id.

## Test

All tests append results to:
- gold.test_results

Required tests:
1. Row Count Reconciliation
2. Gold Dimension PK Not Null
3. Gold Dimension PK Uniqueness
4. Referential Integrity
5. Business Rule Validation

## Semantic model

Mode:
- Direct Lake

Tables:
- Fact Sales Order
- Customer
- Product
- Sales Person
- Order
- Order Date
- Ship Date

## Report

Page 1 — Executive Sales Overview
Page 2 — Regional Performance
Page 3 — Discount & Salesperson Analysis
Page 4 — Orders & Fulfillment
Page 5 — Data Quality

## Data Agent

Role:
- AI Sales Performance Analyst for the SalesAnalytics semantic model.

Guardrails:
- Answer only using the semantic model.
- Do not fabricate geography beyond city and postal code.
- Clearly state when requested information is unavailable.
- Never invent missing ProductModel names because the source schema does not contain them.