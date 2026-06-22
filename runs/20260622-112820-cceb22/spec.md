# Run Spec 20260622-112600-ba55fb

## Inputs
- Workspace: `4b20377d-db30-4783-9bc0-4ab7d24a7046`
- Source Lakehouse: **SalesLake** (`040b6dbc-1c93-4448-9b22-cb2c26c79ee9`)
- Tables to ingest into Bronze:
  - `customeraddress`
  - `salesorderdetail`
  - `productdescription`
  - `customer`
  - `productcategory`
  - `productmodel`
  - `salesorderheader`
  - `productmodelproductdescription`
  - `product`
  - `address`
- Target Lakehouse: **Marc**

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
- Use alias-prefixed joins and explicitly rename overlapping columns inside join projections.
- Assert groupBy and agg columns exist before execution.
- For REST/API calls use defensive handling: `if x is None: raise RuntimeError(...)` before any `.get(...)`.
- Create schemas before writes: `CREATE SCHEMA IF NOT EXISTS bronze`, `silver`, `gold`, `test`.
- Write all outputs with schema-qualified `saveAsTable('<schema>.<table>')`.
- Never write target lakehouse outputs using raw abfss `.save()` paths.
- Include notebook parameter cells for workspace, lakehouse, run_id, source tables, and environment settings.
- Use idempotent overwrite patterns with Delta and overwriteSchema enabled.
- Use error-loud try/except blocks that call `_save_error(layer, e)` and re-raise.
- Process source tables independently within loops.
- Every notebook code cell must begin with a short comment block explaining purpose and business intent.

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
- Correct pattern — return a typed null literal as a Spark Column:
  ```
  def _maybe(df, name, dtype='timestamp'):
      return F.col(name) if name in df.columns else F.lit(None).cast(dtype)
  ```
  Pick the `dtype` to match the surrounding expression (`'timestamp'` for date/time coalesces, `'string'` for text, `'double'` for numeric, etc.) so Spark can resolve the result type without ambiguity.
- Alternative pattern (when the helper genuinely cannot know the dtype) — filter `None`s at the call site BEFORE invoking the Spark function:
  ```
  candidates = [c for c in (_maybe(df, 'modified_date'), _maybe(df, 'order_date')) if c is not None]
  df = df.withColumn('source_dt', F.to_date(F.coalesce(*candidates, F.current_timestamp())))
  ```
  Either approach is acceptable, but **never pass Python `None` directly into a Spark function**.
- Applies to ALL optional-column lookups across Bronze, Silver, Gold — including audit-timestamp coalesces, optional-key joins, fallback string formatting, etc. This is a layer-agnostic rule.

Rule H — Per-table isolation; one table's failure must not cancel the Spark session for the rest.
- Spark cancels the entire session when one statement crashes. If your notebook builds a single chained plan that touches every source table (one big SELECT, one big DataFrame, one big SQL script), any one table's failure kills ALL tables.
- ALWAYS process source tables in a `for tbl in source_tables:` loop where each iteration is a SELF-CONTAINED unit: read → transform → write → record-result → recover. Wrap the loop body in `try/except` that calls `_save_error(layer, e, table=tbl)` and APPENDS the failure to a results dict, then re-raises only AFTER the loop has attempted all tables.
- Do NOT build a single multi-CTE Spark SQL statement that joins/transforms many source tables in one shot.
- Do NOT share intermediate temp views across tables.

Rule I — Optional audit columns on junction / bridge / view tables.
- Use audit-column guards and composite-key deduplication for junction tables.
- `customeraddress` dedupe key: `(customer_id, address_id, address_type)`.
- `productmodelproductdescription` dedupe key: `(product_model_id, product_description_id, culture)`.

Rule J — Validate column existence BEFORE the expensive transform.
- Assert required columns exist before joins, aggregations, filters, and derived-column creation.

Rule K — Resilience to partial output: every layer MUST write Delta tables the next layer can discover.
- Bronze writes to `bronze.<table>`.
- Silver writes to `silver.<table>`.
- Gold writes to `gold.<table>`.
- Tests write to `test.test_results`.

Rule L — Disambiguate shared columns in join projections (avoid AMBIGUOUS_REFERENCE).
- Alias-qualify overlapping columns in all joins and rename explicitly.

## Bronze

Land each source table 1:1 into the `bronze` schema with no business transformations.

Source-to-target:
- `customeraddress` → `bronze.customeraddresses`
- `salesorderdetail` → `bronze.salesorderdetails`
- `productdescription` → `bronze.productdescription`
- `customer` → `bronze.customer`
- `productcategory` → `bronze.productcategory`
- `productmodel` → `bronze.productmodel`
- `salesorderheader` → `bronze.salesorderheader`
- `productmodelproductdescription` → `bronze.productmodelproductdescription`
- `product` → `bronze.product`
- `address` → `bronze.address`

Common metadata columns:
- `_ingested_at`
- `_run_id`
- `_source_table`
- `_bronze_ts`

Write pattern:
- Delta format
- Overwrite mode
- overwriteSchema=true
- saveAsTable into `bronze` schema

Suggested partitioning:
- `salesorderheader`: partition by order year derived from `OrderDate`
- `salesorderdetail`: partition by `SalesOrderID`
- Remaining tables: unpartitioned due to expected dimension-sized volume

## Silver

Standardize all column names to snake_case and preserve business keys.

Common silver rules:
- Trim strings
- Normalize empty strings to null where appropriate
- Add `_silver_ts`
- Deduplicate using latest `modified_date`
- Remove exact duplicate rows
- OPTIMIZE after load

Table-specific deduplication:
- customer: `customer_id`
- address: `address_id`
- salesorderheader: `sales_order_id`
- salesorderdetail: `sales_order_detail_id`
- product: `product_id`
- productcategory: `product_category_id`
- productmodel: `product_model_id`
- productdescription: `product_description_id`
- customeraddress: `(customer_id,address_id,address_type)`
- productmodelproductdescription: `(product_model_id,product_description_id,culture)`

Additional silver transformations:
- Customer:
  - Create full_name from title, first_name, middle_name, last_name, suffix.
  - Create normalized_sales_person_source.
- Product:
  - Create is_discontinued flag from discontinued_date.
  - Create active_product flag.
- SalesOrderHeader:
  - Derive order_year, order_month, order_date_key.
  - Derive ship_date_key when ship_date exists.
- SalesOrderDetail:
  - Calculate line_discount_amount = order_qty * unit_price * unit_price_discount.
  - Calculate gross_line_amount = order_qty * unit_price.
  - Calculate net_line_amount = order_qty * unit_price * (1 - unit_price_discount).

## Gold

Gold schema required by user.

Dimension: `gold.dim_order_date`
- Source: salesorderheader.order_date
- One row per calendar date.
- Attributes:
  - date
  - year
  - quarter
  - month
  - month_name
  - week
  - day_of_week
- Hierarchy:
  - Year → Quarter → Month → Date

Dimension: `gold.dim_ship_date`
- Source: salesorderheader.ship_date
- Same structure as order date dimension.
- Hierarchy:
  - Year → Quarter → Month → Date

Dimension: `gold.dim_customer`
- User requirement: combine Customer and Address and do not use CustomerAddress.
- Since no direct Customer→Address relationship exists in the provided schema, this is not technically derivable from the available columns.
- Fallback implementation:
  - Primary customer attributes from Customer.
  - If user later provides a direct relationship, enrich with address attributes.
- Keep:
  - customer_id
  - company_name
  - full_name
  - email_address
  - phone
  - sales_person_key
- Hierarchy:
  - Company → Customer

Dimension: `gold.dim_sales_person`
- Source: customer.sales_person
- Extract username component.
- Example:
  - `adventure-works\jillian0` → `jillian0`
- Keep:
  - sales_person_key
  - sales_person_username
  - original_sales_person
- Hierarchy:
  - Sales Person

Dimension: `gold.dim_order`
- Source: salesorderheader
- Move descriptive order attributes out of fact:
  - sales_order_id
  - status
  - online_order_flag
  - purchase_order_number
  - account_number
  - ship_method
  - comment
  - revision_number
- Hierarchy:
  - Status → Order

Dimension: `gold.dim_product`
- User-required combined dimension.
- Source tables:
  - product
  - productcategory
  - productmodel
  - productmodelproductdescription
  - productdescription

Join strategy:
- product.product_category_id → productcategory.product_category_id
- product.product_model_id → productmodel.product_model_id
- productmodelproductdescription.product_model_id → productmodel.product_model_id
- Filter productmodelproductdescription.culture = 'en'
- productmodelproductdescription.product_description_id → productdescription.product_description_id

Parent-child category handling:
- Join category as child and parent.
- Expose:
  - category_name
  - subcategory_name

Keep relevant attributes:
- product_id
- product_name
- product_number
- color
- size
- weight
- standard_cost
- list_price
- model_name
- description
- category_name
- subcategory_name
- active_product

Hierarchies:
- Category → Subcategory → Product
- Model → Product

Fact: `gold.fact_sales_order`
- Grain:
  - One row per sales order line.
- Source:
  - salesorderheader + salesorderdetail

Keys:
- sales_order_id
- sales_order_detail_id
- customer_id
- product_id
- order_date_key
- ship_date_key
- sales_person_key

Measures stored as additive facts:
- order_qty
- unit_price
- unit_price_discount
- gross_line_amount
- net_line_amount
- line_discount_amount
- subtotal
- tax_amount
- freight

Fact calculations:
- discount_pct = unit_price_discount * 100

Business focus:
- Regional reporting will use address geography dimensions available from Address. Because customer-to-address linkage exists only through CustomerAddress, create a supporting geography dimension from Address and connect through SalesOrderHeader bill_to_address_id and ship_to_address_id.
- Additional dimension:
  - `gold.dim_geography`
  - address_id
  - city
  - state_province
  - country_region
  - postal_code
- Hierarchy:
  - Country → State/Province → City

## Test

All tests append results to:
- `test.test_results`

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

Standard tests:

1. Row Count Reconciliation
- Bronze vs Silver row counts
- Tolerance: ±1%

2. Gold Dimension PK Null Check
- dim_customer.customer_id
- dim_product.product_id
- dim_order.sales_order_id
- dim_sales_person.sales_person_key
- dim_geography.address_id

3. Gold Dimension PK Uniqueness
- Verify unique primary keys for all dimensions.

4. Referential Integrity
- fact_sales_order.customer_id exists in dim_customer
- fact_sales_order.product_id exists in dim_product
- fact_sales_order.sales_person_key exists in dim_sales_person
- fact_sales_order.order_date_key exists in dim_order_date
- fact_sales_order.ship_date_key exists in dim_ship_date

5. Business Rule Sanity Check
- net_line_amount <= gross_line_amount
- unit_price_discount between 0 and 1
- order_qty > 0
- average discount percentage < 100

## Semantic model

Storage mode:
- Direct Lake

Tables:
- dim_order_date
- dim_ship_date
- dim_customer
- dim_sales_person
- dim_order
- dim_product
- dim_geography
- fact_sales_order

Relationships:
- fact_sales_order → dim_customer
- fact_sales_order → dim_product
- fact_sales_order → dim_order
- fact_sales_order → dim_sales_person
- fact_sales_order → dim_order_date
- fact_sales_order → dim_ship_date
- fact_sales_order → dim_geography

Hierarchies:
- Order Date: Year → Quarter → Month → Date
- Ship Date: Year → Quarter → Month → Date
- Geography: Country → State → City
- Product: Category → Subcategory → Product
- Customer: Company → Customer

Measures:
- Total Sales = SUM(net_line_amount)
- Gross Sales = SUM(gross_line_amount)
- Total Discount Amount = SUM(line_discount_amount)
- Average Discount % = AVERAGE(discount_pct)
- Maximum Discount % = MAX(discount_pct)
- Average Sale Value = AVERAGE(net_line_amount)
- Maximum Sale Value = MAX(net_line_amount)
- Total Orders = DISTINCTCOUNT(sales_order_id)
- Total Quantity Sold = SUM(order_qty)
- Average Order Value = DIVIDE([Total Sales],[Total Orders])
- Sales per Product = [Total Sales]
- Sales per Region = [Total Sales]

## Report

Page 1: Executive Sales Overview
- KPI cards:
  - Total Sales
  - Total Orders
  - Average Sale Value
  - Maximum Sale Value
  - Average Discount %
- Monthly sales trend line chart
- Sales by product category treemap
- Sales by subcategory bar chart
- Product category themed layout and color palette driven by category mix

Page 2: Regional Performance
- Filled map by country/state
- Bubble map by city
- High-performing vs low-performing regions ranking
- Average Sales by Region
- Maximum Sales by Region
- Regional sales heatmap
- Geographic drill hierarchy

Page 3: Orders & Discount Analysis
- Monthly order trend
- Salesperson ranking by Total Discount Amount
- Salesperson ranking by Average Discount %
- Discount distribution histogram
- Order status breakdown
- Top discounted products by category

Page 4: Product Performance
- Category → Subcategory → Product drill-down
- Sales by category
- Margin-oriented view using ListPrice vs StandardCost
- Product lifecycle indicators using sell/discontinued dates

Page 5: Data Quality
- Test result summary
- Failed test table
- Row-count reconciliation visuals
- Referential integrity scorecards

## Data Agent

Role:
- Sales Performance and Regional Analytics Assistant

Domain hints:
- Understand sales orders, products, categories, customers, geography, discounts, and sales trends.
- Product category hierarchy is the primary merchandising structure.
- Regional analysis is based on address geography.

Starter questions:
- Which regions generate the highest sales?
- Which regions generate the lowest sales?
- What is the monthly sales trend?
- What are the average and maximum sales values by region?
- Which product categories generate the most revenue?
- Which subcategories are growing fastest?
- Which salespeople offer the largest discounts?
- What is the average discount percentage by salesperson?
- Which products have the highest sales volume?
- How many orders were placed this month?

Guardrails:
- Answer only using semantic model data.
- Do not infer unavailable customer demographics.
- Clearly distinguish sales amount, discount amount, and discount percentage.
- Use category and subcategory hierarchies where applicable.
- Flag missing customer-address linkage when questions require customer residence geography.
- Prefer aggregated reporting over row-level customer detail.
- Respect all Power BI security filters and semantic model permissions.
