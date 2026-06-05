# Run Spec 20260605-083031-0b36b8

## Inputs
- Workspace: `1aec2347-3511-4e15-91d0-aeda945b41d8`
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
- Target Lakehouse: **p**

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
- Use defensive column existence validation before every join, filter, aggregation, withColumn, Window specification, and write.
- After every join, immediately project alias-prefixed columns into flat names before downstream transformations.
- Assert all groupBy and aggregation columns exist before execution.
- For REST/API responses, use `if x is None: raise RuntimeError(...)` before any `.get(...)` access.
- Create schemas explicitly with `CREATE SCHEMA IF NOT EXISTS bronze|silver|gold|test`.
- Write all managed tables via `saveAsTable('<schema>.<table>')`.
- Never write target outputs using raw abfss `.save()` paths.
- Use notebook parameter cells for run_id, workspace_id, source_lakehouse_id, target_lakehouse_id, and source table list.
- Use idempotent overwrite patterns with `mode('overwrite')` and `option('overwriteSchema','true')`.
- Wrap major processing blocks in try/except, call `_save_error(layer, e)` (or `_save_error(layer, e, table=tbl)` for table loops), then re-raise.
- All notebooks must process source tables in isolated loops and record per-table outcomes.
- Every notebook cell must begin with a short comment block explaining intent and business purpose.

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
- ALWAYS process source tables in a `for tbl in source_tables:` loop where each iteration is a SELF-CONTAINED unit: read → transform → write → record-result → recover. Wrap the loop body in `try/except` that calls `_save_error(layer, e, table=tbl)` and APPENDS the failure to a results dict, then **re-raises only AFTER the loop has attempted all tables** (or, if your spec says "fail-fast-first-table", re-raise immediately — but per-table-isolated by default).
- Do NOT build a single multi-CTE Spark SQL statement that joins/transforms many source tables in one shot. Each table's transform is its own DataFrame chain with its own `.write` call.
- Do NOT share intermediate temp views across tables. Temp views from one iteration must not be assumed to exist in the next. If you need cross-table joins (typical for Gold), do them in a SECOND loop AFTER all per-table Silver/Gold writes are complete.

Rule I — Optional audit columns on junction / bridge / view tables.
- In typical operational sources (AdventureWorksLT, Northwind, AdventureWorks2019, etc.), entity tables (Customer, Product, SalesOrderHeader) have system audit columns: `ModifiedDate`, `rowguid`. **Junction / bridge tables** (CustomerAddress, ProductModelProductDescription) typically have only the FK columns and may have NO ModifiedDate and NO rowguid. **Views** may have whatever columns the underlying query projects — frequently NO audit columns.
- When you write Silver dedup / tie-break / audit logic, you MUST NOT assume `modified_date` (or any other audit column) exists on every table. Use `'modified_date' in df.columns` as a guard and fall back to:
  - For dedup: a deterministic ranking expression that uses only the natural-key columns.
  - For `source_dt` / `_silver_ts`: a typed null literal or `F.current_timestamp()`.
- Junction tables: dedupe on the composite FK key.
- View tables: project only columns actually returned.

Rule J — Validate column existence BEFORE the expensive transform.
- For every join, withColumn, groupBy, agg, or filter that names a specific column, assert the column exists in `df.columns` before use.
- Especially important after select(), drop(), or rename().

Rule K — Resilience to partial output: every layer MUST write Delta tables the next layer can discover.
- Bronze writes to `bronze.<table>`.
- Silver writes to `silver.<table>`.
- Gold writes to `gold.<table>`.
- Test writes to `test.test_results`.
- Each notebook must raise if no discoverable output tables were written.

## Bronze

Land each source object unchanged into the `bronze` schema as Delta tables:

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

Common Bronze pattern:
- Preserve source schema and datatypes.
- Add metadata columns:
  - _bronze_ingested_at
  - _bronze_run_id
  - _bronze_source_table
- Overwrite schema-enabled managed tables.
- Partition large transactional tables by year/month derived from:
  - salesorderheader.OrderDate
  - salesorderdetail.ModifiedDate
- Small reference tables remain unpartitioned.
- Capture row counts and write summary JSON.
- Retain binary ThumbnailPhoto column in Product without transformation.

## Silver

Standardize all tables:
- Convert column names to snake_case.
- Add:
  - _silver_loaded_at
  - _silver_run_id
  - source_modified_date
- Remove duplicate records using latest modified_date where available.
- OPTIMIZE all silver tables after write.

Silver deduplication keys:
- address: address_id
- customer: customer_id
- customeraddress: (customer_id, address_id)
- product: product_id
- productcategory: product_category_id
- productdescription: product_description_id
- productmodel: product_model_id
- productmodelproductdescription: (product_model_id, product_description_id, culture)
- salesorderheader: sales_order_id
- salesorderdetail: (sales_order_id, sales_order_detail_id)

Silver business preparation:

address
- Standardize city and postal code formatting.
- Create normalized city field.

customer
- Retain business attributes.
- Exclude password_hash and password_salt from downstream gold dimensions.
- Create cleaned_salesperson_source.

salesorderheader
- Derive:
  - order_year
  - order_month
  - order_date_key
  - ship_date_key
- Validate customer_id, ship_to_address_id, bill_to_address_id.

salesorderdetail
- Derive:
  - line_sales_amount = order_qty * unit_price
  - line_discount_amount = order_qty * unit_price * unit_price_discount
  - net_sales_amount = order_qty * unit_price * (1 - unit_price_discount)
- NOTE: UnitPriceDiscount appears to be stored as a discount factor. Gold calculations should validate actual business meaning using sample data.

productcategory
- Build parent-child lookup helper fields for category/subcategory derivation.

productmodel
- NOTE: Requested ProductModel.Name → ModelName is not directly possible because ProductModel contains only ProductModelID, rowguid, and ModifiedDate. Gold Product dimension will carry ProductModelID and expose it as model identifier unless a name field becomes available.

productmodelproductdescription
- Filter helper view available for Culture='en' during Gold assembly.

## Gold

Target star schema optimized for Direct Lake reporting.

Dimensions

1. gold.dim_order_date
- Source: salesorderheader.order_date
- Grain: one row per calendar date.
- Attributes:
  - date_key
  - full_date
  - year
  - quarter
  - month
  - month_name
  - week
  - day
- Hierarchy:
  - Year → Quarter → Month → Date

2. gold.dim_ship_date
- Source: salesorderheader.ship_date
- Grain: one row per ship date.
- Attributes similar to Order Date.
- Hierarchy:
  - Year → Quarter → Month → Date

3. gold.dim_customer
- User requirement: combine Customer and Address without using CustomerAddress.
- Because Customer has no AddressID and Address has no CustomerID, there is no direct relationship available between the two tables.
- Gold implementation:
  - Primary customer attributes from Customer:
    - customer_id
    - company_name
    - title
    - suffix
    - email_address
  - Address attributes sourced through SalesOrderHeader joins where available:
    - city
    - postal_code
  - Keep only reporting-relevant fields.
- Alternative modeling note:
  - The actual schema indicates CustomerAddress is the true bridge. Excluding it may create incomplete customer-address attribution. Current build follows user instruction.

4. gold.dim_salesperson
- Source: customer.sales_person
- Distinct salesperson values.
- Transform:
  - If value contains `\`, keep username portion.
  - Remove trailing numeric suffix where present for cleaner display (example: adventure-works\jillian0 → jillian).
- Columns:
  - salesperson_key
  - salesperson_name
  - original_salesperson
- Hierarchy:
  - Single-level dimension.

5. gold.dim_order
- Source: salesorderheader
- Grain: sales_order_id
- Attributes:
  - sales_order_id
  - revision_number
  - status
  - ship_method
  - credit_card_approval_code
  - comment
- Exclude measures and foreign keys moved to fact.
- Hierarchy:
  - Status → Order

6. gold.dim_product
- Source:
  - product
  - productcategory
  - productmodel
  - productmodelproductdescription
  - productdescription
- Filter ProductModelProductDescription to culture='en'.
- Build category hierarchy:
  - category
  - subcategory
- Include relevant attributes:
  - product_id
  - product_number
  - color
  - size
  - weight
  - standard_cost
  - list_price
  - description
  - product_model_id
  - sell_start_date
  - sell_end_date
  - discontinued_date
- NOTE:
  - ProductCategory table contains identifiers only. No category name columns exist in provided schema.
  - Category/subcategory hierarchy will therefore use parent and child category IDs unless descriptive names become available.
- Hierarchies:
  - Category → Subcategory → Product
  - Color → Product

Facts

gold.fact_sales_order
- Grain: one order line.
- Source:
  - salesorderheader joined to salesorderdetail
- Keys:
  - sales_order_id
  - sales_order_detail_id
  - customer_id
  - product_id
  - salesperson_key
  - order_date_key
  - ship_date_key
  - order_dim_key
- Measures:
  - order_qty
  - unit_price
  - unit_price_discount
  - gross_sales_amount
  - discount_amount
  - net_sales_amount
  - subtotal
  - tax_amount
  - freight
- Regional reporting:
  - Derive region surrogate from available address city/postal attributes.
  - NOTE: No state, province, country, territory, or region columns exist in the source. Regional analysis will therefore be based on city/postal-code geography unless richer geography becomes available.

## Test

All tests append results into:
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
- Compare Bronze vs Silver row counts.
- PASS when variance <= 1%.

2. Gold Dimension PK Not Null
- dim_customer.customer_id
- dim_product.product_id
- dim_order.sales_order_id
- dim_order_date.date_key
- dim_ship_date.date_key
- dim_salesperson.salesperson_key

3. Gold Dimension PK Uniqueness
- Verify uniqueness of each dimension primary key.

4. Referential Integrity
- fact_sales_order.customer_id exists in dim_customer.
- fact_sales_order.product_id exists in dim_product.
- fact_sales_order.order_date_key exists in dim_order_date.
- fact_sales_order.ship_date_key exists in dim_ship_date.
- fact_sales_order.order_dim_key exists in dim_order.
- fact_sales_order.salesperson_key exists in dim_salesperson.

5. Business Rule Sanity Check
- net_sales_amount <= gross_sales_amount.
- discount_amount >= 0.
- order_qty > 0.
- ship_date >= order_date when ship_date is not null.
- Average discount percentage between 0% and 100%.

## Semantic model

Mode:
- Direct Lake

Tables:
- dim_order_date
- dim_ship_date
- dim_customer
- dim_salesperson
- dim_order
- dim_product
- fact_sales_order

Relationships:
- fact_sales_order → dim_customer
- fact_sales_order → dim_product
- fact_sales_order → dim_salesperson
- fact_sales_order → dim_order
- fact_sales_order → dim_order_date
- fact_sales_order → dim_ship_date

Hierarchies

dim_order_date
- Year → Quarter → Month → Date

dim_ship_date
- Year → Quarter → Month → Date

dim_product
- Category → Subcategory → Product

Measures

Sales Measures
- Total Sales = SUM(net_sales_amount)
- Gross Sales = SUM(gross_sales_amount)
- Average Sales = AVERAGE(net_sales_amount)
- Maximum Sales = MAX(net_sales_amount)

Discount Measures
- Total Discount Amount = SUM(discount_amount)
- Average Discount Amount = AVERAGE(discount_amount)
- Discount % =
  DIVIDE(SUM(discount_amount), SUM(gross_sales_amount))
- Average Discount % =
  AVERAGE(unit_price_discount)
- Maximum Discount % =
  MAX(unit_price_discount)

Order Measures
- Total Orders = DISTINCTCOUNT(sales_order_id)
- Total Order Lines = COUNTROWS(fact_sales_order)
- Average Order Value =
  DIVIDE([Total Sales],[Total Orders])

Trend Measures
- Sales MTD
- Sales YTD
- Previous Month Sales
- Monthly Sales Growth %

Geography Measures
- Sales by City
- Average Sales by City
- Maximum Sales by City

## Report

Page 1 — Executive Sales Overview
- KPI cards:
  - Total Sales
  - Gross Sales
  - Average Sales
  - Maximum Sales
  - Total Orders
  - Average Order Value
- Monthly sales trend line chart.
- Sales by product hierarchy matrix.
- Salesperson ranking bar chart.

Page 2 — Regional Performance
- Filled map or bubble map using city/postal geography.
- High-performing regions visual (Top N cities by sales).
- Low-performing regions visual (Bottom N cities by sales).
- Average Sales by City.
- Maximum Sales by City.
- Geographic drill-through page.

Page 3 — Orders and Discounts
- Orders by month.
- Discount % trend.
- Top salespeople by average discount.
- Top salespeople by maximum discount.
- Order status distribution.
- Detailed order table.

Page 4 — Product Performance
- Sales by category/subcategory.
- Product profitability view:
  - Gross Sales
  - Discount Amount
  - Net Sales
- Top and bottom products.

Page 5 — Data Quality
- Test result summary.
- Pass/fail counts.
- Failed test details.
- Layer reconciliation metrics.
- Refresh metadata and run ID.

## Data Agent

Role
- Enterprise Sales Performance Analyst for the SalesLT reporting solution.
- Grounded exclusively on the Direct Lake semantic model.
- Specialized in sales performance, geographic analysis, customer behavior, product performance, discounts, and order trends.

Domain Hints
- Interpret city/postal-code geography as the available regional construct.
- Use Order Date for sales trends unless the user explicitly requests shipping trends.
- Use Net Sales as the default sales metric.
- Explain calculations using semantic-model measures whenever possible.

Starter Questions
- Which cities generated the highest sales this month?
- Which cities generated the lowest sales this month?
- What are the monthly sales trends over time?
- What is the average and maximum sales amount by city?
- Which salespeople offer the largest discounts?
- Which products generate the most net sales?
- Which product categories perform best?
- What is the average order value by month?
- Which customers generate the most revenue?
- How do discounts affect net sales performance?

Guardrails
- Answer only using data exposed through the semantic model.
- Do not infer missing geography beyond available city/postal-code attributes.
- Clearly distinguish Gross Sales, Discount Amount, and Net Sales.
- State when requested information is unavailable in the model.
- Prefer aggregated results over row-level sensitive details.
- Never expose password_hash or password_salt fields.
- Explain filter context and time period when presenting metrics.
- Use defined measures rather than recreating calculations whenever possible.
