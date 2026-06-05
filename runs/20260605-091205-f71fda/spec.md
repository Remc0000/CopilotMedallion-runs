# Run Spec 20260605-091115-6334db

## Inputs
- Workspace: `423348df-cb05-4fa0-bb36-72cf92932692`
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
- Target Lakehouse: **q**

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
- Use defensive column references and validate required columns before every transformation.
- Alias-qualify all join columns inside join projections and immediately rename to flat names.
- Assert column existence before every groupBy/agg.
- For REST/API responses use `if x is None: raise` before any `.get()`.
- Create schemas with `CREATE SCHEMA IF NOT EXISTS`.
- Write all outputs with schema-qualified `saveAsTable('<schema>.<table>')`.
- Never write target lakehouse outputs using raw abfss `.save()`.
- Use parameter cells for workspace, lakehouse, schema, table names, and run_id.
- Use idempotent overwrite patterns with `overwriteSchema=true`.
- Use error-loud try/except blocks that call `_save_error(layer, e)` and re-raise.
- Process source tables independently with per-table error isolation.
- All notebooks must emit discoverable Delta tables in bronze, silver, gold, and test schemas.
- Every code cell must begin with a short explanatory comment block using `# ---`.

### Global Spark column-reference rules (apply to ALL layers: Bronze, Silver, Gold)
These rules exist to prevent recurring `UNRESOLVED_COLUMN` / `AnalysisException` analyzer errors. They are layer-agnostic — apply them anywhere a Spark DataFrame is transformed.

Rule A — No dotted alias strings.
- Never pass dotted strings like "c.customer_id", "ca.address_type", "h.sales_person", or "pc_child.name" to F.col(...), withColumn(...), Window.partitionBy(...), Window.orderBy(...), or select(...). Spark treats "c.customer_id" as a single column literally named c.customer_id, which does not resolve once any projection or rename has been applied.
- Alias scope (.alias("c"), .alias("ca"), ...) is only valid inside the SAME select / join expression that introduces it. Once you produce a new DataFrame via select(...) or withColumn(...), the dotted alias form is gone and you must reference plain column names.

Rule B — Materialize helper columns before they are needed downstream.
- For any column that will later be referenced by a Window, a withColumn, or a downstream join after a projection, first materialize it as a flat, unambiguous helper column (e.g. rank_customer_id, rank_address_type, sales_person_source) in the same select that introduces the join aliases.

Rule C — Do not drop a column before its last consumer has run.
- Before adding a withColumn, verify every F.col(...) referenced by that expression still exists on the DataFrame at that step. If a previous select(...) projection removed it, either:
  - (preferred) move the withColumn BEFORE the projection that drops the source column, OR
  - keep the source column in the projection, OR
  - re-derive the value from a column that IS still present (often a boolean/flag that was computed earlier from the same source).
- Example of the failure to avoid: dropping discontinued_date in a select(...) and then later writing F.when(F.col('discontinued_date').isNotNull(), ...) inside withColumn('is_sellable_currently', ...). The column is gone and Spark raises UNRESOLVED_COLUMN.
- When a boolean flag derived from a raw column already exists on the DataFrame (e.g. is_discontinued derived from discontinued_date), prefer reusing the flag (F.col('is_discontinued')) over re-reading the dropped raw column.

Rule D — Order of derived-column computations matters.
- When building several derived columns where one depends on another (e.g. is_discontinued, then is_sellable_currently which uses is_discontinued), add them in dependency order with sequential withColumn calls, and reference the already-derived flag in the next expression — do NOT reach back to a raw source column that may have been dropped.

Rule E — Validate schema between non-trivial transformation steps.
- After any select(...) / drop(...) / heavy withColumn chain, and BEFORE the next step that depends on specific columns, assert those columns exist. Fail fast with an error message that names the missing column and the DataFrame variable, so the auto-fixer gets an actionable diagnostic instead of a deep analyzer stack trace.

Rule F — Self-check pattern for every withColumn / Window.
- For every withColumn(name, expr) and every Window definition, confirm: "Every column referenced inside expr / inside the window's partitionBy / orderBy exists on the DataFrame at this exact point." If not, fix per Rule C before generating the code.

Rule G — Optional-column helpers must return typed Column nulls, not Python None.
- When defining a helper like `_maybe(df, name)` that returns the column if it exists on the DataFrame and a fallback otherwise, NEVER return Python `None`. Spark functions (`F.coalesce`, `F.greatest`, `F.least`, `F.concat`, `F.when(...).otherwise(...)`, etc.) reject `None` arguments with `PySparkTypeError: [NOT_COLUMN_OR_STR]` and the cell crashes BEFORE any later fallback (e.g. `F.current_timestamp()`) gets a chance to satisfy the call.
- Correct pattern — return a typed null literal as a Spark Column.
- Pick the dtype to match the surrounding expression.
- Never pass Python `None` directly into a Spark function.

Rule H — Per-table isolation; one table's failure must not cancel the Spark session for the rest.
- Process each source table independently.
- Capture success/failure results per table.
- Save errors and continue where appropriate.
- Perform cross-table joins only after source outputs exist.

Rule I — Optional audit columns on junction / bridge / view tables.
- Do not assume ModifiedDate exists on every table.
- Junction tables must deduplicate on composite business keys.
- Use guarded audit logic.

Rule J — Validate column existence BEFORE the expensive transform.
- Assert required columns before joins, filters, aggregations, and derived columns.
- Emit actionable errors including layer and table names.

Rule K — Resilience to partial output: every layer MUST write Delta tables the next layer can discover.
- Bronze writes to `bronze.<table>`.
- Silver writes to `silver.<table>`.
- Gold writes to `gold.<table>`.
- Test writes to `test.test_results`.
- Raise an error if no discoverable tables are produced.

Rule L — Disambiguate shared columns in join projections (avoid AMBIGUOUS_REFERENCE).
- Alias-qualify shared columns in joins.
- Rename projected columns immediately.
- After projection, use flat column names only.

ALSO REQUIRE for every generated notebook: EACH code cell must start with a short markdown comment block (Python `# ---` divider + 1-3 lines of `# ` comments) describing what the cell is doing and why.

## Bronze

Land each source table 1:1 into the `bronze` schema with minimal transformation.

Tables:
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

Standard metadata:
- ingestion_ts
- source_table
- run_id
- source_file_or_object
- bronze_load_ts

Write pattern:
- Delta format
- overwrite mode with overwriteSchema=true
- saveAsTable into bronze schema

Partitioning:
- salesorderheader: partition by order year derived from OrderDate
- salesorderdetail: partition by SalesOrderID hash bucket or unpartitioned if volume is small
- Remaining master tables: unpartitioned

Preserve all source columns including:
- rowguid
- ModifiedDate
- binary thumbnail content in Product

## Silver

Common standards:
- Convert all columns to snake_case.
- Trim strings and normalize empty strings to null where appropriate.
- Preserve business keys.
- Add silver_load_ts and source_dt.
- Deduplicate using latest modified_date when available.
- OPTIMIZE and V-ORDER after writes.

Silver tables and deduplication:

- silver.address
  - PK: address_id
  - Dedup key: address_id

- silver.customer
  - PK: customer_id
  - Dedup key: customer_id
  - Create cleaned salesperson_source from sales_person

- silver.customeraddress
  - Composite PK: customer_id, address_id
  - Dedup key: customer_id + address_id

- silver.product
  - PK: product_id
  - Dedup key: product_id
  - Derive is_discontinued from discontinued_date
  - Derive is_active_product

- silver.productcategory
  - PK: product_category_id
  - Dedup key: product_category_id

- silver.productdescription
  - PK: product_description_id
  - Dedup key: product_description_id

- silver.productmodel
  - PK: product_model_id
  - Dedup key: product_model_id
  - NOTE: schema contains only ProductModelID, rowguid, ModifiedDate. Requested ProductModel.Name does not exist in supplied source schema. Gold layer will use ProductModelID only unless an additional name column is later provided.

- silver.productmodelproductdescription
  - Composite PK: product_model_id, product_description_id, culture
  - Filter support retained for gold layer.
  - Dedup key: product_model_id + product_description_id + culture

- silver.salesorderheader
  - PK: sales_order_id
  - Dedup key: sales_order_id

- silver.salesorderdetail
  - PK: sales_order_detail_id
  - Dedup key: sales_order_detail_id

## Gold

Target star schema aligned to requested sales-reporting solution.

Dimensions:

- gold.dim_order_date
  - Source: salesorderheader.order_date
  - Key: date_key
  - Attributes: date, day, month, month_name, quarter, year, fiscal attributes
  - Hierarchy: Year > Quarter > Month > Date

- gold.dim_ship_date
  - Source: salesorderheader.ship_date
  - Key: ship_date_key
  - Attributes: date hierarchy fields
  - Hierarchy: Year > Quarter > Month > Date

- gold.dim_customer
  - Source: customer joined directly to address as requested
  - Preferred join:
    - customer.customer_id
    - salesorderheader.customer_id
    - salesorderheader.bill_to_address_id
  - Do not use customeraddress bridge.
  - Attributes:
    - customer_id
    - company_name
    - title
    - suffix
    - email_address
    - city
    - postal_code
  - Keep only reporting-relevant fields.
  - Regional analysis will use city and postal_code because no state/province/country fields exist in the supplied schema.

- gold.dim_salesperson
  - Source: customer.sales_person
  - Extract username portion from values formatted as `domain\username`.
  - Example:
    - adventure-works\jillian0 → jillian0
  - Attributes:
    - salesperson_key
    - salesperson_name
    - original_salesperson
  - Deduplicate unique salespeople.
  - Hierarchy not applicable.

- gold.dim_order
  - Source: salesorderheader
  - Key: sales_order_id
  - Attributes:
    - revision_number
    - status
    - ship_method
    - comment
  - Exclude measures and foreign-key references that belong in the fact.

- gold.dim_product
  - Source:
    - product
    - productcategory
    - productmodelproductdescription
    - productdescription
    - productmodel
  - Filter productmodelproductdescription to culture='en'
  - Parent-child category flattening:
    - category_id
    - category_name surrogate not available in source
    - parent_product_category_id
    - category_level
  - Include:
    - product_id
    - product_number
    - color
    - size
    - weight
    - standard_cost
    - list_price
    - description
    - product_model_id
    - active/discontinued indicators
  - NOTE:
    - ProductCategory table contains IDs only; category names are not available.
    - ProductModel.Name requested by user is not available in supplied schema.
    - Product dimension will expose available identifiers and description data.

Fact:

- gold.fact_sales_order
  - Grain: one sales order line item
  - Source:
    - salesorderheader
    - salesorderdetail
  - Join:
    - sales_order_id
  - Foreign keys:
    - order_date_key
    - ship_date_key
    - customer_key
    - salesperson_key
    - order_key
    - product_key
  - Measures retained:
    - order_qty
    - unit_price
    - unit_price_discount
    - extended_amount = order_qty * unit_price
    - discount_amount = order_qty * unit_price * unit_price_discount
    - net_sales_amount = order_qty * unit_price * (1 - unit_price_discount)
    - subtotal
    - tax_amt
    - freight
  - Fact classification:
    - SalesOrderHeader = transactional fact header
    - SalesOrderDetail = transactional fact detail
    - CustomerAddress = bridge table, not exposed in final star schema

## Test

Write all test outcomes to:
- test.test_results

Schema:
- run_id
- test_name
- layer
- table_name
- status
- actual
- expected
- details
- checked_at

Required tests:

1. Row Count Reconciliation
- Compare bronze vs silver row counts per table.
- PASS if variance <= 1%.

2. Gold Dimension PK Not Null
- dim_customer.customer_id
- dim_salesperson.salesperson_key
- dim_product.product_id
- dim_order.sales_order_id
- dim_order_date.date_key
- dim_ship_date.ship_date_key

3. Gold Dimension PK Uniqueness
- Verify uniqueness of all dimension primary keys.

4. Referential Integrity
- fact_sales_order.product_key exists in dim_product
- fact_sales_order.customer_key exists in dim_customer
- fact_sales_order.salesperson_key exists in dim_salesperson
- fact_sales_order.order_key exists in dim_order
- fact_sales_order.order_date_key exists in dim_order_date
- fact_sales_order.ship_date_key exists in dim_ship_date

5. Business Rule Sanity Checks
- net_sales_amount >= 0
- discount_amount >= 0
- unit_price >= 0
- order_qty > 0
- average discount percentage between 0 and 100
- monthly sales totals should not contain null month assignments

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

Hierarchies:

- Order Date Hierarchy
  - Year
  - Quarter
  - Month
  - Date

- Ship Date Hierarchy
  - Year
  - Quarter
  - Month
  - Date

- Customer Geography Hierarchy
  - City
  - Postal Code

- Product Category Hierarchy
  - Parent Product Category ID
  - Product Category ID
  - Product

Measures:

- Total Sales =
  - SUM(net_sales_amount)

- Average Sales =
  - AVERAGE(net_sales_amount)

- Maximum Sale =
  - MAX(net_sales_amount)

- Total Orders =
  - DISTINCTCOUNT(order_key)

- Total Quantity =
  - SUM(order_qty)

- Total Discount Amount =
  - SUM(discount_amount)

- Average Discount Amount =
  - AVERAGE(discount_amount)

- Discount Percentage =
  - DIVIDE(SUM(discount_amount), SUM(extended_amount), 0)

- Average Discount Percentage =
  - AVERAGE(unit_price_discount)

- Maximum Discount Percentage =
  - MAX(unit_price_discount)

- Average Order Value =
  - DIVIDE([Total Sales], [Total Orders], 0)

## Report

### Page 1 - Executive Sales Overview
Visuals:
- KPI cards:
  - Total Sales
  - Average Sales
  - Maximum Sale
  - Total Orders
- Monthly sales trend line chart
- Sales by salesperson bar chart
- Sales by ship method column chart
- Slicers:
  - Date
  - Salesperson
  - Product

### Page 2 - Regional Performance
Visuals:
- Filled map or bubble map using city and postal code from dim_customer
- Sales by region map colored by Total Sales
- High-performing regions ranked bar chart
- Low-performing regions ranked bar chart
- Average Sales by region
- Maximum Sale by region
- Regional order count matrix

Note:
- Source schema provides city and postal code but no state/country fields. Regional analysis is limited to available geography attributes.

### Page 3 - Orders and Discounts
Visuals:
- Monthly order volume trend
- Top discounted products
- Salesperson discount leaderboard
- Average Discount Percentage by salesperson
- Maximum Discount Percentage by salesperson
- Order status distribution
- Product performance matrix

### Page 4 - Product Insights
Visuals:
- Product sales ranking
- Product description drill-through
- Quantity sold by product
- Active vs discontinued products
- Product category hierarchy matrix

### Page 5 - Data Quality
Visuals:
- Test result summary
- PASS/FAIL counts
- Failed test detail table
- Row count reconciliation trends
- Referential integrity status

## Data Agent

Role:
- Sales Performance Intelligence Agent

Grounding:
- Use only the Direct Lake semantic model.
- Answer exclusively from model data and measures.
- Prefer approved measures over ad hoc aggregations.

Domain focus:
- Sales performance
- Regional performance
- Order trends
- Discount analysis
- Product performance
- Salesperson effectiveness

Instructions:
- Explain calculations using model measures.
- Always identify the time period used.
- When discussing regional performance, use available city/postal-code geography.
- Highlight top and bottom performers when ranking results.
- Use Average Sales, Maximum Sale, Total Sales, and Discount Percentage measures whenever relevant.
- If a requested attribute does not exist in the model, state that it is unavailable rather than inferring values.
- Distinguish clearly between gross sales, discount amount, and net sales.
- When answering trend questions, include change over time where possible.
- Recommend relevant report pages when visual exploration would help.

Starter questions:
- Which cities generated the highest total sales?
- Which cities generated the lowest total sales?
- What are the monthly sales trends?
- Which salesperson offered the largest average discount percentage?
- Which salesperson offered the largest maximum discount percentage?
- What are the top 10 products by net sales?
- Which orders generated the highest sales amounts?
- What is the average order value by month?
- How do discounts impact net sales?
- Which products are discontinued but still appear in sales history?

Guardrails:
- Do not fabricate geography beyond city and postal code.
- Do not infer missing product category names.
- Do not expose password_hash or password_salt values.
- Do not answer using data outside the semantic model.
- Do not generate forecasts unless explicit forecasting measures are added.
- When confidence is limited by missing attributes, explain the limitation.
