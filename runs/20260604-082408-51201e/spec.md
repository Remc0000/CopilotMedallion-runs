# Run Spec 20260604-082256-3144db

## Updated specs

### Iteration 1 — 2026-06-04 08:28:53Z — failed layer: bronze (run: 20260604-082408-51201e)
- **Root cause (1-line summary)**: Bronze completed without producing discoverable Delta tables under `Tables/bronze`, causing Silver to find zero upstream inputs and abort.
- **Cross-table audit**: Address: yes — discoverability requirement applies; Customer: yes — discoverability requirement applies; CustomerAddress: yes — discoverability requirement applies; Product: yes — discoverability requirement applies; ProductCategory: yes — discoverability requirement applies; ProductDescription: yes — discoverability requirement applies; ProductModel: yes — discoverability requirement applies; ProductModelProductDescription: yes — discoverability requirement applies; SalesOrderDetail: yes — discoverability requirement applies; SalesOrderHeader: yes — discoverability requirement applies.
- **Fix approach**: GENERALIZE — the failure is not table-specific; every Bronze table must be written to a discoverable Delta location and validated after write.
- **What was changed**:
  - Tightened Bronze output path requirements to require physical Delta tables under `Tables/bronze/<bronze_table_name>`.
  - Added post-write validation that each expected Bronze table exists and is readable before the notebook completes.
  - Added a fail-fast rule when fewer than the expected 10 Bronze tables are discoverable.

### Iteration 2 — 2026-06-04 08:30:33Z — failed layer: bronze (run: 20260604-082408-51201e)
- **Root cause (1-line summary)**: Build interruption due to Spark/session restart mid-build, risking partial Bronze output and non-idempotent reruns.
- **Cross-table audit**: Address: yes — restart can interrupt write; Customer: yes — restart can interrupt write; CustomerAddress: yes — restart can interrupt write; Product: yes — restart can interrupt write; ProductCategory: yes — restart can interrupt write; ProductDescription: yes — restart can interrupt write; ProductModel: yes — restart can interrupt write; ProductModelProductDescription: yes — restart can interrupt write; SalesOrderDetail: yes — restart can interrupt write; SalesOrderHeader: yes — restart can interrupt write.
- **Fix approach**: GENERALIZE — the interruption risk applies uniformly to all Bronze tables and requires restart-safe processing rules.
- **What was changed**:
  - Added restart-safe Bronze processing requirements with per-table isolation and validation.
  - Required skipping already-valid Bronze outputs on rerun using path-level validation.
  - Added final completeness checks that verify all 10 expected Bronze Delta tables are present before success.

### Iteration 3 — 2026-06-04 08:38:10Z — failed layer: bronze (run: 20260604-082408-51201e)
- **Root cause (1-line summary)**: Spark server restarted mid-build; Bronze processing must resume safely without reprocessing assumptions or notebook-state dependencies.
- **Cross-table audit**: Address: yes — restart can occur before or after write validation; Customer: yes — same risk; CustomerAddress: yes — same risk; Product: yes — same risk; ProductCategory: yes — same risk; ProductDescription: yes — same risk; ProductModel: yes — same risk; ProductModelProductDescription: yes — same risk; SalesOrderDetail: yes — same risk; SalesOrderHeader: yes — same risk.
- **Fix approach**: GENERALIZE — the restart condition is infrastructure-wide and affects every Bronze table identically.
- **What was changed**:
  - Added explicit resume-from-lakehouse behavior requiring reconstruction of progress from existing Delta paths after any restart.
  - Required per-table checkpoint validation before processing and after processing.
  - Required final Bronze success only when all 10 expected tables are validated in the current Spark session after any restart recovery.

## Inputs
- Workspace: `fa4681d4-fabe-41bc-b3c8-d6daa3601f10`
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

Cross-cutting code rules:
- Use defensive column references and validate schema before transformations.
- Apply alias-prefixed joins and immediately flatten required columns after joins.
- Assert required columns exist before every join, filter, groupBy, agg, Window, or withColumn operation.
- Use defensive REST handling with `if x is None: raise RuntimeError(...)` before any `.get()` access.
- Do not use `saveAsTable`; write Delta files directly to lakehouse paths.
- All notebooks must begin with parameter cells for workspace, source, target, layer, and run_id.
- Use idempotent overwrite patterns with `overwriteSchema=true`.
- Wrap table processing in error-loud try/except blocks that call `_save_error(layer, e)` and re-raise.
- Process source tables independently per-table wherever possible.
- Every code cell must start with a short markdown-style Python comment block describing purpose and intent.
- After every layer write, validate the target path exists, is Delta formatted, and is readable before continuing.
- After any Spark restart or notebook rerun, reconstruct processing state exclusively from lakehouse contents; never rely on Python variables, cached DataFrames, temp views, or prior notebook execution state.

### Global Spark column-reference rules (apply to ALL layers: Bronze, Silver, Gold)
(Keep all existing rules A–K exactly as written in the original spec.)

Rule K — Resilience to partial output: Bronze MUST write Delta tables that the next layer can discover.
- Write discoverable Delta outputs under Tables/bronze, Tables/silver, and Tables/gold.
- Emit completion summaries.
- Raise an error if no tables were written.

## Bronze

Land each source table unchanged into `Tables/bronze/<table_name>` with:
- Source metadata columns:
  - `_run_id`
  - `_ingested_at`
  - `_source_table`
  - `_bronze_ts`
- Write mode: overwrite
- Format: Delta
- Schema evolution enabled via overwriteSchema=true

Bronze tables:
- bronze_address
- bronze_customer
- bronze_customeraddress
- bronze_product
- bronze_productcategory
- bronze_productdescription
- bronze_productmodel
- bronze_productmodelproductdescription
- bronze_salesorderdetail
- bronze_salesorderheader

Required physical output locations:
- `Tables/bronze/bronze_address`
- `Tables/bronze/bronze_customer`
- `Tables/bronze/bronze_customeraddress`
- `Tables/bronze/bronze_product`
- `Tables/bronze/bronze_productcategory`
- `Tables/bronze/bronze_productdescription`
- `Tables/bronze/bronze_productmodel`
- `Tables/bronze/bronze_productmodelproductdescription`
- `Tables/bronze/bronze_salesorderdetail`
- `Tables/bronze/bronze_salesorderheader`

Bronze completion requirements:
- Process all 10 source tables independently.
- After writing each table, immediately read the Delta path back and verify row count > 0 when the source table is non-empty.
- Record the written path for each table in the completion summary.
- Before the Bronze notebook exits, assert that all expected Bronze table paths are discoverable under `Tables/bronze`.
- Fail the run with a clear error if fewer than 10 Bronze tables are discoverable.
- Silver must be able to discover Bronze inputs exclusively from the written Delta outputs, without relying on notebook state.

Bronze restart-resilience requirements:
- Treat every source table as an independent unit of work; a failure or session restart while processing one table must not prevent validation of previously written tables.
- On rerun, first check whether each expected Delta path already exists and is readable; if it contains a valid Delta table, it may be overwritten or reused, but the notebook must not assume prior in-memory state exists.
- Maintain a completion manifest in the notebook summary containing all successfully validated Bronze table paths.
- Before reporting success, perform a fresh filesystem scan of `Tables/bronze` and verify that all 10 required Delta table locations exist and are readable in the current Spark session.
- Any interruption, session restart, or partial-write condition must result in a rerunnable notebook that reconstructs state solely from lakehouse contents.
- At notebook startup, build the processing queue by scanning the 10 required Bronze output paths and marking each table as validated, missing, or unreadable.
- If a restart occurs, re-read every previously completed Delta path from storage before continuing with remaining tables.
- Do not require all 10 tables to be processed within a single Spark session; success is determined only by the final validation scan of all required Delta outputs.

Partitioning:
- SalesOrderHeader: partition by OrderDate year/month helper columns.
- SalesOrderDetail: partition by SalesOrderID hash bucket helper.
- Remaining tables: unpartitioned due to expected small dimension size.

## Silver

Common standards:
- Rename all columns to snake_case.
- Preserve business keys.
- Add:
  - `_silver_ts`
  - `_source_modified_date`
  - `_is_current`
- Remove duplicate records.
- Retain rowguid only for lineage, not business modeling.
- OPTIMIZE after write.

Deduplication keys:
- address: AddressID
- customer: CustomerID
- customeraddress: CustomerID + AddressID
- product: ProductID
- productcategory: ProductCategoryID
- productdescription: ProductDescriptionID
- productmodel: ProductModelID
- productmodelproductdescription: ProductModelID + ProductDescriptionID + Culture
- salesorderheader: SalesOrderID
- salesorderdetail: SalesOrderID + SalesOrderDetailID

Silver business enhancements:
- Customer:
  - Normalize EmailAddress.
  - Trim text fields.
- Product:
  - Create is_discontinued flag from DiscontinuedDate.
  - Create is_active_product flag from SellStartDate/SellEndDate.
- SalesOrderHeader:
  - Create order_year, order_month, order_date_key.
  - Create ship_year, ship_month, ship_date_key.
- SalesOrderDetail:
  - Create line_discount_amount = OrderQty * UnitPrice * UnitPriceDiscount.
  - Create gross_line_amount = OrderQty * UnitPrice.
  - Create net_line_amount = OrderQty * UnitPrice * (1 - UnitPriceDiscount).

Important modeling note:
- User requested Customer dimension by combining Customer and Address without using CustomerAddress.
- The available schema contains no direct Customer-to-Address relationship. CustomerAddress is the only relationship table available.
- To satisfy reporting requirements while preserving data correctness, Silver should retain CustomerAddress and Gold should use it internally to obtain the current customer-address association. The junction table will not be exposed as a Gold dimension.

Important modeling note:
- User requested ProductModel Name. The provided ProductModel schema contains only ProductModelID, rowguid, and ModifiedDate.
- No Name column exists in source data.
- Gold Product dimension will expose ProductModelID and English product description attributes instead unless additional source columns become available.

## Gold

Target star schema optimized for Direct Lake.

Dimensions:
- dim_order_date
- dim_ship_date
- dim_customer
- dim_salesperson
- dim_order
- dim_product

Fact:
- fact_sales_order

(Use the Gold design exactly as defined in the original spec.)

## Test

All test results append to:
- `Tables/test/test_results`

(Use all tests exactly as defined in the original spec.)

## Semantic model

Storage mode:
- Direct Lake

(Use the semantic model exactly as defined in the original spec.)

## Report

(Use the report definition exactly as defined in the original spec.)

## Data Agent

(Use the data agent definition exactly as defined in the original spec.)