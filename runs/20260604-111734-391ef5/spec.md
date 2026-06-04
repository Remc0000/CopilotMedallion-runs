# Run Spec 20260604-111656-40a6d8

## Updated specs

### Iteration 1 — 2026-06-04 11:24:03Z — failed layer: bronze (run: 20260404-111734-391ef5)
- **Root cause (1-line summary)**: Bronze completed without producing discoverable Delta tables under `Tables/bronze/`, causing Silver generation to stop.
- **Cross-table audit**:
  - Address: yes — discoverability failure can affect any Bronze write.
  - Customer: yes — discoverability failure can affect any Bronze write.
  - CustomerAddress: yes — discoverability failure can affect any Bronze write.
  - Product: yes — discoverability failure can affect any Bronze write.
  - ProductCategory: yes — discoverability failure can affect any Bronze write.
  - ProductDescription: yes — discoverability failure can affect any Bronze write.
  - ProductModel: yes — discoverability failure can affect any Bronze write.
  - ProductModelProductDescription: yes — discoverability failure can affect any Bronze write.
  - SalesOrderDetail: yes — discoverability failure can affect any Bronze write.
  - SalesOrderHeader: yes — discoverability failure can affect any Bronze write.
- **Fix approach**: GENERALIZE — the issue is not table-specific; every Bronze source table must follow the same discoverable write pattern and validation.
- **What was changed**:
  - Tightened Bronze write-path requirements to require physical Delta writes under `Tables/bronze/<table_name_lower>`.
  - Added mandatory post-write validation that each target path exists and contains Delta data before marking the table successful.
  - Added explicit requirement to raise an error if fewer than the 10 expected Bronze tables are discoverable at notebook completion.

### Iteration 2 — 2026-06-04 11:31:20Z — failed layer: bronze (run: 20260604-111734-391ef5)
- **Root cause (1-line summary)**: Bronze validation still reported zero discoverable tables; the build needs explicit Lakehouse-relative write locations and discoverability verification using the exact required folder names.
- **Cross-table audit**:
  - Address: yes — same discoverability requirement applies.
  - Customer: yes — same discoverability requirement applies.
  - CustomerAddress: yes — same discoverability requirement applies.
  - Product: yes — same discoverability requirement applies.
  - ProductCategory: yes — same discoverability requirement applies.
  - ProductDescription: yes — same discoverability requirement applies.
  - ProductModel: yes — same discoverability requirement applies.
  - ProductModelProductDescription: yes — same discoverability requirement applies.
  - SalesOrderDetail: yes — same discoverability requirement applies.
  - SalesOrderHeader: yes — same discoverability requirement applies.
- **Fix approach**: GENERALIZE — every Bronze table must use the same path convention and validation logic.
- **What was changed**:
  - Added explicit requirement to write to Lakehouse-relative paths rooted at `Tables/bronze/`.
  - Added required Delta-log validation (`_delta_log` presence and successful Delta read).
  - Added requirement to enumerate and verify all 10 expected folder names before Bronze completion.

## Inputs
- Workspace: `35fe2703-387e-4ca0-a948-16313a09cf18`
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
- Target Lakehouse: **b**

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
- Use defensive column references and assert required columns before every join, filter, aggregation, and derived-column expression.
- Do not use `saveAsTable`; write Delta directly to Lakehouse paths.
- Every notebook must begin with parameter cells for workspace, lakehouse, run_id, source paths, and target paths.
- Use idempotent overwrite patterns with `overwriteSchema=true`.
- Process source tables independently per layer.
- Every code cell must start with a short explanatory comment block.
- Bronze, Silver, and Gold outputs must be physically discoverable Delta folders under `Tables/<layer>/...` before the layer is considered successful.
- Discoverability checks must validate both successful Delta reads and the existence of a `_delta_log` directory in each expected output folder.

## Bronze

- Land each source table unchanged into `Tables/bronze/<table_name_lower>`.
- Preserve all source columns and datatypes.
- Add metadata columns:
  - `_run_id`
  - `_ingested_at`
  - `_source_table`
  - `_bronze_ts`
- Write mode: overwrite with schema overwrite enabled.
- REQUIRED write pattern:
  - Write each table as Delta to a physical Lakehouse-relative path rooted at `Tables/bronze/`.
  - Do not write to `Files/`, temporary folders, notebook-local storage, or any path outside `Tables/bronze/`.
  - The final folder names must be exactly:
    - `address`
    - `customer`
    - `customeraddress`
    - `product`
    - `productcategory`
    - `productdescription`
    - `productmodel`
    - `productmodelproductdescription`
    - `salesorderheader`
    - `salesorderdetail`
  - Do not write only to Files, temp locations, temp views, or notebook-local paths.
- REQUIRED post-write validation for every table:
  - Verify the target path exists.
  - Verify a `_delta_log` directory exists in the target path.
  - Verify it is readable as Delta immediately after write.
  - Verify the Delta read returns a row count greater than or equal to the source row count.
  - Record row count and target path in the results summary.
  - Mark the table failed if validation does not pass.
- REQUIRED notebook completion check:
  - Enumerate folders directly under `Tables/bronze/`.
  - Expected discoverable table count: 10.
  - Required folders:
    - `address`
    - `customer`
    - `customeraddress`
    - `product`
    - `productcategory`
    - `productdescription`
    - `productmodel`
    - `productmodelproductdescription`
    - `salesorderheader`
    - `salesorderdetail`
  - Raise an error if zero tables are discoverable.
  - Raise an error if any required folder is missing.
  - Raise an error if any required folder lacks a readable Delta table.
- Partitioning:
  - SalesOrderHeader: partition by year derived from OrderDate.
  - SalesOrderDetail: partition by SalesOrderID hash bucket or non-partitioned if volume is small.
  - All master tables: non-partitioned.
- Bronze outputs:
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

General rules:
- Convert all columns to snake_case.
- Standardize timestamp columns.
- Add `_silver_ts`, `_run_id`, and lineage metadata.
- Deduplicate using latest modified_date where available.
- Remove duplicate records while preserving business keys.

Table-specific deduplication:
- address: dedupe on `address_id`.
- customer: dedupe on `customer_id`.
- customeraddress: dedupe on (`customer_id`, `address_id`).
- product: dedupe on `product_id`.
- productcategory: dedupe on `product_category_id`.
- productdescription: dedupe on `product_description_id`.
- productmodel:
  - dedupe on `product_model_id`.
  - NOTE: source schema contains no Name column. User requested ProductModel.Name → modelname in Gold. This cannot be satisfied from the supplied schema.
- productmodelproductdescription:
  - dedupe on (`product_model_id`, `product_description_id`, `culture`).
- salesorderheader: dedupe on `sales_order_id`.
- salesorderdetail: dedupe on (`sales_order_id`, `sales_order_detail_id`).

## Gold

Target star schema requested by user.

- Build:
  - dim_order_date
  - dim_ship_date
  - dim_customer
  - dim_sales_person
  - dim_order
  - dim_product
  - fact_sales_order

- Use the join paths and constraints defined in the original specification.

## Test

- Persist test results to `Tables/test/test_results`.

Standard tests:
1. Row Count Reconciliation
2. Gold Dimension PK Not Null
3. Gold Dimension PK Uniqueness
4. Referential Integrity
5. Business Rule Sanity Checks

## Semantic model

Storage mode:
- Direct Lake.

Tables:
- dim_order_date
- dim_ship_date
- dim_customer
- dim_sales_person
- dim_order
- dim_product
- fact_sales_order

Relationships:
- fact_sales_order -> dim_order_date
- fact_sales_order -> dim_ship_date
- fact_sales_order -> dim_customer
- fact_sales_order -> dim_sales_person
- fact_sales_order -> dim_product
- fact_sales_order -> dim_order

## Report

Page 1: Executive Sales Overview
Page 2: Regional Performance
Page 3: Orders and Discounts
Page 4: Product Performance
Page 5: Data Quality

Use the visual and KPI requirements defined in the original specification.

## Data Agent

Role:
- Sales Performance Analytics Agent for the SalesLT reporting solution.

Instructions:
- Use only the approved semantic model.
- Do not fabricate unavailable attributes.
- Follow the original guardrails and starter-question requirements.