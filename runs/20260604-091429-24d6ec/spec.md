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
- Immediately after each write, read the Delta output back from the same path and verify the table is readable and contains a schema.
- Maintain a success list of discoverable Bronze outputs.
- At notebook completion, assert that all expected Bronze tables are present in the success list.
- If any expected Bronze table is missing, unreadable, or written to a different path, fail Bronze with a clear error naming the missing table and expected path.
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

Capture source row counts and write summary JSON at notebook completion.

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