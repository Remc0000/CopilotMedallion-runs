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

### Iteration 2 — 2026-06-04 18:31:25Z — failed layer: bronze (run: 20260604-182917-5456ba)
- **Root cause (1-line summary)**: Spark session was cancelled after statement failures; a likely Bronze-wide cause is attempting MERGE, partitioning, or key-based logic against columns that are missing, differently cased, or not unique in the discovered source schema.
- **Cross-table audit**:
  - Address: yes — key/partition validation required before MERGE.
  - Customer: yes — key/partition validation required before MERGE.
  - CustomerAddress: yes — composite-key validation required before MERGE.
  - Product: yes — key/partition validation required before MERGE.
  - ProductCategory: yes — key/partition validation required before MERGE.
  - ProductDescription: yes — key/partition validation required before MERGE.
  - ProductModel: yes — key/partition validation required before MERGE.
  - ProductModelProductDescription: yes — composite-key validation required before MERGE.
  - SalesOrderDetail: yes — key validation required before MERGE.
  - SalesOrderHeader: yes — key and OrderDate validation required before MERGE/partitioning.
- **Fix approach**: GENERALIZE — the same defensive schema-validation and write fallback rules should apply to every Bronze source table.
- **What was changed**:
  - Added mandatory schema logging and column-existence checks before any MERGE, partition, or incremental logic.
  - Added fallback to overwrite/full-load when required MERGE keys are unavailable in the discovered schema.
  - Required Bronze notebooks to continue table-by-table and record failures rather than terminating the Spark session on the first table error.

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

Mandatory ingestion safeguards:
- For every source table, log and validate the discovered schema before transformations.
- Treat source column names using the exact discovered casing.
- Do not construct a MERGE condition until all referenced key columns are confirmed to exist in the source DataFrame.
- If a configured business key is missing from the discovered schema, do not execute MERGE; instead perform a full overwrite of the Bronze target table and record the condition in notebook logs.
- Partition columns must be validated before write. If the configured partition column does not exist, write the table unpartitioned.
- Process tables independently so that one table failure does not cancel the entire Spark session.

Write strategy:
- Daily incremental load using `ModifiedDate` only when that column exists in the source table.
- If `ModifiedDate` does not exist, perform a full-table load or use an alternative available business timestamp for that table.
- MERGE/UPSERT into bronze on business key(s) only after key-column validation succeeds.
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
- dim_order_date
- dim_ship_date
- dim_customer
- dim_salesperson
- dim_order
- dim_product

Fact table:
- fact_sales_order

Use the definitions already specified in this document.

## Test

Use the tests already specified in this document.

## Semantic model

Mode:
- Direct Lake

Use the tables, relationships, hierarchies, and measures already specified in this document.

## Report

Create the report pages and visuals already specified in this document.

## Data Agent

Use the role, domain hints, starter questions, and guardrails already specified in this document.