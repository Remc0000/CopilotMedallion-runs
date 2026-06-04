# Run Spec 20260604-114516-1d5df3

## Updated specs

### Iteration 1 — 2026-06-04 11:52:09Z — failed layer: bronze (run: 20260604-114547-dbff99)
- **Root cause (1-line summary)**: Bronze completed without creating discoverable Delta tables under `Tables/bronze`, causing Silver to find zero upstream inputs and stop.
- **Cross-table audit**: Address: yes — discoverability requirement applies; Customer: yes — discoverability requirement applies; CustomerAddress: yes — discoverability requirement applies; Product: yes — discoverability requirement applies; ProductCategory: yes — discoverability requirement applies; ProductDescription: yes — discoverability requirement applies; ProductModel: yes — discoverability requirement applies; ProductModelProductDescription: yes — discoverability requirement applies; SalesOrderDetail: yes — discoverability requirement applies; SalesOrderHeader: yes — discoverability requirement applies.
- **Fix approach**: GENERALIZE — the failure is not table-specific; every Bronze table must be written to a discoverable Delta location and validated after write.
- **What was changed**:
  - Tightened Bronze output-path requirements to explicitly write each table under `Tables/bronze/<table_name_lower>`.
  - Added mandatory post-write validation that each written table is discoverable and registered in the Bronze output inventory.
  - Added a hard failure if any expected Bronze table is missing or if fewer than 10 Bronze tables are discoverable.

### Iteration 2 — 2026-06-04 11:57:45Z — failed layer: bronze (run: 20260604-114547-dbff99)
- **Root cause (1-line summary)**: Build was interrupted by a server restart mid-execution, requiring Bronze processing to be safely resumable and idempotent.
- **Cross-table audit**: Address: yes — interruption can occur during write; Customer: yes — interruption can occur during write; CustomerAddress: yes — interruption can occur during write; Product: yes — interruption can occur during write; ProductCategory: yes — interruption can occur during write; ProductDescription: yes — interruption can occur during write; ProductModel: yes — interruption can occur during write; ProductModelProductDescription: yes — interruption can occur during write; SalesOrderDetail: yes — interruption can occur during write; SalesOrderHeader: yes — interruption can occur during write.
- **Fix approach**: GENERALIZE — restart resilience must apply uniformly to every Bronze table.
- **What was changed**:
  - Added per-table independent processing and checkpoint-style validation so a restart does not invalidate already completed tables.
  - Required Bronze to re-check existing outputs before rewriting and only consider a table complete after successful read-back validation.
  - Added final inventory reconciliation to ensure all 10 expected Bronze tables exist after a resumed run.

## Inputs
- Workspace: `1f02de75-3d95-4694-b090-c7cecaae69bf`
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
- powerbi-report-authoring: https://github.com/RuiRomano/powerbi-agentic-plugins/tree/main/plugins/powerbi/skills/powerbi-report-authoring

Cross-cutting code rules:
- Use defensive column references and validate existence before use.
- Use alias-prefixed joins only within the join/select scope and materialize flat column names immediately after joins.
- Assert required columns before every join, filter, groupBy, agg, Window, and withColumn operation.
- For REST/API calls: use `if x is None: raise RuntimeError(...)` before any `.get()` access.
- Do not use `saveAsTable`; write Delta directly to managed lakehouse paths.
- Every notebook must begin with parameter cells for workspace, source, target, run_id, layer, and paths.
- Use idempotent overwrite patterns with `overwriteSchema=true`.
- Wrap per-table processing in error-loud try/except blocks that call `_save_error(layer, e)` and re-raise after recording failures.
- Process source tables independently; avoid session-wide dependency chains.
- Every notebook code cell must begin with a short explanatory comment block.
- A layer is not considered successful until its expected output tables are physically discoverable in the target lakehouse location and can be enumerated by the next layer.
- All layers must tolerate notebook interruption or cluster restart by re-validating existing outputs when execution resumes.

### Global Spark column-reference rules (apply to ALL layers: Bronze, Silver, Gold)
Keep all existing Rules A–K exactly as currently specified.

## Bronze

Land each source table unchanged into `Tables/bronze/<table_name_lower>`.

For every table:
- Read directly from the specified source lakehouse table.
- Preserve original source schema and datatypes.
- Add metadata columns:
  - `_run_id`
  - `_ingested_at`
  - `_source_table`
  - `_bronze_ts`
- Write as Delta using overwrite mode with overwriteSchema=true.
- Partition by ingestion date derived from `_ingested_at`.
- The physical Delta output location MUST be discoverable under `Tables/bronze/<table_name_lower>`.
- Process each source table independently so a restart or failure in one table does not invalidate completed tables.
- At the start of processing a table, check whether `Tables/bronze/<table_name_lower>` already exists and is readable as Delta.
- If an existing output passes read-back validation, it may be overwritten idempotently; do not assume prior notebook state exists.
- Immediately after each write, validate:
  - the Delta path exists;
  - the table can be read back successfully;
  - row count is greater than zero unless the source table itself is empty.
- Record the successfully written table name in a Bronze output inventory.
- If any expected Bronze table is not discoverable after write, fail Bronze with an explicit error naming the missing table.
- Before Bronze completes, rebuild or re-read the Bronze output inventory from the physical lakehouse paths and assert that all 10 expected Bronze tables listed below are discoverable.
- Do not report Bronze success if fewer than 10 tables are available.

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

Primary keys identified:
- Address: AddressID
- Customer: CustomerID
- CustomerAddress: (CustomerID, AddressID)
- Product: ProductID
- ProductCategory: ProductCategoryID
- ProductDescription: ProductDescriptionID
- ProductModel: ProductModelID
- ProductModelProductDescription: (ProductModelID, ProductDescriptionID, Culture)
- SalesOrderHeader: SalesOrderID
- SalesOrderDetail: SalesOrderDetailID

## Silver

Standard processing:
- Convert all column names to snake_case.
- Preserve business keys.
- Add:
  - `_silver_ts`
  - `_record_source`
  - `_run_id`
- Remove exact duplicates.
- Apply table-specific deduplication.
- OPTIMIZE and V-ORDER after write.

Deduplication strategy:
- silver.address → AddressID
- silver.customer → CustomerID
- silver.customeraddress → (CustomerID, AddressID)
- silver.product → ProductID
- silver.productcategory → ProductCategoryID
- silver.productdescription → ProductDescriptionID
- silver.productmodel → ProductModelID
- silver.productmodelproductdescription → (ProductModelID, ProductDescriptionID, Culture)
- silver.salesorderheader → SalesOrderID
- silver.salesorderdetail → SalesOrderDetailID

Business cleansing:
- Parse customer.sales_person into normalized username.
- If value follows `<domain>\username`, keep only username.
- Remove password_hash and password_salt from downstream Gold outputs.
- Create product status flags:
  - is_discontinued
  - is_active_for_sale
- Create order lifecycle flags:
  - is_shipped
  - is_open_order

Important modeling note:
- User requested ProductModel.Name as modelname, but ProductModel contains only ProductModelID, rowguid, and ModifiedDate. No Name column exists in the provided schema. Gold Product dimension will therefore include ProductModelID but cannot expose modelname unless the source schema is expanded.

## Gold

Target star schema for sales analytics.

Dimension: dim_order_date
- Source: SalesOrderHeader.OrderDate

Dimension: dim_ship_date
- Source: SalesOrderHeader.ShipDate

Dimension: dim_customer
- Use CustomerAddress internally to relate Customer and Address.
- Keep:
  - customer_id
  - company_name
  - sales_person_username
  - email_address
  - title
  - city
  - postal_code

Dimension: dim_salesperson
- Source: Customer.SalesPerson

Dimension: dim_order
- Source: SalesOrderHeader

Dimension: dim_product
- Source combination:
  - Product
  - ProductCategory
  - ProductModelProductDescription
  - ProductDescription
  - ProductModel
- Filter ProductModelProductDescription to Culture='en'.

Fact: fact_sales_order
- Grain: one row per SalesOrderDetail line.
- Source: SalesOrderHeader joined to SalesOrderDetail.

## Test

All tests append results into:
- gold.test_results

Tests:
- Row count reconciliation
- Gold dimension PK null check
- Gold dimension PK uniqueness
- Referential integrity
- Business-rule validation

## Semantic model

Mode:
- Direct Lake

Tables:
- fact_sales_order
- dim_order_date
- dim_ship_date
- dim_customer
- dim_salesperson
- dim_product
- dim_order

Relationships:
- fact_sales_order → dim_order_date
- fact_sales_order → dim_ship_date
- fact_sales_order → dim_customer
- fact_sales_order → dim_salesperson
- fact_sales_order → dim_product
- fact_sales_order → dim_order

Measures:
- Total Sales = SUM(net_sales_amount)
- Gross Sales = SUM(extended_amount)
- Total Discount Amount = SUM(discount_amount)
- Average Sales = AVERAGE(net_sales_amount)
- Maximum Sale = MAX(net_sales_amount)
- Total Orders = DISTINCTCOUNT(order_key)
- Total Quantity = SUM(order_qty)
- Average Discount % = AVERAGE(unit_price_discount) * 100
- Maximum Discount % = MAX(unit_price_discount) * 100
- Average Order Value = DIVIDE([Total Sales],[Total Orders])
- Sales per Customer = DIVIDE([Total Sales], DISTINCTCOUNT(customer_key))

## Report

Page 1: Executive Sales Overview
- KPI cards:
  - Total Sales
  - Average Sales
  - Maximum Sale
  - Total Orders

Page 2: Regional Performance

Page 3: Orders & Discounts

Page 4: Product Performance

Page 5: Data Quality

## Data Agent

Role:
- AI Sales Performance Analyst for the SalesLT sales reporting solution.

Domain instructions:
- Answer questions only using the semantic model.
- Prioritize business outcomes, sales performance, customer trends, regional performance, product performance, and discount analysis.
- Always use measures rather than raw aggregation when available.
- Highlight data quality issues when test results indicate failures.
- If category names or product model names are unavailable, explain that the source data only contains IDs.

Guardrails:
- Do not fabricate missing category names or model names.
- Do not answer using data outside the semantic model.
- Distinguish between averages, maxima, and totals.
- State filters and time periods used in every analytical answer.
- Surface uncertainty when source attributes are unavailable.
- Respect row-level aggregation and avoid exposing sensitive source fields such as password hashes or salts.