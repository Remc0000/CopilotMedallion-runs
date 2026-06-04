# Run Spec 20260604-084600-e0d98a

## Updated specs

### Iteration 1 — 2026-06-04 08:48:51Z — failed layer: bronze (run: 20260604-084703-aac9f2)
- **Root cause (1-line summary)**: Bronze completed without producing discoverable Delta tables under `Tables/bronze/`, causing Silver to stop because no Bronze outputs could be found.
- **Cross-table audit**:
  - SalesLT/Address: yes — Bronze output must be discoverable.
  - SalesLT/Customer: yes — Bronze output must be discoverable.
  - SalesLT/CustomerAddress: yes — Bronze output must be discoverable.
  - SalesLT/Product: yes — Bronze output must be discoverable.
  - SalesLT/ProductCategory: yes — Bronze output must be discoverable.
  - SalesLT/ProductDescription: yes — Bronze output must be discoverable.
  - SalesLT/ProductModel: yes — Bronze output must be discoverable.
  - SalesLT/ProductModelProductDescription: yes — Bronze output must be discoverable.
  - SalesLT/SalesOrderDetail: yes — Bronze output must be discoverable.
  - SalesLT/SalesOrderHeader: yes — Bronze output must be discoverable.
- **Fix approach**: GENERALIZE — the failure is not table-specific; every Bronze table must be written and validated using the same discoverability rules.
- **What was changed**:
  - Tightened Bronze write requirements to mandate one Delta table per source under `Tables/bronze/<table_name>`.
  - Added post-write validation that each Bronze table path exists and contains a readable Delta table before proceeding.
  - Added explicit layer-level failure criteria if any required Bronze table is missing or if zero discoverable Bronze tables are found.

### Iteration 2 — 2026-06-04 08:52:18Z — failed layer: bronze (run: 20260604-084703-aac9f2)
- **Root cause (1-line summary)**: Silver could not discover any Bronze outputs; Bronze validation was insufficient because tables may have been written outside the Lakehouse `Tables/bronze/*` namespace or not registered as discoverable Delta tables.
- **Cross-table audit**:
  - SalesLT/Address: yes — same discoverability requirement applies.
  - SalesLT/Customer: yes — same discoverability requirement applies.
  - SalesLT/CustomerAddress: yes — same discoverability requirement applies.
  - SalesLT/Product: yes — same discoverability requirement applies.
  - SalesLT/ProductCategory: yes — same discoverability requirement applies.
  - SalesLT/ProductDescription: yes — same discoverability requirement applies.
  - SalesLT/ProductModel: yes — same discoverability requirement applies.
  - SalesLT/ProductModelProductDescription: yes — same discoverability requirement applies.
  - SalesLT/SalesOrderDetail: yes — same discoverability requirement applies.
  - SalesLT/SalesOrderHeader: yes — same discoverability requirement applies.
- **Fix approach**: GENERALIZE — all Bronze tables use the same write path and discoverability mechanism.
- **What was changed**:
  - Tightened Bronze path requirements to require writes to the target Lakehouse `SalesLTAnalytics` and nowhere else.
  - Added mandatory end-of-layer discovery validation against all expected Bronze table paths.
  - Added explicit prohibition on writing Bronze outputs to Files, temporary locations, workspace-default storage, or alternate Lakehouses.

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
- Target Lakehouse: **SalesLTAnalytics**

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
- Use defensive column references and assert required columns before every join, filter, groupBy, agg, Window, and withColumn operation.
- Use alias-prefixed joins during join construction, then materialize flat column names immediately after joins.
- Assert all groupBy and aggregation columns exist before execution.
- For REST/API responses, use explicit validation (`if x is None: raise RuntimeError(...)`) before any `.get()` access.
- Never use `saveAsTable`; write Delta directly to Lakehouse paths.
- All notebooks must begin with parameter cells for workspace, source, target, run_id, and layer configuration.
- Use idempotent overwrite patterns with `.mode("overwrite").option("overwriteSchema","true")`.
- Wrap per-table processing in error-loud try/except blocks that call `_save_error(layer, e)` (or `_save_error(layer, e, table=tbl)` inside loops) and re-raise appropriately.
- Enforce per-table isolation and resilient execution.
- Every code cell must start with a short comment block describing purpose and intent.
- Validate outputs after each layer and fail if no discoverable Delta tables were produced.
- All writes must target the explicitly specified target Lakehouse and must not use temporary paths, Files folders, workspace-default storage, or alternate Lakehouses.

## Bronze

Land each source table unchanged into `Tables/bronze/<table_name>` with source fidelity preserved.

Source-to-Bronze tables:
- bronze.address
- bronze.customer
- bronze.customer_address
- bronze.product
- bronze.product_category
- bronze.product_description
- bronze.product_model
- bronze.product_model_product_description
- bronze.sales_order_detail
- bronze.sales_order_header

Standard Bronze metadata:
- ingest_run_id
- ingest_timestamp_utc
- source_table
- source_lakehouse
- bronze_loaded_at

Write pattern:
- Delta format
- Overwrite mode with schema overwrite
- All Bronze outputs MUST be written into the target Lakehouse `SalesLTAnalytics`.
- Physical write location for every table MUST be exactly:
  - `Tables/bronze/address`
  - `Tables/bronze/customer`
  - `Tables/bronze/customer_address`
  - `Tables/bronze/product`
  - `Tables/bronze/product_category`
  - `Tables/bronze/product_description`
  - `Tables/bronze/product_model`
  - `Tables/bronze/product_model_product_description`
  - `Tables/bronze/sales_order_detail`
  - `Tables/bronze/sales_order_header`
- Do NOT write Bronze outputs to:
  - `Files/*`
  - temporary folders
  - local paths
  - workspace-default storage locations
  - any Lakehouse other than `SalesLTAnalytics`
- After each write:
  - Verify the path exists.
  - Verify Spark can read it as Delta.
  - Verify row count is greater than or equal to zero.
  - Record row count and write status.
- End-of-layer validation:
  - Enumerate and validate all ten required Bronze paths listed above.
  - Read each path back using Spark before marking Bronze complete.
  - Build a manifest of discovered Bronze tables and row counts.
  - Fail Bronze immediately if fewer than ten required Bronze tables are discoverable.
- Bronze completion criteria:
  - At least one Delta table must be discoverable under `Tables/bronze/`.
  - All required source tables listed above must have corresponding Bronze outputs.
  - If any required Bronze table is missing, fail Bronze with a clear error naming the missing table(s).
- Partition large transactional tables by year extracted from ModifiedDate when available:
  - sales_order_header
  - sales_order_detail
- Small dimension/master tables remain unpartitioned.

Data handling:
- No business transformations.
- Preserve binary ThumbnailPhoto in Product.
- Preserve rowguid and ModifiedDate columns.
- Record row counts per table.

## Silver

Standard processing for all tables:
- Convert column names to snake_case
- Add silver_loaded_at
- Add source_modified_date from modified_date where present
- Remove exact duplicate rows
- Validate primary-key uniqueness after dedup
- Optimize and vacuum according to Fabric best practices

Table-specific deduplication keys:
- address: address_id
- customer: customer_id
- customer_address: (customer_id, address_id)
- product: product_id
- product_category: product_category_id
- product_description: product_description_id
- product_model: product_model_id
- product_model_product_description: (product_model_id, product_description_id, culture)
- sales_order_header: sales_order_id
- sales_order_detail: (sales_order_id, sales_order_detail_id)

Silver business enrichment:
- customer:
  - Create sales_person_username derived from sales_person.
  - If value contains "\" then retain username portion only.
  - Remove numeric suffixes where present (example: jillian0 → jillian).
  - Preserve original sales_person field for traceability.
- product:
  - Create is_discontinued flag from discontinued_date.
  - Create is_currently_sellable based on sell_start_date, sell_end_date, and discontinued_date.
- sales_order_detail:
  - Calculate discount_amount = unit_price * unit_price_discount * order_qty.
- sales_order_header:
  - Create order_date_key and ship_date_key helper values for downstream dimensions.

Note:
- User requested ProductModel.Name as modelname, but ProductModel schema contains only ProductModelID, rowguid, and ModifiedDate. No Name column exists. Gold will therefore use ProductModelID as the available model reference unless the source schema is expanded.

## Gold

Target star schema per user requirements.

Fact table:
- fact_sales_order
  - Grain: one sales order line item (SalesOrderDetail joined to SalesOrderHeader)

Dimensions:
- dim_order_date
- dim_ship_date
- dim_customer
- dim_sales_person
- dim_order
- dim_product

Modeling notes:
- ProductDescription join path:
  Product → ProductModel → ProductModelProductDescription (Culture='en') → ProductDescription.
- ProductModel name requested by user is unavailable in source data.
- Regional reporting will be based on City and PostalCode from customer-address geography because no state/province/country fields exist.

## Test

Write all results to:
- gold.test_results

Required tests:
- Row count reconciliation
- Gold dimension PK not null
- Gold dimension PK uniqueness
- Referential integrity
- Business-rule sanity checks

Each test appends one result row and never overwrites prior test history.

## Semantic model

Mode:
- Direct Lake

Tables:
- fact_sales_order
- dim_customer
- dim_product
- dim_order
- dim_sales_person
- dim_order_date
- dim_ship_date

Relationships:
- fact_sales_order → dim_product
- fact_sales_order → dim_customer
- fact_sales_order → dim_sales_person
- fact_sales_order → dim_order
- fact_sales_order → dim_order_date
- fact_sales_order → dim_ship_date

## Report

Page 1 — Executive Sales Overview
- KPI cards
- Monthly sales trend
- Sales by product category
- Top customers
- Salesperson performance

Page 2 — Regional Performance
- Bubble map using City
- Regional analysis visuals

Page 3 — Discounts & Salesperson Analysis

Page 4 — Orders & Fulfillment

Page 5 — Data Quality

## Data Agent

Role:
- Sales Performance Intelligence Agent for SalesLTAnalytics.

Grounding:
- Use only the Direct Lake semantic model.

Guardrails:
- Never answer using data outside the semantic model.
- Clearly distinguish sales amount, gross sales, and discount metrics.
- Use measures rather than ad hoc calculations whenever equivalent measures exist.
- When geography is requested, explain that available regional granularity is City and PostalCode.
- If requested attributes are unavailable in source data (for example ProductModel name), state that the field is not present in the model.
- Surface filter context and date range assumptions when reporting results.
- Prioritize dimensional hierarchies for drill-down recommendations.
- Do not expose technical columns such as rowguid, password_hash, password_salt, or ingestion metadata.