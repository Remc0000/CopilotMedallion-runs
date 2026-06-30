# Run Spec 20260630-091040-1454c5

## Updated specs

### Iteration 1 — 2026-06-30 09:21:48Z — failed layer: gold (run: 20260630-091209-7b614a)
- **Root cause (1-line summary)**: Gold-layer build failed with session cancellation; the most likely underlying cause is an analyzer failure during multi-table Gold joins where overlapping key columns are referenced ambiguously or projected without explicit alias qualification.
- **Cross-table audit**:
  - customeraddress: yes — bridge table introduces shared customer_id/address_id keys when joined.
  - salesorderdetail: yes — shares sales_order_id and product_id with other Gold sources.
  - productdescription: yes — shares product_description_id with bridge joins.
  - customer: yes — shares customer_id with salesorderheader.
  - productcategory: yes — self-referencing hierarchy and shared product_category_id.
  - productmodel: yes — shares product_model_id with product.
  - salesorderheader: yes — shares customer_id and sales_order_id with other Gold sources.
  - productmodelproductdescription: yes — bridge table shares product_model_id and product_description_id.
  - product: yes — shares product_model_id and product_category_id with lookup tables.
  - address: yes — shares address identifiers with customer geography derivations.
- **Fix approach**: GENERALIZE — the risk is systemic across nearly every Gold join because multiple source tables expose overlapping business keys.
- **What was changed**:
  - Tightened Gold join requirements to require alias-qualified projections for every selected column from joined DataFrames.
  - Added explicit join keys and projection rules for dim_customer, dim_product, and fact_sales_order.
  - Added mandatory schema validation before each Gold join and before final writes.

## Inputs
- Workspace: `f7395359-9652-4e13-8db5-610cef19a78d`
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
- Target Lakehouse: **erwin**

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
- Use defensive column references and validate required columns before every join, filter, aggregation, window, and derived-column expression.
- After every join, project alias-qualified columns and rename immediately to avoid ambiguity.
- Assert column existence before every groupBy/agg operation.
- For REST/API calls, use defensive handling and raise on `None` before any `.get()` access.
- Create schemas explicitly with `CREATE SCHEMA IF NOT EXISTS bronze`, `silver`, `gold`, and `test`.
- Write all outputs with schema-qualified `saveAsTable('<schema>.<table>')`.
- Never write target outputs using raw abfss `.save()` paths.
- Include notebook parameter cells for run_id, workspace_id, source_lakehouse, and target_lakehouse.
- Use idempotent overwrite patterns with schema overwrite enabled.
- Use error-loud try/except blocks that call `_save_error(layer, e)` and re-raise.
- Process source tables independently per-table in Bronze and Silver.
- Persist discoverable Delta tables in every layer.
- Every generated notebook cell must start with a short explanatory comment block.

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

Rule L — Disambiguate shared columns in join projections (avoid AMBIGUOUS_REFERENCE).

ALSO REQUIRE for every generated notebook: EACH code cell must start with a short markdown comment block (Python `# ---` divider + 1-3 lines of `# ` comments) describing what the cell is doing and why — never emit a cell with no leading comment.

## Bronze

Land each source table unchanged into the `bronze` schema with source metadata.

Tables:
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

Bronze standards:
- Preserve source schema and data types.
- Add ingestion metadata:
  - _run_id
  - _bronze_loaded_at
  - _source_table
- Partition large transactional tables by:
  - salesorderheader: year(orderdate)
  - salesorderdetail: salesorderid
- Smaller master-data tables can remain unpartitioned.
- Overwrite existing Bronze tables idempotently.
- Capture row counts per table in notebook summary output.

## Silver

Apply standardized cleansing:
- Rename all columns to snake_case.
- Trim string columns.
- Preserve binary thumbnail_photo.
- Standardize timestamps to UTC.
- Add:
  - _silver_loaded_at
  - _source_dt
  - _is_current_record

Deduplication strategy:
- customer: customer_id using latest modified_date
- address: address_id using latest modified_date
- salesorderheader: sales_order_id using latest modified_date
- salesorderdetail: sales_order_detail_id using latest modified_date
- product: product_id using latest modified_date
- productcategory: product_category_id using latest modified_date
- productmodel: product_model_id using latest modified_date
- productdescription: product_description_id using latest modified_date
- customeraddress: composite key (customer_id, address_id, address_type)
- productmodelproductdescription: composite key (product_model_id, product_description_id, culture)

Silver business preparation:
- Derive product lifecycle flags:
  - is_discontinued
  - is_currently_sellable
- Derive product_category_level:
  - category
  - subcategory
- Derive sales_person_clean from customer.sales_person:
  - When value contains `domain\username`, keep username only.
  - Remove trailing numeric identifiers where present to support user requirement example (e.g. adventure-works\jillian0 → jillian).
- Build reusable cleaned lookup tables for Gold.
- OPTIMIZE all Silver Delta tables after load.

## Gold

Target star schema aligned to requested reporting requirements.

Gold join safety requirements (mandatory for every Gold table):
- Before every join, validate all join-key columns exist on both sides.
- Use DataFrame aliases for every joined source (`c`, `a`, `h`, `d`, `p`, `pm`, `pc`, `pmpd`, `pd`, etc.).
- After every join, immediately project the final schema using only alias-qualified expressions such as `F.col('p.product_id').alias('product_id')`.
- Do not use bare-string projections for any column that could exist on more than one joined DataFrame.
- After the projection step, all downstream logic must reference the renamed flat column names only.
- Before writing each Gold table, assert the final DataFrame contains every column listed in the dimension/fact specification.

Dimensions:

1. gold.dim_order_date
- Source: salesorderheader.order_date
- Grain: one row per calendar date
- Attributes:
  - date_key
  - full_date
  - day
  - month
  - month_name
  - quarter
  - year
  - year_month
- Hierarchy:
  - Year → Quarter → Month → Date

2. gold.dim_ship_date
- Source: salesorderheader.ship_date
- Separate role-playing date dimension as requested.
- Same attributes as dim_order_date.
- Hierarchy:
  - Year → Quarter → Month → Date

3. gold.dim_customer
- Source: customer + address
- User requested not to use customeraddress.
- Join customer.customer_id to salesorderheader.customer_id and associate the primary reporting address through salesorderheader bill_to_address_id and ship_to_address_id usage.
- Required join sequence:
  - customer c ↔ salesorderheader h on c.customer_id = h.customer_id
  - h ↔ address a using bill_to_address_id or ship_to_address_id
- All projected columns must be explicitly selected from c or a aliases and renamed.
- Keep relevant attributes:
  - customer_id
  - customer_name (constructed from available name fields)
  - company_name
  - email_address
  - phone
  - city
  - state_province
  - country_region
  - postal_code
- Geography hierarchy:
  - Country → State/Province → City

Note:
- Because customer and address have no direct relationship without customeraddress, customer geography will be derived from addresses referenced by orders. This satisfies the user instruction while relying only on available keys.

4. gold.dim_sales_person
- Source: customer.sales_person
- Distinct cleaned sales_person values.
- Attributes:
  - sales_person_key
  - sales_person_name
- Hierarchy:
  - Sales Person

5. gold.dim_order
- Source: salesorderheader
- Move descriptive order attributes out of fact:
  - sales_order_id
  - revision_number
  - status
  - online_order_flag
  - ship_method
  - purchase_order_number
  - account_number
  - comment
- Hierarchy:
  - Status → Order

6. gold.dim_product
- Source:
  - product
  - productcategory
  - productmodel
  - productmodelproductdescription
  - productdescription
- Required join keys:
  - product.product_model_id = productmodel.product_model_id
  - product.product_category_id = productcategory.product_category_id
  - productmodel.product_model_id = productmodelproductdescription.product_model_id
  - productmodelproductdescription.product_description_id = productdescription.product_description_id
- Filter productmodelproductdescription to culture='en'
- In the join projection, ALWAYS alias-qualify:
  - product_id
  - product_model_id
  - product_category_id
  - product_description_id
  - name columns from product, productmodel, and productcategory
- Product model name sourced from productmodel.name as model_name.
- Description sourced from productdescription.description.
- Resolve parent-child productcategory structure:
  - Category (parent)
  - Subcategory (child)
- Keep relevant attributes:
  - product_id
  - product_name
  - product_number
  - color
  - size
  - model_name
  - description
  - category_name
  - subcategory_name
  - standard_cost
  - list_price
  - sell_start_date
  - sell_end_date
  - discontinued_date
- Hierarchies:
  - Category → Subcategory → Product
  - Model → Product

Fact:

7. gold.fact_sales_order
- Source: salesorderheader + salesorderdetail
- Required join key:
  - salesorderheader.sales_order_id = salesorderdetail.sales_order_id
- Project all columns using explicit aliases (`h.*` references are not permitted in the final projection).
- Derive dimension foreign keys only after the joined dataset has been projected into uniquely named columns.
- Grain:
  - One row per sales order detail line.

Foreign keys:
- order_date_key
- ship_date_key
- customer_key
- sales_person_key
- order_key
- product_key

Measures stored as fact columns:
- order_qty
- unit_price
- unit_price_discount
- gross_sales_amount = order_qty * unit_price
- discount_amount = order_qty * unit_price * unit_price_discount
- net_sales_amount = gross_sales_amount - discount_amount
- subtotal
- tax_amt
- freight

Reporting focus:
- Regional performance by country, state, city.
- Product category and subcategory performance.
- Monthly sales trends.
- Discount analysis by salesperson.
- Average and maximum sales metrics.

## Test

Store all results in:
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
- dim_sales_person.sales_person_key
- dim_order_date.date_key
- dim_ship_date.date_key

3. Gold Dimension PK Uniqueness
- Validate uniqueness of each dimension key.

4. Referential Integrity
- Every fact product_key exists in dim_product.
- Every fact customer_key exists in dim_customer.
- Every fact order_key exists in dim_order.
- Every fact sales_person_key exists in dim_sales_person.
- Every fact order_date_key exists in dim_order_date.
- Every fact ship_date_key exists in dim_ship_date.

5. Business Rule Sanity Checks
- net_sales_amount <= gross_sales_amount.
- discount_amount >= 0.
- unit_price >= 0.
- order_qty > 0.
- product category assignment exists for at least 95% of sold products.
- average sales and maximum sales measures return non-null values.

## Semantic model

Mode:
- Direct Lake

Tables:
- dim_order_date
- dim_ship_date
- dim_customer
- dim_sales_person
- dim_order
- dim_product
- fact_sales_order

Relationships:
- fact_sales_order → dim_order_date
- fact_sales_order → dim_ship_date
- fact_sales_order → dim_customer
- fact_sales_order → dim_sales_person
- fact_sales_order → dim_order
- fact_sales_order → dim_product

Hierarchies:
- Order Date: Year → Quarter → Month → Date
- Ship Date: Year → Quarter → Month → Date
- Geography: Country → State → City
- Product: Category → Subcategory → Product
- Product Model: Model → Product

Measures:
- Total Sales = SUM(net_sales_amount)
- Gross Sales = SUM(gross_sales_amount)
- Total Discount Amount = SUM(discount_amount)
- Average Sales = AVERAGE(net_sales_amount)
- Maximum Sale = MAX(net_sales_amount)
- Order Count = DISTINCTCOUNT(sales_order_id)
- Total Quantity Sold = SUM(order_qty)
- Average Discount Amount = AVERAGE(discount_amount)
- Discount Percentage = DIVIDE([Total Discount Amount],[Gross Sales])
- Maximum Discount Percentage = MAXX(fact_sales_order,DIVIDE(discount_amount,gross_sales_amount))
- Average Unit Price = AVERAGE(unit_price)
- Average Freight = AVERAGE(freight)

## Report

Page 1: Executive Sales Overview
- KPI cards:
  - Total Sales
  - Gross Sales
  - Average Sales
  - Maximum Sale
  - Order Count
- Monthly sales trend line chart.
- Category performance ribbon chart.
- Top and bottom performing regions visual.
- Executive layout themed using major product categories from dim_product.

Page 2: Regional Performance Map
- Filled map and bubble map.
- Geography:
  - Country
  - State/Province
  - City
- Color by Total Sales.
- Size by Order Count.
- Tooltip:
  - Average Sales
  - Maximum Sale
  - Discount Percentage
- Drill-down geography hierarchy.
- High-performing and low-performing region ranking visuals.

Page 3: Product & Category Insights
- Category → Subcategory → Product hierarchy matrix.
- Treemap by category sales.
- Product profitability scatter plot:
  - Gross Sales
  - Discount Amount
  - Quantity Sold
- Product model analysis.
- Category trend decomposition tree.

Page 4: Orders & Discounts
- Order status breakdown.
- Monthly order volume trend.
- Salesperson ranking table.
- Top discounting salespeople.
- Discount Percentage trend.
- Average and Maximum Discount visuals.

Page 5: Data Quality
- Test execution summary.
- Pass/Fail counts.
- Referential integrity status.
- Row count reconciliation results.
- Latest run metadata.

## Data Agent

Role:
- Sales Performance Analytics Agent

Domain hints:
- Regional sales analysis
- Product category and subcategory performance
- Order trends
- Customer geography
- Salesperson discount behavior
- Monthly sales patterns
- Product model performance

Starter questions:
- Which regions generated the highest sales this month?
- Which regions generated the lowest sales this quarter?
- What is the average and maximum sale by country?
- Which product categories generate the most revenue?
- Which subcategories are growing fastest?
- Which salespeople provide the largest discounts?
- How has discount percentage changed over time?
- What are monthly sales trends by category?
- Which products have the highest net sales?
- How do online orders compare to other orders?

Guardrails:
- Answer only using semantic model data.
- Do not infer missing customer attributes.
- Do not expose password_hash or password_salt fields.
- Never expose rowguid values.
- Use approved business measures when reporting sales.
- Clearly distinguish gross sales, discount amount, and net sales.
- Prefer hierarchy-aware aggregations for geography and product analysis.
- Surface data quality issues when related tests fail.