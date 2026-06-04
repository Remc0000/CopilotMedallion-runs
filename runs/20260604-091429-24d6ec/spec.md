# Run Spec 20260604-091237-ae5763

## Updated specs

### Iteration 1 — 2026-06-04 09:18:05Z — failed layer: bronze (run: 20260604-091429-24d6ec)
- **Root cause (1-line summary)**: Bronze completed without producing discoverable Delta tables under `Tables/bronze/`, causing Silver to halt because no Bronze outputs were found.
- **Cross-table audit**:
  - SalesLT/Address: yes — Bronze discoverability requirements apply.
  - SalesLT/Customer: yes — Bronze discoverability requirements apply.
  - SalesLT/CustomerAddress: yes — Bronze discoverability requirements apply.
  - SalesLT/Product: yes — Bronze discoverability requirements apply.
  - SalesLT/ProductCategory: yes — Bronze discoverability requirements apply.
  - SalesLT/ProductDescription: yes — Bronze discoverability requirements apply.
  - SalesLT/ProductModel: yes — Bronze discoverability requirements apply.
  - SalesLT/ProductModelProductDescription: yes — Bronze discoverability requirements apply.
  - SalesLT/SalesOrderDetail: yes — Bronze discoverability requirements apply.
  - SalesLT/SalesOrderHeader: yes — Bronze discoverability requirements apply.
- **Fix approach**: GENERALIZE — the failure is not table-specific; every Bronze table must be written and validated using the same discoverability rules.
- **What was changed**:
  - Tightened Bronze write requirements to require physical Delta outputs at `Tables/bronze/<table>`.
  - Added post-write validation that each Bronze table is immediately discoverable and readable.
  - Added a mandatory layer-level failure if any expected Bronze table is missing after writes complete.

### Iteration 2 — 2026-06-04 09:21:52Z — failed layer: bronze (run: 20260604-091429-24d6ec)
- **Root cause (1-line summary)**: Silver again found no discoverable Bronze outputs, indicating Bronze writes were not materialized at the exact Lakehouse `Tables/bronze/<table>` locations expected by downstream discovery.
- **Cross-table audit**:
  - SalesLT/Address: yes — must exist as a physical Delta table under the exact Bronze path.
  - SalesLT/Customer: yes — must exist as a physical Delta table under the exact Bronze path.
  - SalesLT/CustomerAddress: yes — must exist as a physical Delta table under the exact Bronze path.
  - SalesLT/Product: yes — must exist as a physical Delta table under the exact Bronze path.
  - SalesLT/ProductCategory: yes — must exist as a physical Delta table under the exact Bronze path.
  - SalesLT/ProductDescription: yes — must exist as a physical Delta table under the exact Bronze path.
  - SalesLT/ProductModel: yes — must exist as a physical Delta table under the exact Bronze path.
  - SalesLT/ProductModelProductDescription: yes — must exist as a physical Delta table under the exact Bronze path.
  - SalesLT/SalesOrderDetail: yes — must exist as a physical Delta table under the exact Bronze path.
  - SalesLT/SalesOrderHeader: yes — must exist as a physical Delta table under the exact Bronze path.
- **Fix approach**: GENERALIZE — the same discoverability requirement applies uniformly to every Bronze table.
- **What was changed**:
  - Added explicit prohibition on writing Bronze outputs anywhere except the listed `Tables/bronze/<table>` locations.
  - Added mandatory Delta-log validation and directory enumeration after each write.
  - Added a final Bronze manifest check requiring all expected table directories to exist before the layer can succeed.

### Iteration 3 — 2026-06-04 09:25:11Z — failed layer: bronze (run: 20260604-091429-24d6ec)
- **Root cause (1-line summary)**: Build interruption due to server restart during Bronze execution; partial writes must be handled safely and resumably.
- **Cross-table audit**:
  - SalesLT/Address: yes — interruption can occur during write or validation.
  - SalesLT/Customer: yes — interruption can occur during write or validation.
  - SalesLT/CustomerAddress: yes — interruption can occur during write or validation.
  - SalesLT/Product: yes — interruption can occur during write or validation.
  - SalesLT/ProductCategory: yes — interruption can occur during write or validation.
  - SalesLT/ProductDescription: yes — interruption can occur during write or validation.
  - SalesLT/ProductModel: yes — interruption can occur during write or validation.
  - SalesLT/ProductModelProductDescription: yes — interruption can occur during write or validation.
  - SalesLT/SalesOrderDetail: yes — interruption can occur during write or validation.
  - SalesLT/SalesOrderHeader: yes — interruption can occur during write or validation.
- **Fix approach**: GENERALIZE — restart resilience is required uniformly for every Bronze table.
- **What was changed**:
  - Added per-table checkpoint and completion-manifest requirements.
  - Added restart-safe validation to reuse already validated Bronze outputs.
  - Required final manifest verification before Bronze success is declared.

## Inputs
- Workspace: `44cf7adf-0561-49b1-bf73-44f3c5b38c21`
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
- Target Lakehouse: **LakeSales**

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
- Alias-prefix joins only inside the join expression; materialize flat column names immediately after joins.
- Assert all groupBy and aggregation columns exist before execution.
- Use defensive REST handling with `if x is None: raise RuntimeError(...)` before any `.get()` access.
- Never use `saveAsTable`; write Delta directly to lakehouse paths.
- Every notebook must start with parameter cells for workspace, lakehouse, run_id, paths, and execution options.
- Use idempotent overwrite patterns with `.mode("overwrite").option("overwriteSchema","true")`.
- All exception handling must be error-loud, call `_save_error(layer, e)` (or `_save_error(layer, e, table=tbl)` in loops), and re-raise after logging.
- Process source tables independently per layer using per-table isolation patterns.
- Every generated notebook cell must begin with a short comment block describing intent.
- After every write, validate that the target Delta location exists and is readable before marking the table successful.

### Global Spark column-reference rules (apply to ALL layers: Bronze, Silver, Gold)

All Rules A–K from the current specification remain in force.

## Bronze

Land each source table unchanged into `bronze` with original schema preserved plus:
- `_ingested_at` timestamp
- `_run_id`
- `_source_table`
- `_bronze_ts`

Write pattern:
- Delta format
- Overwrite mode with schema overwrite enabled
- One table per source entity
- Partition by ingestion date derived from `_ingested_at`

Mandatory discoverability requirements:
- Write each table to a physical Delta path under `Tables/bronze/<table_name>` using the exact table names listed below.
- Do not write Bronze outputs to `Files/`, temporary folders, notebook-local storage, alternative casing, or any path other than the exact locations listed below.
- Immediately after each write, verify:
  - the target directory exists;
  - a `_delta_log` directory exists beneath the target directory;
  - a Delta read from the same path succeeds;
  - the returned schema is non-empty;
  - the row count is greater than or equal to zero.
- Maintain a success list containing only tables that pass all validations above.
- After a table passes validation, persist a completion record in a Bronze manifest/checkpoint artifact containing table name, path, row count, validation status, and run_id.
- On restart or rerun, if a table already exists at the expected path and passes all validation checks, treat it as completed and do not require re-ingestion before continuing with remaining tables.
- At notebook completion, enumerate the contents of `Tables/bronze/` and compare against the expected table list.
- Assert that all expected Bronze table directories exist and are readable from their exact paths.
- Rebuild the final success list from physical Delta validation rather than in-memory state so a notebook restart cannot lose progress.
- If any expected Bronze table is missing, unreadable, lacks a `_delta_log`, or is written to a different path, fail Bronze with a clear error naming the table and expected path.
- Do not rely solely on metadata registration; physical Delta files under `Tables/bronze/<table_name>` must exist.

Bronze tables:
- bronze.address → `Tables/bronze/address`
- bronze.customer → `Tables/bronze/customer`
- bronze.customeraddress → `Tables/bronze/customeraddress`
- bronze.product → `Tables/bronze/product`
- bronze.productcategory → `Tables/bronze/productcategory`
- bronze.productdescription → `Tables/bronze/productdescription`
- bronze.productmodel → `Tables/bronze/productmodel`
- bronze.productmodelproductdescription → `Tables/bronze/productmodelproductdescription`
- bronze.salesorderdetail → `Tables/bronze/salesorderdetail`
- bronze.salesorderheader → `Tables/bronze/salesorderheader`

Capture source row counts and write summary JSON at notebook completion. The summary must include the exact Bronze path, validation status, and row count for every expected table.

## Silver

Standardize all tables:
- Convert column names to snake_case.
- Preserve business keys.
- Add `_silver_ts`, `_run_id`, `_record_source`.
- Remove exact duplicates.
- Apply deterministic deduplication using latest modified_date when available.

Table-specific deduplication keys:
- silver.address: `address_id`
- silver.customer: `customer_id`
- silver.customeraddress: (`customer_id`, `address_id`)
- silver.product: `product_id`
- silver.productcategory: `product_category_id`
- silver.productdescription: `product_description_id`
- silver.productmodel: `product_model_id`
- silver.productmodelproductdescription: (`product_model_id`, `product_description_id`, `culture`)
- silver.salesorderheader: `sales_order_id`
- silver.salesorderdetail: (`sales_order_id`, `sales_order_detail_id`)

Business transformations:
- Customer:
  - Normalize email address casing.
  - Trim text fields.
- Product:
  - Derive `is_discontinued`.
  - Derive `is_currently_sellable` from sell dates and discontinued status.
- SalesOrderHeader:
  - Derive order_year, order_month, order_date_key.
  - Derive ship_date_key when ship_date exists.
- ProductModelProductDescription:
  - Filter English records (`culture = 'en'`) for downstream product dimension use while retaining full silver table.

Optimize and vacuum silver outputs after successful write.

## Gold

Target star schema requested by user.

Dimension: `dim_order_date`
- Source: SalesOrderHeader.order_date
- Grain: one row per calendar date.

Dimension: `dim_ship_date`
- Source: SalesOrderHeader.ship_date.
- Grain: one row per calendar date.

Dimension: `dim_customer`
- Source: Customer + Address.
- User requested bypassing CustomerAddress.
- NOTE: Customer and Address have no direct key relationship in the supplied schema. A valid customer-address link only exists through CustomerAddress.
- Implementation recommendation: build dim_customer from Customer only for guaranteed correctness.

Dimension: `dim_salesperson`
- Source: Customer.sales_person.

Dimension: `dim_order`
- Source: SalesOrderHeader.

Dimension: `dim_product`
- Source:
  - Product
  - ProductCategory
  - ProductModel
  - ProductModelProductDescription (culture='en')
  - ProductDescription

Fact: `fact_sales_order`
- Source:
  - SalesOrderHeader
  - SalesOrderDetail

Regional-performance requirement:
- Region data is not explicitly present.
- Closest available geography is Address.city and postal_code.

## Test

Write all results into:
- `test/test_results`

Required tests:
1. Row count reconciliation
2. Gold dimension PK not null
3. Gold dimension PK uniqueness
4. Referential integrity
5. Business-rule sanity checks

## Semantic model

Mode:
- Direct Lake

Tables:
- fact_sales_order
- dim_product
- dim_customer
- dim_salesperson
- dim_order
- dim_order_date
- dim_ship_date

Relationships:
- fact_sales_order ↔ dim_product
- fact_sales_order ↔ dim_customer
- fact_sales_order ↔ dim_salesperson
- fact_sales_order ↔ dim_order
- fact_sales_order ↔ dim_order_date
- fact_sales_order ↔ dim_ship_date

## Report

Page 1: Executive Sales Overview
- KPI cards
- Monthly sales trend
- Sales by product category
- Top products

Page 2: Regional Performance
- Map visuals
- City ranking
- Geographic analysis

Page 3: Orders and Discounts
- Order trends
- Salesperson analysis
- Discount analysis

Page 4: Product Performance
- Category hierarchy analysis
- Product profitability

Page 5: Data Quality
- Test results summary
- Failed-test details

## Data Agent

Role:
- Sales Performance Intelligence Agent for LakeSales.

Domain instructions:
- Answer questions using only the semantic model.
- Focus on sales performance, order behavior, discount trends, product performance, customer activity, salesperson effectiveness, and geographic analysis.

Guardrails:
- Do not invent regions, territories, countries, or states not present in the model.
- Use only published semantic-model tables and measures.
- Distinguish clearly between gross sales and net sales.
- Flag data-quality issues if test results indicate failures.