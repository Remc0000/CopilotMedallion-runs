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

### Iteration 4 — 2026-06-04 09:28:26Z — failed layer: bronze (run: 20260604-091429-24d6ec)
- **Root cause (1-line summary)**: Silver could not discover any Bronze tables because Bronze outputs were not exposed as Lakehouse Tables under the expected `Tables/bronze/<table_name>` hierarchy.
- **Cross-table audit**:
  - SalesLT/Address: yes — requires discoverable Lakehouse table path.
  - SalesLT/Customer: yes — requires discoverable Lakehouse table path.
  - SalesLT/CustomerAddress: yes — requires discoverable Lakehouse table path.
  - SalesLT/Product: yes — requires discoverable Lakehouse table path.
  - SalesLT/ProductCategory: yes — requires discoverable Lakehouse table path.
  - SalesLT/ProductDescription: yes — requires discoverable Lakehouse table path.
  - SalesLT/ProductModel: yes — requires discoverable Lakehouse table path.
  - SalesLT/ProductModelProductDescription: yes — requires discoverable Lakehouse table path.
  - SalesLT/SalesOrderDetail: yes — requires discoverable Lakehouse table path.
  - SalesLT/SalesOrderHeader: yes — requires discoverable Lakehouse table path.
- **Fix approach**: GENERALIZE — the discoverability failure affects every Bronze table identically.
- **What was changed**:
  - Required writes to resolve against the target Lakehouse physical Tables area, not relative or notebook-local paths.
  - Added mandatory post-write enumeration of `Tables/bronze/` and validation that all ten expected table directories are present before Bronze succeeds.
  - Added a hard failure if the final discovered table count under `Tables/bronze/` is not exactly ten.

### Iteration 5 — 2026-06-04 09:31:49Z — failed layer: bronze (run: 20260604-091429-24d6ec)
- **Root cause (1-line summary)**: Server restarted mid-build; Bronze execution must resume from durable state without reprocessing already validated tables.
- **Cross-table audit**:
  - SalesLT/Address: yes — restart can occur after write or validation.
  - SalesLT/Customer: yes — restart can occur after write or validation.
  - SalesLT/CustomerAddress: yes — restart can occur after write or validation.
  - SalesLT/Product: yes — restart can occur after write or validation.
  - SalesLT/ProductCategory: yes — restart can occur after write or validation.
  - SalesLT/ProductDescription: yes — restart can occur after write or validation.
  - SalesLT/ProductModel: yes — restart can occur after write or validation.
  - SalesLT/ProductModelProductDescription: yes — restart can occur after write or validation.
  - SalesLT/SalesOrderDetail: yes — restart can occur after write or validation.
  - SalesLT/SalesOrderHeader: yes — restart can occur after write or validation.
- **Fix approach**: GENERALIZE — restart recovery requirements are identical for all Bronze source tables.
- **What was changed**:
  - Required Bronze processing to be table-at-a-time with immediate durable checkpoint persistence after each successful table.
  - Required notebook startup recovery to rebuild state from physical Delta validation and manifest records rather than in-memory variables.
  - Added prohibition on failing the run solely because previously completed tables already exist.

### Iteration 6 — 2026-06-04 09:32:24Z — failed layer: bronze (run: 20260604-091429-24d6ec)
- **Root cause (1-line summary)**: Repeated server restarts interrupted Bronze before completion; recovery state must be stored in durable Lakehouse artifacts and flushed after every table.
- **Cross-table audit**:
  - SalesLT/Address: yes — recovery metadata must survive restart.
  - SalesLT/Customer: yes — recovery metadata must survive restart.
  - SalesLT/CustomerAddress: yes — recovery metadata must survive restart.
  - SalesLT/Product: yes — recovery metadata must survive restart.
  - SalesLT/ProductCategory: yes — recovery metadata must survive restart.
  - SalesLT/ProductDescription: yes — recovery metadata must survive restart.
  - SalesLT/ProductModel: yes — recovery metadata must survive restart.
  - SalesLT/ProductModelProductDescription: yes — recovery metadata must survive restart.
  - SalesLT/SalesOrderDetail: yes — recovery metadata must survive restart.
  - SalesLT/SalesOrderHeader: yes — recovery metadata must survive restart.
- **Fix approach**: GENERALIZE — the same restart-resilience pattern applies to every Bronze table.
- **What was changed**:
  - Required a durable checkpoint/manifest stored in the target Lakehouse after every successful table write and validation.
  - Required startup logic to enumerate all expected Bronze paths and rebuild progress solely from persisted artifacts and Delta validation.
  - Required committing and validating one table at a time before moving to the next table.

### Iteration 6 — 2026-06-04 09:36:44Z — failed layer: bronze (run: 20260604-091429-24d6ec)
- **Root cause (1-line summary)**: Silver detected zero discoverable Bronze tables because Bronze outputs were not visible under the exact `Tables/bronze/` hierarchy at runtime.
- **Cross-table audit**:
  - SalesLT/Address: yes — must be discoverable through directory enumeration and Delta read.
  - SalesLT/Customer: yes — must be discoverable through directory enumeration and Delta read.
  - SalesLT/CustomerAddress: yes — must be discoverable through directory enumeration and Delta read.
  - SalesLT/Product: yes — must be discoverable through directory enumeration and Delta read.
  - SalesLT/ProductCategory: yes — must be discoverable through directory enumeration and Delta read.
  - SalesLT/ProductDescription: yes — must be discoverable through directory enumeration and Delta read.
  - SalesLT/ProductModel: yes — must be discoverable through directory enumeration and Delta read.
  - SalesLT/ProductModelProductDescription: yes — must be discoverable through directory enumeration and Delta read.
  - SalesLT/SalesOrderDetail: yes — must be discoverable through directory enumeration and Delta read.
  - SalesLT/SalesOrderHeader: yes — must be discoverable through directory enumeration and Delta read.
- **Fix approach**: GENERALIZE — the failure affects all Bronze tables uniformly because downstream discovery scans the same `Tables/bronze/` hierarchy.
- **What was changed**:
  - Added a mandatory Bronze completion gate requiring successful enumeration and Delta reads of all ten expected paths before Bronze can finish.
  - Required the notebook to fail immediately if `Tables/bronze/` contains fewer than ten expected directories.
  - Required the completion summary JSON to include a discovered-table count and per-table discoverability status.

### Iteration 7 — 2026-06-04 09:40:44Z — failed layer: bronze (run: 20260604-091429-24d6ec)
- **Root cause (1-line summary)**: Silver found zero discoverable Bronze tables, indicating Bronze validation was not using the same table-discovery mechanism as downstream layer startup.
- **Cross-table audit**:
  - SalesLT/Address: yes — downstream discovery requires a readable Delta table at the expected path.
  - SalesLT/Customer: yes — downstream discovery requires a readable Delta table at the expected path.
  - SalesLT/CustomerAddress: yes — downstream discovery requires a readable Delta table at the expected path.
  - SalesLT/Product: yes — downstream discovery requires a readable Delta table at the expected path.
  - SalesLT/ProductCategory: yes — downstream discovery requires a readable Delta table at the expected path.
  - SalesLT/ProductDescription: yes — downstream discovery requires a readable Delta table at the expected path.
  - SalesLT/ProductModel: yes — downstream discovery requires a readable Delta table at the expected path.
  - SalesLT/ProductModelProductDescription: yes — downstream discovery requires a readable Delta table at the expected path.
  - SalesLT/SalesOrderDetail: yes — downstream discovery requires a readable Delta table at the expected path.
  - SalesLT/SalesOrderHeader: yes — downstream discovery requires a readable Delta table at the expected path.
- **Fix approach**: GENERALIZE — every Bronze table must pass the exact same discovery checks used by Silver.
- **What was changed**:
  - Added a mandatory Bronze self-discovery phase that re-enumerates `Tables/bronze/` from a fresh read after all writes complete.
  - Required validation against the exact ten expected table names and paths before Bronze can report success.
  - Required Bronze to fail if self-discovery returns fewer than ten tables, regardless of manifest contents.

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
- Write each table to the target LakeSales Lakehouse physical Tables area using the exact path `Tables/bronze/<table_name>`.
- Resolve paths against the attached target Lakehouse; do not use relative filesystem locations, local disk, temporary storage, or alternate lakehouses.
- Do not write Bronze outputs to `Files/`, temporary folders, notebook-local storage, alternative casing, or any path other than the exact locations listed below.
- Process exactly one source table at a time and persist its completion manifest immediately after validation succeeds.
- Store the Bronze manifest/checkpoint in a durable Lakehouse location that survives Spark session loss and notebook restarts.
- Flush and persist manifest updates immediately after each individual table is validated; do not defer manifest creation until notebook completion.
- On notebook startup, enumerate all expected Bronze table paths and rebuild completion state exclusively from the durable Bronze manifest plus physical Delta validation of existing `Tables/bronze/<table_name>` locations.
- If a target table already exists, has a `_delta_log`, is readable, and passes validation, mark it complete and continue instead of rewriting it.
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
- Perform a fresh self-discovery pass after all writes complete; do not rely on cached filesystem results, manifests, or in-memory success lists.
- The discovered directories under `Tables/bronze/` must be exactly:
  - address
  - customer
  - customeraddress
  - product
  - productcategory
  - productdescription
  - productmodel
  - productmodelproductdescription
  - salesorderdetail
  - salesorderheader
- Assert that all expected Bronze table directories exist and are readable from their exact paths.
- Rebuild the final success list from physical Delta validation rather than in-memory state so a notebook restart cannot lose progress.
- Before declaring Bronze successful, assert that the discovered table count under `Tables/bronze/` equals 10.
- Before declaring Bronze successful, perform a Delta read of every expected path and persist the results in the completion summary.
- Fail Bronze immediately if the discovered table count is less than 10, even if manifest records indicate success.
- The completion summary JSON must include: expected_table_count, discovered_table_count, and per-table discoverable=true/false status.
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

Capture source row counts and write summary JSON at notebook completion. The summary must include the exact Bronze path, validation status, row count for every expected table, discovered-table count, and discoverability status.

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