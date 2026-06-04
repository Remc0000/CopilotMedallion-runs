# Run Spec 20260604-121533-af0e87

## Updated specs

### Iteration 1 — 2026-06-04 12:21:18Z — failed layer: silver (run: 20260604-121637-7d1d0a)
- **Root cause (1-line summary)**: Silver-layer execution failed with a session-wide cancellation, indicating a single table transformation likely terminated the Spark session before remaining Silver tables could complete.
- **Cross-table audit**:
  - Address: yes — any Silver transform failure could cancel the session.
  - Customer: yes — contains derived-column logic and deduplication.
  - CustomerAddress: yes — composite-key deduplication and optional audit columns.
  - Product: yes — multiple derived columns and date-based business logic.
  - ProductCategory: yes — rename/deduplication path can fail similarly.
  - ProductDescription: yes — standard Silver processing path.
  - ProductModel: yes — standard Silver processing path.
  - ProductModelProductDescription: yes — junction-table handling and composite keys.
  - SalesOrderDetail: yes — calculated measures and deduplication.
  - SalesOrderHeader: yes — derived date attributes and deduplication.
- **Fix approach**: GENERALIZE — the failure signature is session-level rather than table-specific, so all Silver tables must be processed independently with validation and isolated error handling.
- **What was changed**:
  - Tightened the Silver section to require per-table processing with independent try/except boundaries.
  - Added mandatory schema validation before deduplication and business-derived columns.
  - Required Silver completion tracking and prohibition of a single multi-table Silver transformation plan.

### Iteration 2 — 2026-06-04 12:23:02Z — failed layer: silver (run: 20260604-121637-7d1d0a)
- **Root cause (1-line summary)**: Silver layer again terminated with a session-wide cancellation; a table-level Spark action likely failed and propagated to the entire session.
- **Cross-table audit**:
  - Address: yes — schema validation, deduplication, and write operations can trigger session-ending failures.
  - Customer: yes — derived salesperson columns and deduplication can fail.
  - CustomerAddress: yes — composite-key handling can fail.
  - Product: yes — date-derived business logic can fail.
  - ProductCategory: yes — rename and deduplication path can fail.
  - ProductDescription: yes — standard Silver transformation path can fail.
  - ProductModel: yes — standard Silver processing path can fail.
  - ProductModelProductDescription: yes — junction-table composite-key processing can fail.
  - SalesOrderDetail: yes — calculated columns can fail.
  - SalesOrderHeader: yes — date derivations can fail.
- **Fix approach**: GENERALIZE — the failure remains non-table-specific, so all Silver tables require execution isolation, explicit materialization, and write validation.
- **What was changed**:
  - Tightened Silver execution to require one notebook-level processing unit per table iteration with forced materialization before write.
  - Added mandatory row-count validation before and after write operations.
  - Required failures to be logged and skipped so remaining Silver tables continue processing.

### Iteration 3 — 2026-06-04 12:32:57Z — failed layer: silver (run: 20260604-121637-7d1d0a)
- **Root cause (1-line summary)**: Product Silver business logic incorrectly treated `discontinued_date` as mandatory, and error logging failed because `_save_error(table=...)` was called with an unsupported argument.
- **Cross-table audit**:
  - Address: yes — optional source columns could be absent and trigger the same validation pattern.
  - Customer: yes — derived fields based on `sales_person` must tolerate missing optional columns.
  - CustomerAddress: yes — optional audit columns may not exist.
  - Product: yes — confirmed failure; `discontinued_date` may be absent in source schema.
  - ProductCategory: yes — optional attributes may be absent after snake_case conversion.
  - ProductDescription: yes — optional attributes may be absent.
  - ProductModel: yes — schema differences can cause the same issue.
  - ProductModelProductDescription: yes — optional columns may vary by source schema.
  - SalesOrderDetail: yes — derived calculations must validate only truly required columns.
  - SalesOrderHeader: yes — optional date columns such as `ship_date` may be absent.
- **Fix approach**: GENERALIZE — the root cause is improper handling of optional columns and inconsistent error-handler signatures, which can affect all Silver tables.
- **What was changed**:
  - Tightened Silver schema-validation rules to distinguish required versus optional columns.
  - Made `product.discontinued_date` optional and defined fallback behavior when absent.
  - Required Silver error handling to call `_save_error` using only supported positional arguments and never assume a `table=` keyword parameter exists.

## Inputs
- Workspace: `120db309-94d0-4c4a-9183-504d81b9a3bf`
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
- Target Lakehouse: **e**

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
- Use defensive column references and validate required columns before every join, filter, groupBy, agg, window, and withColumn.
- After every join, immediately project alias-prefixed columns into flat names before downstream transformations.
- Assert aggregation and groupBy columns exist before execution.
- For REST/API responses, use `if x is None: raise RuntimeError(...)` before any `.get(...)` access.
- Never use `saveAsTable`; write Delta directly to lakehouse paths.
- Every notebook must begin with parameter cells for workspace, source lakehouse, target lakehouse, run_id, layer, and paths.
- Use idempotent overwrite patterns with `mode("overwrite")` and `overwriteSchema=true`.
- All exception handling must be error-loud: call `_save_error` only with the arguments supported by its implementation; do not assume a `table=` keyword argument exists.
- Each code cell must start with a short comment block explaining purpose and business intent.
- Process source tables independently per layer wherever possible.
- Validate outputs exist before allowing downstream layers to execute.

### Global Spark column-reference rules (apply to ALL layers: Bronze, Silver, Gold)
Retain all existing Rules A–K exactly as currently specified.

## Bronze

Land each source table unchanged into `Tables/bronze/<table_name>`.

Per-table actions:
- Preserve source schema exactly.
- Add metadata columns:
  - `_run_id`
  - `_ingested_at`
  - `_source_table`
  - `_source_lakehouse`
- Write Delta format with overwrite and overwriteSchema enabled.
- Partition large transactional tables by year/month derived from:
  - SalesOrderHeader.OrderDate
  - SalesOrderDetail.ModifiedDate
- Smaller master tables remain unpartitioned.
- Preserve binary content in Product.ThumbNailPhoto.
- Record row counts written for each table.

Expected Bronze outputs:
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

## Silver

Common Silver standards:
- Process each Bronze table in an independent loop iteration.
- Each table must have its own try/except block, schema validation, transformation, and Delta write.
- Never build a single DataFrame lineage, SQL statement, or execution plan spanning multiple Silver tables.
- Before any rename, deduplication, window, filter, or derived-column logic, validate that all required source columns for that table exist and fail with a table-specific error message naming the missing column.
- Distinguish REQUIRED columns from OPTIONAL columns. Missing OPTIONAL columns must not fail the table; instead skip the related derivation and populate the derived output with NULL/default values as defined below.
- A failure on one Silver table must not prevent attempted processing of the remaining Silver tables; emit a per-table status summary at completion.
- For every table, force materialization of the transformed DataFrame and validate a non-error row count before attempting the Delta write.
- After every Silver write, immediately validate that the target Delta path exists and that the written row count is retrievable.
- Record table name, pre-write row count, post-write row count, status, and error message (if any) in a Silver execution summary.
- Continue processing remaining tables after a table-specific failure; only fail the Silver layer after the summary is produced.
- Error logging must use `_save_error` only with its supported signature; do not pass unsupported keyword arguments.
- Rename all columns to snake_case.
- Add `_silver_loaded_at`.
- Trim string columns.
- Deduplicate using business keys and latest modified_date when available.
- Standardize timestamps.
- Remove exact duplicate records.
- OPTIMIZE/VACUUM according to Fabric guidance.

Table-specific deduplication:
- address: key = address_id
- customer: key = customer_id
- customeraddress: key = (customer_id, address_id)
- product: key = product_id
- productcategory: key = product_category_id
- productdescription: key = product_description_id
- productmodel: key = product_model_id
- productmodelproductdescription: key = (product_model_id, product_description_id, culture)
- salesorderheader: key = sales_order_id
- salesorderdetail: key = (sales_order_id, sales_order_detail_id)

Silver business enhancements:
- customer:
  - Validate sales_person exists before deriving salesperson fields.
  - Derive cleaned_salesperson_raw from sales_person.
  - Derive salesperson_username by extracting username after "\" when present.
  - If sales_person is absent, set cleaned_salesperson_raw and salesperson_username to NULL and continue.
- product:
  - Validate sell_start_date and sell_end_date before derived calculations.
  - Treat discontinued_date as OPTIONAL.
  - Derive is_discontinued from discontinued_date when the column exists; otherwise set is_discontinued = false.
  - Derive is_currently_sellable using sell_start_date and sell_end_date; when discontinued_date exists, also incorporate it into the calculation.
- salesorderheader:
  - Validate order_date before deriving date attributes.
  - Derive order_year, order_month, order_date_key.
  - Derive ship_date_key only when ship_date exists.
- salesorderdetail:
  - Validate order_qty, unit_price, and unit_price_discount before calculations.
  - Derive line_discount_amount = order_qty * unit_price * unit_price_discount.
  - Derive gross_line_amount = order_qty * unit_price.

NOTE:
- User requested ProductModel.Name → modelname. The provided schema contains no Name column on ProductModel. Gold can only expose ProductModelID unless a model-name attribute is added to the source later.

## Gold

Create a business-facing star schema aligned to the requested sales reporting requirements.

Dimensions

1. dim_order_date
- Source: salesorderheader.order_date
- Grain: one row per calendar date.

2. dim_ship_date
- Source: salesorderheader.ship_date
- Grain: one row per calendar date.

3. dim_customer
- User requirement: combine Customer and Address and do not use CustomerAddress as an intermediary.
- Since Customer and Address have no direct join key in the provided schemas, a direct customer-address relationship cannot be reliably established.
- Fallback implementation:
  - Base dimension from Customer.
  - Expose company_name, title, suffix, email_address.

4. dim_salesperson
- Source: customer.sales_person.

5. dim_order
- Source: salesorderheader.

6. dim_product
- Source:
  - Product
  - ProductCategory
  - ProductModelProductDescription
  - ProductDescription
  - ProductModel

Facts

fact_sales_order
- Source:
  - SalesOrderHeader
  - SalesOrderDetail

Regional reporting note:
- Use City as the highest available geography level.

## Test

Store all results in:
- `Tables/test/test_results`

Required tests:
- Row Count Reconciliation
- Gold Dimension PK Not Null
- Gold Dimension PK Uniqueness
- Referential Integrity
- Business Rule Validation

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

## Report

Page 1 — Sales Executive Overview
- KPI cards:
  - Total Sales
  - Average Sales
  - Maximum Sales
  - Total Orders

Page 2 — Geographic Performance
- Use City-based reporting.

Page 3 — Orders & Discounts

Page 4 — Product Performance

Page 5 — Data Quality

## Data Agent

Role:
- Sales Performance Analytics Agent for the SalesLT reporting platform.

Domain Instructions:
- Answer questions using only the semantic model.
- Use City as the available geographic proxy for regional analysis.
- Distinguish gross sales, discount amount, and net sales.

Guardrails:
- Do not invent regions, countries, or territories not present in the model.
- Do not expose PasswordHash or PasswordSalt fields.
- Do not answer with data outside the semantic model.