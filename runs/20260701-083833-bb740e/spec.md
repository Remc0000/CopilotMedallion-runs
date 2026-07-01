# Run Spec 20260701-083709-83716a

## Inputs
- Workspace: `586d3e08-a02f-4277-84c4-49167b1a671b`
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
- Target Lakehouse: **hallo**

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
- Use defensive column references and validate required columns before joins, filters, aggregations, windows, and derived-column logic.
- Use alias-prefixed join projections immediately after every join and explicitly rename overlapping columns.
- Assert column existence before every groupBy/agg operation.
- Use defensive REST handling with `if x is None: raise` before any `.get()` access.
- Create schemas with `CREATE SCHEMA IF NOT EXISTS` and write via `saveAsTable('<schema>.<table>')`.
- Never write target lakehouse outputs using raw abfss `.save()` paths.
- Include parameter/configuration cells at notebook start.
- Use idempotent overwrite patterns with schema evolution support.
- Wrap table processing in error-loud try/except blocks that call `_save_error(layer, e)` and re-raise.
- Process tables independently per layer and ensure discoverable Delta outputs.
- Every notebook cell must begin with a short comment block using `# ---` and explanatory comments.

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
- In typical operational sources (AdventureWorksLT, Northwind, AdventureWorks2019, etc.), entity tables (Customer, Product, SalesOrderHeader) have system audit columns: `ModifiedDate`, `rowguid`. **Junction / bridge tables** (CustomerAddress, ProductModelProductDescription, SalesTerritoryHistory) typically have only the FK columns and may have NO ModifiedDate and NO rowguid. **Views** (vGetAllCategories, vProductAndDescription) may have whatever columns the underlying query projects — frequently NO audit columns.
- When you write Silver dedup / tie-break / audit logic, you MUST NOT assume `modified_date` (or any other audit column) exists on every table. Use `'modified_date' in df.columns` as a guard and fall back to:
  - For dedup: a deterministic ranking expression that uses only the natural-key columns (`row_number().over(Window.partitionBy(*pk_cols).orderBy(*pk_cols))`), OR a literal F.lit(timestamp).
  - For `source_dt` / `_silver_ts`: a typed null literal (`F.lit(None).cast('timestamp')`) or `F.current_timestamp()`.
- Junction tables: dedupe on the COMPOSITE FK key (e.g. `(customer_id, address_id, address_type)`) — never on a non-existent surrogate key.
- View tables: project ONLY the columns actually returned by the view. Do not assume any standard naming.

Rule J — Validate column existence BEFORE the expensive transform.
- For every join, withColumn, groupBy, agg, or filter that names a specific column, ASSERT the column exists in `df.columns` BEFORE the line that uses it.

Rule K — Resilience to partial output: every layer MUST write Delta tables the next layer can discover.
- Bronze writes to `bronze.<table>`.
- Silver writes to `silver.<table>`.
- Gold writes to `gold.<table>` and tests to `test.test_results`.
- Raise an error if no discoverable tables are written.

Rule L — Disambiguate shared columns in join projections (avoid AMBIGUOUS_REFERENCE).
- Explicitly alias all overlapping columns in join projections and switch back to plain column names only after projection materialization.

## Bronze

Landing strategy:
- Ingest all 10 source tables unchanged into the `bronze` schema.
- Preserve original source column names and datatypes.
- Add metadata columns:
  - `_bronze_ingested_at`
  - `_bronze_run_id`
  - `_bronze_source_table`
  - `_bronze_record_hash`
- Write mode: Delta overwrite with schema evolution enabled.
- Partitioning:
  - `salesorderheader`: partition by OrderDate year/month.
  - `salesorderdetail`: partition by ModifiedDate year/month.
  - Remaining tables: unpartitioned unless volume justifies partitioning.
- Persist as:
  - bronze.customer
  - bronze.address
  - bronze.customeraddress
  - bronze.salesorderheader
  - bronze.salesorderdetail
  - bronze.product
  - bronze.productcategory
  - bronze.productmodel
  - bronze.productdescription
  - bronze.productmodelproductdescription

## Silver

Common standards:
- Convert column names to snake_case.
- Preserve business keys.
- Add:
  - `_silver_loaded_at`
  - `_silver_run_id`
  - `_source_modified_date`
- Remove exact duplicates.
- Normalize strings using trim.
- Validate PK presence before write.
- OPTIMIZE all Silver Delta tables after load.

Deduplication keys:
- customer → customer_id
- address → address_id
- customeraddress → (customer_id, address_id, address_type)
- salesorderheader → sales_order_id
- salesorderdetail → sales_order_detail_id
- product → product_id
- productcategory → product_category_id
- productmodel → product_model_id
- productdescription → product_description_id
- productmodelproductdescription → (product_model_id, product_description_id, culture)

Silver business preparation:
- customer:
  - Create cleaned salesperson source field.
  - Extract username from patterns like `domain\username`.
  - Store as salesperson_username.
- product:
  - Create is_discontinued flag from discontinued_date.
  - Create current_sellable flag.
- salesorderdetail:
  - Calculate line_discount_amount = order_qty * unit_price * unit_price_discount.
  - Calculate gross_line_amount = order_qty * unit_price.
  - Calculate net_line_amount = gross_line_amount - line_discount_amount.
  - Calculate discount_pct = unit_price_discount.
- salesorderheader:
  - Derive order_year, order_month, order_date_key.
  - Derive ship_date_key when available.

## Gold

Target schema requested by user:

### gold.dim_order_date
Source:
- salesorderheader.order_date

Attributes:
- Date
- Year
- Quarter
- Month
- Month Name
- Week
- Day
- Year-Month

Hierarchy:
- Year → Quarter → Month → Date

### gold.dim_ship_date
Source:
- salesorderheader.ship_date

Attributes:
- Date
- Year
- Quarter
- Month
- Month Name
- Week
- Day
- Year-Month

Hierarchy:
- Year → Quarter → Month → Date

### gold.dim_customer

User requested Customer and Address combined without CustomerAddress.

Implementation note:
- No direct Customer→Address relationship exists in the provided schema.
- CustomerAddress is the only available bridge linking CustomerID to AddressID.
- Gold implementation should therefore use CustomerAddress internally to perform the join, but the resulting dimension exposes only customer and address attributes and does not expose the bridge table.

Included attributes:
- customer_id
- company_name
- first_name
- last_name
- title
- email_address
- phone
- city
- state_province
- country_region
- postal_code

Hierarchy:
- Country Region → State Province → City

### gold.dim_salesperson

Source:
- customer.sales_person

Transformation:
- Extract username from domain-qualified values.
- Remove domain prefix.
- Normalize to lowercase.

Attributes:
- salesperson_key
- salesperson_username

Hierarchy:
- Salesperson Username

### gold.dim_order

Source:
- salesorderheader

Move dimensional attributes out of fact:
- sales_order_id
- revision_number
- status
- online_order_flag
- ship_method
- purchase_order_number
- account_number
- comment

Hierarchy:
- Status → Order

### gold.dim_product

Source combination:
- product
- productcategory
- productmodel
- productmodelproductdescription
- productdescription

Join strategy:
- product.product_category_id → productcategory.product_category_id
- product.product_model_id → productmodel.product_model_id
- productmodelproductdescription.product_model_id → productmodel.product_model_id
- productmodelproductdescription.product_description_id → productdescription.product_description_id
- Filter productmodelproductdescription where culture = 'en'

Parent-child category handling:
- Parent category from productcategory.parent_product_category_id
- Subcategory from productcategory.name
- Resolve both category and subcategory names.

Attributes:
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
- sell_start_date
- sell_end_date
- discontinued_date
- is_discontinued

Hierarchy:
- Category → Subcategory → Product

### gold.fact_sales_order

Source:
- salesorderheader
- salesorderdetail

Grain:
- One row per sales order line.

Keys:
- order_date_key
- ship_date_key
- customer_id
- salesperson_key
- product_id
- sales_order_id

Measures stored:
- order_qty
- unit_price
- unit_price_discount
- gross_sales_amount
- discount_amount
- net_sales_amount
- freight
- tax_amt
- subtotal

Calculated logic:
- gross_sales_amount = order_qty * unit_price
- discount_amount = order_qty * unit_price * unit_price_discount
- net_sales_amount = gross_sales_amount - discount_amount

Regional reporting support:
- Geography comes from dim_customer.
- Product segmentation comes from dim_product category hierarchy.
- Salesperson analysis comes from dim_salesperson.

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

Tests:

1. Row count reconciliation
- Bronze vs Silver row counts.
- PASS when variance <= 1%.

2. Gold dimension PK null check
- dim_customer.customer_id
- dim_product.product_id
- dim_order.sales_order_id
- dim_order_date.date_key
- dim_ship_date.date_key
- dim_salesperson.salesperson_key

3. Gold dimension uniqueness
- Verify uniqueness of all dimension PKs.

4. Referential integrity
- fact_sales_order.customer_id exists in dim_customer
- fact_sales_order.product_id exists in dim_product
- fact_sales_order.sales_order_id exists in dim_order
- fact_sales_order.order_date_key exists in dim_order_date
- fact_sales_order.ship_date_key exists in dim_ship_date
- fact_sales_order.salesperson_key exists in dim_salesperson

5. Business-rule sanity check
- net_sales_amount <= gross_sales_amount
- discount_amount >= 0
- order_qty > 0
- discount percentage between 0 and 1
- category_name populated for active products where category exists

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
- fact_sales_order → dim_order_date
- fact_sales_order → dim_ship_date
- fact_sales_order → dim_customer
- fact_sales_order → dim_salesperson
- fact_sales_order → dim_order
- fact_sales_order → dim_product

Hierarchies:
- Order Date: Year → Quarter → Month → Date
- Ship Date: Year → Quarter → Month → Date
- Geography: Country → State → City
- Product: Category → Subcategory → Product
- Order: Status → Order
- Salesperson: Username

Measures:
- Total Sales = SUM(net_sales_amount)
- Gross Sales = SUM(gross_sales_amount)
- Total Discount Amount = SUM(discount_amount)
- Average Discount Amount = AVERAGE(discount_amount)
- Discount Percentage = DIVIDE([Total Discount Amount],[Gross Sales])
- Average Sales = AVERAGE(net_sales_amount)
- Maximum Sales = MAX(net_sales_amount)
- Total Orders = DISTINCTCOUNT(sales_order_id)
- Average Order Value = DIVIDE([Total Sales],[Total Orders])
- Total Quantity = SUM(order_qty)
- Average Quantity Per Order = DIVIDE([Total Quantity],[Total Orders])
- Maximum Order Line Value = MAX(net_sales_amount)

## Report

### Page 1 — Executive Sales Overview
Visuals:
- KPI cards:
  - Total Sales
  - Average Sales
  - Maximum Sales
  - Total Orders
  - Average Order Value
- Monthly sales trend line chart.
- Category sales treemap using category/subcategory hierarchy.
- Top and bottom performing categories.
- Dynamic category-themed layout driven by product categories.

### Page 2 — Regional Performance
Visuals:
- Filled map by Country Region.
- Bubble map by State Province and City.
- Heat map of regional sales.
- Matrix:
  - Country
  - State
  - City
  - Total Sales
  - Average Sales
  - Maximum Sales
- Top 10 and Bottom 10 regions.

### Page 3 — Product Performance
Visuals:
- Category → Subcategory drill-down bar chart.
- Product ranking chart.
- Gross vs Net sales comparison.
- Product profitability view:
  - Sales
  - Standard Cost
  - Margin estimate

### Page 4 — Orders & Trends
Visuals:
- Monthly sales trend.
- Order volume trend.
- Ship-date vs order-date analysis.
- Order status distribution.
- Average and maximum order values by month.

### Page 5 — Discount & Salesperson Insights
Visuals:
- Top discounting salespeople.
- Discount percentage by salesperson.
- Scatter plot:
  - Discount %
  - Sales
  - Orders
- Highest discounted products and categories.
- Salesperson leaderboard.

### Page 6 — Data Quality
Visuals:
- Test pass/fail summary.
- Failed tests table.
- Row-count reconciliation dashboard.
- Referential integrity status.

## Data Agent

Role:
- Sales Performance Analytics Agent for order, customer, geography, product-category, discount, and salesperson analysis.

Domain hints:
- Sales transactions are stored at sales-order-line grain.
- Geography originates from customer address data.
- Product hierarchy contains Category and Subcategory.
- Date analysis can use Order Date or Ship Date.
- Discount analysis uses discount percentage and discount amount.

Starter questions:
- Which regions generated the highest sales?
- Which regions generated the lowest sales?
- What is the average sales amount by country?
- What is the maximum sales value by region?
- Which product categories sell the most?
- Which subcategories are growing fastest month over month?
- Which salespeople offer the largest discounts?
- What is the average discount percentage by salesperson?
- How have monthly sales trends changed over time?
- Which orders contributed the highest revenue?

Guardrails:
- Answer only using semantic model data.
- Use approved business measures where available.
- Distinguish Gross Sales, Discount Amount, and Net Sales.
- Use Order Date and Ship Date consistently and identify which is used.
- Never infer customer demographics not present in the model.
- Do not expose password_hash, password_salt, rowguid, or technical audit fields.
