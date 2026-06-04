# Run Spec 20260604-094054-ac79f4

## Updated specs

### Iteration 1 — 2026-06-04 09:44:41Z — failed layer: bronze (run: 20260604-094133-042506)
- **Root cause (1-line summary)**: Bronze completed without producing discoverable Delta tables under `Tables/bronze/`, causing Silver generation to stop.
- **Cross-table audit**:
  - Address: yes — discoverability requirement applies.
  - Customer: yes — discoverability requirement applies.
  - CustomerAddress: yes — discoverability requirement applies.
  - Product: yes — discoverability requirement applies.
  - ProductCategory: yes — discoverability requirement applies.
  - ProductDescription: yes — discoverability requirement applies.
  - ProductModel: yes — discoverability requirement applies.
  - ProductModelProductDescription: yes — discoverability requirement applies.
  - SalesOrderDetail: yes — discoverability requirement applies.
  - SalesOrderHeader: yes — discoverability requirement applies.
- **Fix approach**: GENERALIZE — the failure is not table-specific; every Bronze table must be written to a discoverable path using a consistent naming convention and verified after write.
- **What was changed**:
  - Tightened Bronze output-path rules and required flat table-name mapping.
  - Added mandatory post-write verification that each Delta path exists and contains data.
  - Added notebook-level failure condition if any source table is skipped or if discoverable table count differs from source table count.

## Inputs
- Workspace: `6b270c7d-4c29-43c3-a8de-4debee058dd2`
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
- Use defensive column references and validate required columns before every join, filter, aggregation, window, and derived-column operation.
- After every join, immediately project alias-prefixed columns into flat names before downstream use.
- Do not use `saveAsTable`; write Delta files directly to lakehouse paths.
- Process source tables independently per layer and record results.
- Every notebook must start with parameter cells for workspace, lakehouse, run_id, source paths, and target paths.
- Every code cell must begin with a short comment block explaining purpose and intent.
- Retain all existing Rules A–K exactly as written in this spec.

## Bronze

- Land each source table 1:1 into `Tables/bronze/<table_name_lower>`.
- Preserve source schema exactly; no business transformations.
- Add ingestion metadata:
  - `_run_id`
  - `_ingested_at`
  - `_source_table`
  - `_bronze_ts`
- Write mode: Delta overwrite with schema evolution enabled.
- REQUIRED table-name mapping:
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
- Write each table directly to the corresponding Delta path under `Tables/bronze/`; do not write to alternate folders, nested schema folders, temporary folders, or non-Delta locations.
- After every write:
  - Verify the target path exists.
  - Verify it is readable as Delta.
  - Verify row count > 0 when the source table row count > 0.
  - Record rows written and physical path in the results summary.
- Partitioning:
  - `salesorderheader`: partition by year derived from `OrderDate`.
  - `salesorderdetail`: partition by year derived from `ModifiedDate`.
  - Remaining tables: unpartitioned due to expected small size.
- Persist all audit columns (`rowguid`, `ModifiedDate`) unchanged.
- Record row counts written for each table and emit bronze summary JSON.
- Mandatory completion check:
  - Expected Bronze outputs = 10 tables.
  - Raise an error if fewer than 10 discoverable Delta tables exist under `Tables/bronze/` after processing.
  - Raise an error if zero tables were written, even if no Spark exception occurred.

## Silver

Standard actions for all tables:
- Convert column names to snake_case.
- Preserve source business keys.
- Add `_silver_ts`, `_run_id`, `_is_current`.
- Remove exact duplicate rows.
- Validate required key columns before writes.
- OPTIMIZE output tables after successful write.

Per-table deduplication strategy:
- address: dedupe on `address_id`, latest `modified_date`.
- customer: dedupe on `customer_id`, latest `modified_date`.
- customer_address: dedupe on composite key (`customer_id`,`address_id`), latest `modified_date`.
- product: dedupe on `product_id`, latest `modified_date`.
- product_category: dedupe on `product_category_id`, latest `modified_date`.
- product_description: dedupe on `product_description_id`, latest `modified_date`.
- product_model: dedupe on `product_model_id`, latest `modified_date`.
- product_model_product_description: dedupe on (`product_model_id`,`product_description_id`,`culture`), latest `modified_date`.
- sales_order_header: dedupe on `sales_order_id`, latest `modified_date`.
- sales_order_detail: dedupe on (`sales_order_id`,`sales_order_detail_id`), latest `modified_date`.

Additional Silver enrichments:
- Derive product lifecycle flags:
  - `is_discontinued`
  - `is_currently_sellable`
- Derive discount percentage at line level:
  - `discount_pct = unit_price_discount / nullif(unit_price,0)`
- Derive order calendar attributes from `order_date`.
- Normalize salesperson source value into helper field for Gold dimension creation.
- NOTE: User requested ProductModel Name as model name. The provided schema contains only `ProductModelID`, `rowguid`, and `ModifiedDate`; no model name column exists. Gold will therefore expose ProductModelID unless an additional source containing model names is supplied.

## Gold

Target star schema per user requirements.

Dimensions:
- dim_order_date
- dim_ship_date
- dim_customer
- dim_salesperson
- dim_order
- dim_product

Fact:
- fact_sales_order

Use all requirements from the existing spec unchanged.

## Test

All tests append results into:
- `Tables/test/test_results`

Execute all tests from the existing spec unchanged.

## Semantic model

Mode:
- Direct Lake

Use all tables, relationships, hierarchies, measures, and geographic settings from the existing spec unchanged.

## Report

Build all report pages and visuals from the existing spec unchanged.

## Data Agent

Use the role, domain hints, starter questions, and guardrails from the existing spec unchanged.