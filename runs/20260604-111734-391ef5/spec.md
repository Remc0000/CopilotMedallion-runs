# Run Spec 20260604-111656-40a6d8

## Inputs
- Workspace: `35fe2703-387e-4ca0-a948-16313a09cf18`
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
- Target Lakehouse: **b**

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
- Use defensive column references and assert required columns before every join, filter, aggregation, and derived-column expression.
- After every join, immediately project alias-prefixed columns into flat output names.
- Before every groupBy/agg, assert aggregation columns exist.
- For REST/API responses: `if x is None: raise RuntimeError(...)` before any `.get(...)`.
- Do not use `saveAsTable`; write Delta directly to Lakehouse paths.
- Every notebook must begin with parameter cells for workspace, lakehouse, run_id, source paths, and target paths.
- Use idempotent overwrite patterns with `overwriteSchema=true`.
- Wrap table processing in try/except, call `_save_error(layer, e)` (or `_save_error(layer, e, table=tbl)`), record failures, and re-raise appropriately.
- Process source tables independently per layer.
- Every code cell must start with a short explanatory comment block.

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
- For every join, withColumn, groupBy, agg, or filter that names a specific column, ASSERT the column exists in `df.columns` BEFORE the line that uses it. Pattern:
  ```
  for required in ('customer_id', 'order_date'):
      if required not in df.columns:
          raise RuntimeError(f"[{layer}] {tbl}: required column '{required}' missing; available={df.columns}")
  # … now the join / withColumn that uses customer_id and order_date
  ```
- Catches missing-column bugs in a SPECIFIC cell with a SPECIFIC table name, instead of a session-wide Spark cancellation 30 minutes later that the auto-fixer can't pinpoint.
- Especially important AFTER a select(), drop(), or rename() — re-validate before the next consumer of those columns.

Rule K — Resilience to partial output: Bronze MUST write Delta tables that the next layer can discover.
- The build pipeline runs each layer's notebook then inspects the lakehouse for the layer's output Delta tables before generating the next layer. If Bronze runs "successfully" (Spark Completed) but writes zero discoverable tables under `Tables/bronze/`, the build hard-fails with "prior layer produced no discoverable tables".
- To guarantee discoverability, the Bronze notebook MUST:
  - Write via `df.write.format('delta').mode('overwrite').option('overwriteSchema','true').partitionBy(...).save(f"{tgt_base}/Tables/bronze/<flat>")` for every source table, where `<flat>` is the lowercased last segment of `table_relative_path`.
  - Print a final summary line `print(json.dumps({{"bronze_results": {{<table>: {{"rows": N, "path": ...}}, ...}}}}))` listing every table actually written. Use this as a self-check.
  - Raise (not just log) if zero tables were written by the end of the notebook.
- Same rule applies recursively to Silver (`Tables/silver/<table>`) and Gold (`Tables/gold/<table>` + `Tables/test/test_results`).

ALSO REQUIRE for every generated notebook: EACH code cell must start with a short markdown comment block (Python `# ---` divider + 1-3 lines of `# ` comments) describing what the cell is doing and why — never emit a cell with no leading comment. Example:
```
# ---
# Read raw bronze table and apply schema enforcement.
# Drops rows where any required key is null.
df = spark.read.table(...)
```
Keep the comments human-readable, not the code repeated in prose. The goal: someone scrolling through the notebook in Fabric can understand each block at a glance.

## Bronze

- Land each source table unchanged into `Tables/bronze/<table_name_lower>`.
- Preserve all source columns and datatypes.
- Add metadata columns:
  - `_run_id`
  - `_ingested_at`
  - `_source_table`
  - `_bronze_ts`
- Write mode: overwrite with schema overwrite enabled.
- Partitioning:
  - SalesOrderHeader: partition by year derived from OrderDate.
  - SalesOrderDetail: partition by SalesOrderID hash bucket or non-partitioned if volume is small.
  - All master tables: non-partitioned.
- Bronze outputs:
  - bronze.address
  - bronze.customer
  - bronze.customeraddress
  - bronze.product
  - bronze.productcategory
  - bronze.productdescription
  - bronze.productmodel
  - bronze.productmodelproductdescription
  - bronze.salesorderheader
  - bronze.salesorderdetail

## Silver

General rules:
- Convert all columns to snake_case.
- Standardize timestamp columns.
- Add `_silver_ts`, `_run_id`, and lineage metadata.
- Deduplicate using latest modified_date where available.
- Remove duplicate records while preserving business keys.

Table-specific deduplication:
- address: dedupe on `address_id`.
- customer: dedupe on `customer_id`.
- customeraddress: dedupe on (`customer_id`, `address_id`).
- product: dedupe on `product_id`.
- productcategory: dedupe on `product_category_id`.
- productdescription: dedupe on `product_description_id`.
- productmodel:
  - dedupe on `product_model_id`.
  - NOTE: source schema contains no Name column. User requested ProductModel.Name → modelname in Gold. This cannot be satisfied from the supplied schema. Gold will expose ProductModelID and any available model-related description linkage unless ProductModel is enriched with a Name column later.
- productmodelproductdescription:
  - dedupe on (`product_model_id`, `product_description_id`, `culture`).
  - retain culture for Gold filtering.
- salesorderheader: dedupe on `sales_order_id`.
- salesorderdetail: dedupe on (`sales_order_id`, `sales_order_detail_id`).

Business standardization:
- Create `sales_person_username` from Customer.SalesPerson.
- When values follow `domain\username`, extract username portion.
- Apply lowercase normalization and remove trailing numeric suffixes only if business confirms; default behavior is to keep the extracted username exactly as provided.
- Create product lifecycle flags:
  - is_active_product
  - is_discontinued
- Create order lifecycle flags:
  - is_shipped
  - is_open_order

Performance:
- OPTIMIZE Silver tables after successful write.
- ZORDER suggested on:
  - sales_order_id
  - customer_id
  - product_id

## Gold

Target star schema requested by user.

Dimension: dim_order_date
- Source: SalesOrderHeader.OrderDate.
- One row per calendar date.
- Hierarchy:
  - Year > Quarter > Month > Date.
- Include standard calendar attributes.

Dimension: dim_ship_date
- Source: SalesOrderHeader.ShipDate.
- One row per calendar date.
- Hierarchy:
  - Year > Quarter > Month > Date.
- Include shipping calendar attributes.

Dimension: dim_customer
- Source: Customer + Address.
- User requested Customer and Address be combined directly and not use CustomerAddress.
- NOTE: No direct Customer-to-Address key exists in supplied schema. The only available relationship path is Customer -> CustomerAddress -> Address.
- Gold implementation options:
  - Option A (recommended and technically valid): use CustomerAddress bridge to obtain address attributes.
  - Option B: customer-only dimension with no address fields.
- Default build should use Option A unless edited.
- Retain relevant fields:
  - customer_id
  - company_name
  - title
  - email_address
  - city
  - postal_code

Dimension: dim_sales_person
- Source: Customer.SalesPerson.
- One row per unique salesperson.
- Business key: cleaned salesperson username.
- Hierarchy:
  - SalesPerson Username
- Attributes:
  - salesperson_name
  - salesperson_source_value

Dimension: dim_order
- Source: SalesOrderHeader.
- Move descriptive attributes out of fact:
  - sales_order_id
  - revision_number
  - status
  - ship_method
  - comment
- Exclude additive financial metrics.

Dimension: dim_product
- Source:
  - Product
  - ProductCategory
  - ProductModelProductDescription
  - ProductDescription
  - ProductModel
- Join path:
  - Product.ProductCategoryID -> ProductCategory.ProductCategoryID
  - Product.ProductModelID -> ProductModel.ProductModelID
  - ProductModel.ProductModelID -> ProductModelProductDescription.ProductModelID
  - ProductModelProductDescription.ProductDescriptionID -> ProductDescription.ProductDescriptionID
- Filter ProductModelProductDescription to `culture='en'`.
- Parent-child category handling:
  - Build category and subcategory attributes from ProductCategory self-join.
- Relevant attributes:
  - product_id
  - product_number
  - color
  - size
  - weight
  - standard_cost
  - list_price
  - product_description.description
  - category_name surrogate (derived category key structure)
  - subcategory_name surrogate (derived category key structure)
  - sell_start_date
  - sell_end_date
  - discontinued_date
- NOTE:
  - ProductCategory contains only IDs and parent IDs; no category name column exists.
  - ProductModel contains no Name column.
  - Therefore category names and model names cannot be populated from current schema. Expose IDs and hierarchy levels unless additional attributes become available.

Fact: fact_sales_order
- Grain:
  - One row per SalesOrderDetail.
- Join:
  - SalesOrderDetail -> SalesOrderHeader via SalesOrderID.
- Foreign keys:
  - order_date_key
  - ship_date_key
  - customer_key
  - salesperson_key
  - product_key
  - order_key
- Measures retained:
  - order_qty
  - unit_price
  - unit_price_discount
  - line_sales_amount = order_qty * unit_price
  - discount_amount = order_qty * unit_price * unit_price_discount
  - net_sales_amount = line_sales_amount - discount_amount
  - subtotal
  - tax_amt
  - freight
- Fact-level business attributes:
  - due_date
- Fact classification:
  - SalesOrderHeader = transactional header fact source.
  - SalesOrderDetail = transactional line fact source.

## Test

All tests append one record into `Tables/test/test_results`:
- Schema:
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
- Bronze vs Silver row counts.
- Expectation:
  - Variance within 1%.
- Applies to all source tables.

2. Gold Dimension PK Not Null
- Validate:
  - dim_customer.customer_id
  - dim_product.product_id
  - dim_order.sales_order_id
  - dim_sales_person.salesperson_name
  - dim_order_date.date_key
  - dim_ship_date.date_key

3. Gold Dimension PK Uniqueness
- Validate uniqueness of all dimension business keys.

4. Referential Integrity
- fact_sales_order.product_key exists in dim_product.
- fact_sales_order.customer_key exists in dim_customer.
- fact_sales_order.order_key exists in dim_order.
- fact_sales_order.salesperson_key exists in dim_sales_person.
- fact_sales_order.order_date_key exists in dim_order_date.
- fact_sales_order.ship_date_key exists in dim_ship_date.

5. Business Rule Sanity Checks
- net_sales_amount >= 0.
- order_qty > 0.
- unit_price >= 0.
- average discount percentage between 0 and 100.
- max discount percentage between 0 and 100.
- ship_date >= order_date when ship_date is populated.

## Semantic model

Storage mode:
- Direct Lake.

Tables:
- dim_order_date
- dim_ship_date
- dim_customer
- dim_sales_person
- dim_order
- dim_product
- fact_sales_order

Relationships:
- fact_sales_order -> dim_order_date
- fact_sales_order -> dim_ship_date
- fact_sales_order -> dim_customer
- fact_sales_order -> dim_sales_person
- fact_sales_order -> dim_product
- fact_sales_order -> dim_order

Hierarchies:

dim_order_date:
- Year > Quarter > Month > Date

dim_ship_date:
- Year > Quarter > Month > Date

dim_customer:
- City > Customer

dim_product:
- Category > Subcategory > Product
- If category names remain unavailable, use category_id > product_id hierarchy.

Measures:
- Total Sales = SUM(net_sales_amount)
- Gross Sales = SUM(line_sales_amount)
- Total Discount Amount = SUM(discount_amount)
- Total Orders = DISTINCTCOUNT(sales_order_id)
- Total Quantity = SUM(order_qty)
- Average Sales = AVERAGE(net_sales_amount)
- Maximum Sales = MAX(net_sales_amount)
- Average Order Value = DIVIDE([Total Sales],[Total Orders])
- Discount % = DIVIDE([Total Discount Amount],[Gross Sales])
- Average Discount % = AVERAGE(unit_price_discount)
- Maximum Discount % = MAX(unit_price_discount)
- Average Freight = AVERAGE(freight)
- Maximum Freight = MAX(freight)

Analytics focus:
- Regional performance by city and postal code.
- Monthly sales trends.
- Highest discounting salespeople.
- Product performance.
- Order fulfillment trends.

## Report

Page 1: Executive Sales Overview
- KPI cards:
  - Total Sales
  - Average Sales
  - Maximum Sales
  - Total Orders
  - Discount %
- Monthly sales trend line chart.
- Sales by salesperson bar chart.
- Top products by sales.

Page 2: Regional Performance
- Filled map or bubble map using city and postal code.
- Color scale based on Total Sales.
- Drill-through for customer and order detail.
- Average Sales by city.
- Maximum Sales by city.
- Regional ranking table showing high- and low-performing regions.

Page 3: Orders and Discounts
- Orders by status.
- Orders by ship method.
- Salesperson discount leaderboard.
- Average Discount % by salesperson.
- Maximum Discount % by salesperson.
- Order detail matrix.

Page 4: Product Performance
- Product sales ranking.
- Product category/subcategory hierarchy analysis.
- Sales trends by product.
- Discount impact by product.

Page 5: Data Quality
- Test result summary.
- PASS/FAIL counts.
- Referential integrity exceptions.
- Row-count reconciliation results.

## Data Agent

Role:
- Sales Performance Analytics Agent for the SalesLT reporting solution.

Domain knowledge:
- Understands customers, products, orders, salespeople, discounts, shipping activity, regional performance, and sales trends.
- Uses only the approved semantic model.

Instructions:
- Answer questions strictly from the semantic model.
- Prioritize business-friendly explanations.
- Always identify the filters, dates, and dimensions used in calculations.
- For trend questions, compare current and prior periods when possible.
- For ranking questions, return top and bottom performers.
- Highlight unusual discount behavior and sales outliers.
- Explain whether metrics are gross sales, net sales, or discount-adjusted sales.
- If data required for an answer is unavailable in the model, explicitly state what is missing.
- Never infer geographic regions beyond available city/postal-code data.
- Never fabricate category names or model names because those attributes do not exist in the supplied source schema.

Starter questions:
- Which cities generate the highest sales?
- Which cities generate the lowest sales?
- What are the monthly sales trends?
- What is the average and maximum sales value by month?
- Which salespeople provide the largest discounts?
- What is the average discount percentage by salesperson?
- Which products generate the most revenue?
- Which customers generate the most sales?
- How many orders were shipped late?
- What are the top and bottom performing regions this quarter?

Guardrails:
- Use only semantic-model measures and dimensions.
- Do not expose password_hash or password_salt information.
- Do not reveal row-level sensitive values unless permitted by report security.
- Do not create metrics that are not defined in the semantic model.
- Clearly distinguish facts from estimates.
- Cite dimension filters used in every analytical answer.
