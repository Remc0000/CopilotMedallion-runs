# Run Spec 20260604-091237-ae5763

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
- Junction tables: dedupe on the COMPOSITE FK key.
- View tables: project ONLY the columns actually returned by the view. Do not assume any standard naming.

Rule J — Validate column existence BEFORE the expensive transform.
- For every join, withColumn, groupBy, agg, or filter that names a specific column, ASSERT the column exists in `df.columns` BEFORE the line that uses it.
- Especially important AFTER a select(), drop(), or rename() — re-validate before the next consumer of those columns.

Rule K — Resilience to partial output: Bronze MUST write Delta tables that the next layer can discover.
- Bronze writes to `Tables/bronze/<table>`.
- Silver writes to `Tables/silver/<table>`.
- Gold writes to `Tables/gold/<table>` and tests to `Tables/test/test_results`.
- Raise an error if a layer finishes with zero discoverable output tables.

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
- Attributes: date, year, quarter, month, month name, week, day.
- Hierarchy: Year → Quarter → Month → Date.

Dimension: `dim_ship_date`
- Source: SalesOrderHeader.ship_date.
- Grain: one row per calendar date.
- Attributes similar to order date dimension.
- Hierarchy: Year → Quarter → Month → Date.

Dimension: `dim_customer`
- Source: Customer + Address.
- User requested bypassing CustomerAddress.
- NOTE: Customer and Address have no direct key relationship in the supplied schema. A valid customer-address link only exists through CustomerAddress.
- Implementation recommendation: build dim_customer from Customer only for guaranteed correctness.
- Alternative (user may edit): join Customer → CustomerAddress → Address and keep city/postal information.
- Relevant attributes:
  - customer_id
  - company_name
  - title
  - suffix
  - email_address
  - city
  - postal_code
  - modified_date

Dimension: `dim_salesperson`
- Source: Customer.sales_person.
- One row per salesperson.
- Transform:
  - Extract username portion after `\`.
  - If value contains `domain\username`, keep only username.
  - Remove duplicates.
- Attributes:
  - salesperson_key
  - salesperson_username
  - salesperson_full_value
- Hierarchy:
  - Salesperson.

Dimension: `dim_order`
- Source: SalesOrderHeader.
- Grain: one row per order.
- Move non-measure attributes from header out of fact:
  - sales_order_id
  - revision_number
  - status
  - ship_method
  - credit_card_approval_code
  - comment
- Keep customer/date references as dimensional relationships.

Dimension: `dim_product`
- Source:
  - Product
  - ProductCategory
  - ProductModel
  - ProductModelProductDescription (culture='en')
  - ProductDescription
- NOTE: ProductModel table in supplied schema contains only ProductModelID and no Name column. User requested ProductModel.Name as ModelName but that column is not available.
- Fallback:
  - Use ProductDescription.Description as primary descriptive text.
  - Retain ProductModelID as model identifier.
- Category handling:
  - Resolve parent-child ProductCategory hierarchy.
  - Create category and subcategory attributes from parent-child relationship.
- Relevant attributes:
  - product_id
  - product_number
  - color
  - size
  - weight
  - standard_cost
  - list_price
  - category_id
  - category_name (if derivable from available source; otherwise category identifier)
  - subcategory_id
  - product_description
  - product_model_id
  - sell_start_date
  - sell_end_date
  - is_discontinued
- Hierarchy:
  - Category → Subcategory → Product.

Fact: `fact_sales_order`
- Source:
  - SalesOrderHeader
  - SalesOrderDetail
- Grain:
  - One row per sales order detail line.
- Joins:
  - SalesOrderHeader.sales_order_id = SalesOrderDetail.sales_order_id
  - Product via product_id
  - Customer via customer_id
  - SalesPerson via customer.sales_person
- Foreign keys:
  - order_date_key
  - ship_date_key
  - customer_key
  - salesperson_key
  - order_key
  - product_key
- Measures stored:
  - order_qty
  - unit_price
  - unit_price_discount
  - line_sales_amount = order_qty * unit_price
  - line_discount_amount = order_qty * unit_price * unit_price_discount
  - net_sales_amount = order_qty * unit_price * (1 - unit_price_discount)
  - standard_cost
  - freight_allocated (optional proportional allocation from header freight)

Regional-performance requirement:
- Region data is not explicitly present.
- Closest available geography is Address.city and postal_code.
- If CustomerAddress is used, build city-based geographic reporting and map visuals.
- If CustomerAddress remains excluded, regional analysis cannot be fully implemented from available schema.

## Test

Write all results into:
- `test/test_results`

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

1. Row count reconciliation
- Bronze vs Silver counts within 1% per table.

2. Gold dimension PK not null
- dim_customer.customer_id
- dim_product.product_id
- dim_order.sales_order_id
- dim_salesperson.salesperson_key
- dim_order_date.date_key
- dim_ship_date.date_key

3. Gold dimension PK uniqueness
- Validate uniqueness for all dimension primary keys.

4. Referential integrity
- fact_sales_order.product_key exists in dim_product.
- fact_sales_order.customer_key exists in dim_customer.
- fact_sales_order.order_key exists in dim_order.
- fact_sales_order.order_date_key exists in dim_order_date.
- fact_sales_order.ship_date_key exists in dim_ship_date when populated.
- fact_sales_order.salesperson_key exists in dim_salesperson.

5. Business-rule sanity checks
- Net sales amount >= 0.
- Unit price >= 0.
- Discount percentage between 0 and 100%.
- Ship date >= order date when ship date exists.
- Average sales amount <= maximum sales amount.

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

Hierarchies:
- Order Date: Year → Quarter → Month → Date
- Ship Date: Year → Quarter → Month → Date
- Product: Category → Subcategory → Product
- Customer Geography: City → Postal Code → Customer (when address data available)

Measures:
- Total Sales = SUM(net_sales_amount)
- Gross Sales = SUM(line_sales_amount)
- Total Discount Amount = SUM(line_discount_amount)
- Average Sale Amount = AVERAGE(net_sales_amount)
- Maximum Sale Amount = MAX(net_sales_amount)
- Total Orders = DISTINCTCOUNT(sales_order_id)
- Total Quantity = SUM(order_qty)
- Average Discount % = AVERAGE(unit_price_discount)
- Maximum Discount % = MAX(unit_price_discount)
- Sales per Order = DIVIDE([Total Sales],[Total Orders])
- Average Order Value = DIVIDE([Total Sales],[Total Orders])
- Freight Total = SUM(freight_allocated)

Regional analytics:
- Map visuals should use City and Postal Code when customer geography is available through CustomerAddress and Address.

## Report

Page 1: Executive Sales Overview
- KPI cards:
  - Total Sales
  - Gross Sales
  - Total Orders
  - Average Sale Amount
  - Maximum Sale Amount
- Monthly sales trend line chart.
- Sales by product category bar chart.
- Top products by sales.

Page 2: Regional Performance
- Filled map or bubble map using City/Postal Code.
- Sales by city ranking.
- Average sales by city.
- Maximum sales by city.
- High-performing vs low-performing regions visual.
- Geographic sales heatmap.

Page 3: Orders and Discounts
- Order trend by month.
- Order status distribution.
- Top salespeople by total sales.
- Top salespeople by average discount %.
- Top salespeople by maximum discount %.
- Discount analysis scatter chart.

Page 4: Product Performance
- Sales by category/subcategory hierarchy.
- Product profitability view.
- Product discount analysis.
- Top and bottom products.

Page 5: Data Quality
- Test results summary.
- Pass/fail counts.
- Referential-integrity status.
- Row-count reconciliation results.
- Failed-test detail table.

## Data Agent

Role:
- Sales Performance Intelligence Agent for LakeSales.

Domain instructions:
- Answer questions using only the semantic model.
- Focus on sales performance, order behavior, discount trends, product performance, customer activity, salesperson effectiveness, and geographic analysis.
- Prefer measures over raw column aggregation.
- Use date hierarchies when summarizing trends.
- Highlight average and maximum metrics whenever relevant.
- Explain significant outliers and concentration patterns.
- When geography is unavailable due to missing customer-address linkage, explicitly state the limitation.

Starter questions:
- Which cities generated the highest total sales?
- Which cities generated the lowest total sales?
- What are the monthly sales trends?
- What is the average and maximum sale amount by month?
- Which salespeople offer the largest discounts?
- Which salespeople have the highest average discount percentage?
- Which product categories generate the most revenue?
- Which products have the highest net sales?
- How many orders were placed each month?
- What is the average order value by salesperson?

Guardrails:
- Do not invent regions, territories, countries, or states not present in the model.
- Do not infer customer demographics that are not stored.
- Use only published semantic-model tables and measures.
- Distinguish clearly between gross sales and net sales.
- Flag data-quality issues if test results indicate failures.
- Prefer aggregated reporting over row-level customer details.
- When requested information cannot be derived from available data, explain the limitation and identify the missing fields.
