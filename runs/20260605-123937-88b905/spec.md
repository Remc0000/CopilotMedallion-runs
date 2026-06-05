# Run Spec 20260605-123906-ec8a95

## Updated specs

### Iteration 1 — 2026-06-05 12:43:10Z — failed layer: silver (run: 20260605-123937-88b905)
- **Root cause (1-line summary)**: Silver-layer processing allowed a table-level failure to escalate into a Spark session cancellation, preventing remaining tables from completing.
- **Cross-table audit**:
  - Address: yes — any transformation error could terminate the shared Silver run.
  - Customer: yes — contains multiple derived-column cleansing steps that could fail independently.
  - CustomerAddress: yes — junction-table processing can fail and should not stop other tables.
  - Product: yes — derived flags and date logic increase transformation risk.
  - ProductCategory: yes — naming/flattening logic could fail independently.
  - ProductDescription: yes — schema normalization could fail independently.
  - ProductModel: yes — schema normalization could fail independently.
  - ProductModelProductDescription: yes — culture filtering and bridge-table logic could fail independently.
  - SalesOrderDetail: yes — calculated measures could fail independently.
  - SalesOrderHeader: yes — date validation logic could fail independently.
- **Fix approach**: GENERALIZE — the failure pattern is systemic across all Silver tables because any single-table exception can cancel the session and prevent downstream processing.
- **What was changed**:
  - Tightened Silver processing to require strict per-table isolation and independent error handling.
  - Required writing successful Silver tables even when another Silver table fails.
  - Added mandatory schema validation and test-result logging per table before expensive transformations.

### Iteration 2 — 2026-06-05 12:47:26Z — failed layer: gold (run: 20260605-123937-88b905)
- **Root cause (1-line summary)**: A Gold-layer statement failure cancelled the Spark session because Gold entities were not required to be built and validated independently.
- **Cross-table audit**:
  - Address: yes — contributes to dim_customer and missing/invalid columns could fail a dimension build.
  - Customer: yes — contributes to dim_customer and dim_salesperson.
  - CustomerAddress: no — not used in Gold by design.
  - Product: yes — contributes to dim_product and fact joins.
  - ProductCategory: yes — contributes to dim_product.
  - ProductDescription: yes — contributes to dim_product.
  - ProductModel: yes — contributes to dim_product and has known schema limitations.
  - ProductModelProductDescription: yes — contributes to dim_product bridge logic.
  - SalesOrderDetail: yes — contributes to fact_sales_order.
  - SalesOrderHeader: yes — contributes to all date, order, and customer-related dimensions plus fact joins.
- **Fix approach**: GENERALIZE — the session-cancellation pattern can affect every Gold dimension and fact, so all Gold entities must follow the same isolation, validation, and write-verification pattern.
- **What was changed**:
  - Added mandatory per-entity isolation and error handling for all Gold dimensions and facts.
  - Required schema and join-key validation before every Gold build.
  - Required immediate post-write validation and continuation of remaining Gold builds when one entity fails.

### Iteration 3 — 2026-06-05 12:50:05Z — failed layer: gold (run: 20260605-123937-88b905)
- **Root cause (1-line summary)**: PySpark `CANNOT_DETERMINE_TYPE` when creating the test-results DataFrame from in-memory rows containing null/empty values and no explicit schema.
- **Cross-table audit**:
  - Address: yes — any test row referencing this table can contain nullable fields and trigger schema inference issues.
  - Customer: yes — same test-results logging pattern applies.
  - CustomerAddress: yes — same test-results logging pattern applies.
  - Product: yes — same test-results logging pattern applies.
  - ProductCategory: yes — same test-results logging pattern applies.
  - ProductDescription: yes — same test-results logging pattern applies.
  - ProductModel: yes — same test-results logging pattern applies.
  - ProductModelProductDescription: yes — same test-results logging pattern applies.
  - SalesOrderDetail: yes — same test-results logging pattern applies.
  - SalesOrderHeader: yes — same test-results logging pattern applies.
- **Fix approach**: GENERALIZE — the root cause is not table-specific; any Gold validation or test record can fail if Spark must infer types from nullable values.
- **What was changed**:
  - Tightened Gold test logging requirements to require an explicit schema for all test-results DataFrames.
  - Required `checked_at` to be created as a typed timestamp column rather than inferred from Python `None`.
  - Required all test-result fields to be cast to stable types before writing to `test.test_results`.

## Inputs
- Workspace: `db12fd40-6fa7-4998-821c-6ce8e3590ad0`
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
- Target Lakehouse: **c**

## Generic guidance

Apply these reference skills/agents at all times:
- FabricDataEngineer agent: https://github.com/microsoft/skills-for-fabric/blob/main/agents/FabricDataEngineer.agent.md
- e2e-medallion-architecture skill: https://github.com/microsoft/skills-for-fabric/tree/main/skills/e2e-medallion-architecture
- spark-authoring-cli skill: https://github.com/microsoft/skills-for-fabric/tree/main/skills/spark-authoring-cli
- powerbi-authoring-cli skill: https://github.com/microsoft/skills-for-fabric/tree/main/skills/powerbi-authoring-cli
- powerbi-consumption-cli skill: https://github.com/microsoft/skills-for-fabric/tree/main/skills/powerbi-consumption-cli
- powerbi-semantic-model-authoring: https://github.com/RuiRomano/powerbi-agentic-plugins/tree/main/plugins/powerbi/skills/powerbi-semantic-model-authoring
- powerbi-report-authoring: https://github.com/RuiRomano/powerbi-agentic-plugins/tree/main/plugins/powerbi-report-authoring

Cross-cutting code rules:
- Use defensive column references and validate column existence before every join, filter, aggregation, window, and derived-column expression.
- Alias-qualify columns inside join projections and rename immediately after joins.
- Assert required columns exist before every groupBy/agg.
- For REST/API calls use defensive handling: `if x is None: raise` before any `.get()` access.
- Create schemas with `CREATE SCHEMA IF NOT EXISTS` and write only through `saveAsTable('<schema>.<table>')`.
- Never write target layer outputs through raw abfss `.save()` paths.
- Use parameter cells at notebook start for workspace, lakehouse, run_id, source table, and load mode.
- Use idempotent overwrite patterns with `overwriteSchema=true`.
- Use error-loud try/except patterns that call `_save_error(layer, e)` and re-raise.
- Process source tables independently in loops to avoid session-wide failures.
- Every notebook cell must begin with a short comment header block using `# ---` and human-readable comments describing purpose.

### Global Spark column-reference rules (apply to ALL layers: Bronze, Silver, Gold)
These rules exist to prevent recurring `UNRESOLVED_COLUMN` / `AnalysisException` analyzer errors. They are layer-agnostic — apply them anywhere a Spark DataFrame is transformed.

Rule A — No dotted alias strings.

Rule B — Materialize helper columns before they are needed downstream.

Rule C — Do not drop a column before its last consumer has run.

Rule D — Order of derived-column computations matters.

Rule E — Validate schema between non-trivial transformation steps.

Rule F — Self-check pattern for every withColumn / Window.

Rule G — Optional-column helpers must return typed Column nulls, not Python None.

Rule H — Per-table isolation; one table's failure must not cancel the Spark session for the rest.

Rule I — Optional audit columns on junction / bridge / view tables.

Rule J — Validate column existence BEFORE the expensive transform.

Rule K — Resilience to partial output: every layer MUST write Delta tables the next layer can discover.

Rule L — Disambiguate shared columns in join projections (avoid AMBIGUOUS_REFERENCE).

## Bronze

Landing strategy:
- Create schema `bronze`.
- Ingest each source table 1:1 with original structure preserved.
- Add metadata columns:
  - `_run_id`
  - `_ingested_at`
  - `_source_table`
  - `_bronze_ts`
- Write as Delta tables using overwrite mode and overwriteSchema=true.

Bronze tables:
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

Partitioning:
- salesorderheader: partition by year derived from OrderDate.
- salesorderdetail: partition by SalesOrderID hash bucket or unpartitioned if volume is small.
- Remaining master-data tables: unpartitioned.

Primary business keys observed:
- Address: AddressID
- Customer: CustomerID
- CustomerAddress: CustomerID + AddressID
- Product: ProductID
- ProductCategory: ProductCategoryID
- ProductDescription: ProductDescriptionID
- ProductModel: ProductModelID
- ProductModelProductDescription: ProductModelID + ProductDescriptionID + Culture
- SalesOrderHeader: SalesOrderID
- SalesOrderDetail: SalesOrderID + SalesOrderDetailID

## Silver

Common transformations:
- Convert all column names to snake_case.
- Preserve business keys.
- Standardize timestamps.
- Remove duplicate rows.
- Add:
  - _silver_ts
  - _source_modified_date
  - _is_current

Mandatory execution pattern for ALL Silver tables:
- Process each Silver table in its own try/except block.
- A failure in one Silver table must be logged and recorded in `test.test_results` but must NOT stop processing of remaining Silver tables.
- Never reuse a partially failed DataFrame for another table.
- Before any transformation, assert the expected business-key columns for that table exist in the Silver source DataFrame.
- After writing each Silver table, immediately verify the Delta table exists and is readable before moving to the next table.
- Build a per-table success/failure summary and only raise a final exception after all Silver tables have been attempted.

Deduplication keys:
- silver.address → address_id
- silver.customer → customer_id
- silver.customeraddress → customer_id + address_id
- silver.product → product_id
- silver.productcategory → product_category_id
- silver.productdescription → product_description_id
- silver.productmodel → product_model_id
- silver.productmodelproductdescription → product_model_id + product_description_id + culture
- silver.salesorderheader → sales_order_id
- silver.salesorderdetail → sales_order_id + sales_order_detail_id

Business cleansing:
- Customer:
  - Trim company_name, email_address, sales_person.
  - Exclude password_hash and password_salt from downstream gold models.
- Product:
  - Create is_discontinued flag from discontinued_date.
  - Create is_active_product based on sell dates and discontinuation.
- SalesOrderHeader:
  - Validate order_date <= due_date where both exist.
- SalesOrderDetail:
  - Create line_discount_amount = order_qty * unit_price * unit_price_discount.
  - Create line_gross_amount = order_qty * unit_price.
  - Create line_net_amount = order_qty * unit_price * (1 - unit_price_discount).

Performance:
- OPTIMIZE silver.salesorderheader
- OPTIMIZE silver.salesorderdetail
- OPTIMIZE major gold source dimensions after write

User-request note:
- User requested Product dimension include ProductModel Name. The provided ProductModel schema contains only ProductModelID, rowguid, and ModifiedDate. No Name column exists. Gold Product dimension will therefore include ProductModelID and English description linkage, but model name cannot be populated unless an additional source is supplied.

## Gold

Create schema `gold`.

Mandatory execution pattern for ALL Gold entities:
- Build each dimension and fact table in its own isolated try/except block.
- A failure in one Gold entity must be logged to `test.test_results` but must NOT stop remaining Gold entities from being attempted.
- Before building an entity, assert that all required Silver source tables exist and are readable.
- Before every join, explicitly validate the required join keys exist in both inputs.
- Alias all joined DataFrames and project only alias-qualified columns.
- After writing each Gold table, immediately verify the Delta table exists and can be read back successfully.
- Maintain a per-entity success/failure summary and only raise a final exception after all Gold entities have been attempted.
- Never allow a single dimension or fact build failure to cancel the full Gold-stage execution.
- Any DataFrame created from Python lists, Row objects, test results, audit records, or exception logs MUST use an explicit StructType schema; do not rely on Spark schema inference.
- For `test.test_results`, define explicit types for all columns:
  - run_id STRING
  - test_name STRING
  - layer STRING
  - table_name STRING
  - status STRING
  - actual STRING
  - expected STRING
  - details STRING
  - checked_at TIMESTAMP
- Do not populate timestamp fields with untyped Python `None`; use a typed null timestamp column or add `current_timestamp()` after DataFrame creation using the predefined schema.

Dimension: gold.dim_order_date
- Source: SalesOrderHeader.OrderDate
- One row per calendar date.
- Required source column: `sales_orderheader.order_date`.

Dimension: gold.dim_ship_date
- Source: SalesOrderHeader.ShipDate
- Required source column: `sales_orderheader.ship_date`.

Dimension: gold.dim_customer
- Source: Customer + Address.
- User explicitly requested bypassing CustomerAddress.
- Use billing address from SalesOrderHeader.BillToAddressID to associate customer and address.
- Validate existence of:
  - customer.customer_id
  - salesorderheader.customer_id
  - salesorderheader.bill_to_address_id
  - address.address_id
  before performing joins.

Dimension: gold.dim_salesperson
- Source: Customer.SalesPerson
- Validate `customer.sales_person` exists before build.

Dimension: gold.dim_order
- Source: SalesOrderHeader
- Validate `sales_order_id` exists before build.

Dimension: gold.dim_product
- Source:
  - Product
  - ProductCategory
  - ProductModelProductDescription
  - ProductDescription
  - ProductModel
- Validate all join keys before build:
  - product.product_id
  - product.product_category_id
  - product.product_model_id
  - productcategory.product_category_id
  - productmodel.product_model_id
  - productmodelproductdescription.product_model_id
  - productmodelproductdescription.product_description_id
  - productdescription.product_description_id
- Do not reference a ProductModel name column because it is not present in the source schema.

Fact: gold.fact_sales_order
- Grain:
  - One row per SalesOrderDetail line.
- Validate existence of:
  - sales_order_id
  - sales_order_detail_id
  - product_id
  - order_qty
  before build and before joins to dimensions.

Fact/Dimension rationale:
- SalesOrderDetail is transactional and therefore fact-like.
- SalesOrderHeader contributes order-level attributes and dates.
- Customer, Product, SalesPerson, Order, and Date entities are dimensions.
- CustomerAddress is treated as a relationship table and not exposed in Gold.

## Test

Write all results into:
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

Implementation requirements:
- Create `test.test_results` using an explicit Spark schema; never infer schema from Python Row objects.
- `checked_at` must be a TIMESTAMP column and populated via `current_timestamp()` or an explicitly typed timestamp value.
- `actual`, `expected`, and `details` must be stored as STRING values even when null, numeric, boolean, or exception-derived.
- Before appending test results, validate that the DataFrame schema exactly matches the target table schema.

Required tests:

1. Row Count Reconciliation
- Bronze vs Silver for every table.
- PASS when variance <= 1%.

2. Gold Dimension PK Not Null
- dim_customer.customer_id
- dim_salesperson.salesperson_key
- dim_product.product_id
- dim_order.sales_order_id
- dim_order_date.date_key
- dim_ship_date.date_key

3. Gold Dimension PK Uniqueness
- Validate uniqueness of all dimension business keys.

4. Referential Integrity
- fact_sales_order.product_key exists in dim_product.
- fact_sales_order.customer_key exists in dim_customer.
- fact_sales_order.salesperson_key exists in dim_salesperson.
- fact_sales_order.order_key exists in dim_order.
- fact_sales_order.order_date_key exists in dim_order_date.
- fact_sales_order.ship_date_key exists in dim_ship_date.

5. Business Rule Sanity Check
- discount_pct between 0 and 1.
- net_sales_amount >= 0.
- gross_sales_amount >= net_sales_amount.
- ship_date is null or ship_date >= order_date.
- Regional sales aggregation returns at least one populated geography.

## Semantic model

Mode:
- Direct Lake

Tables:
- dim_order_date
- dim_ship_date
- dim_customer
- dim_salesperson
- dim_product
- dim_order
- fact_sales_order

Relationships:
- fact_sales_order → dim_order_date
- fact_sales_order → dim_ship_date
- fact_sales_order → dim_customer
- fact_sales_order → dim_salesperson
- fact_sales_order → dim_product
- fact_sales_order → dim_order

## Report

Page 1: Executive Sales Overview

Page 2: Regional Performance

Page 3: Orders and Discounts

Page 4: Product Performance

Page 5: Data Quality

## Data Agent

Role:
- AI Sales Performance Analyst for the SalesLT reporting environment.

Guardrails:
- Use only measures and dimensions from the semantic model.
- Do not fabricate unavailable geographic levels beyond city and postal code.
- Do not infer product model names because the source does not contain them.
- Clearly distinguish Order Date analysis from Ship Date analysis.
- Prefer governed measures over ad hoc calculations.
- Cite filters and time periods used in every answer.
- If data is unavailable in the model, state the limitation explicitly.
- Do not expose technical columns such as rowguid, password_hash, or password_salt.