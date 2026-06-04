# Run Spec 20260604-094054-ac79f4

## Inputs
- Workspace: `6b270c7d-4c29-43c3-a8de-4debee058dd2`
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
- Target Lakehouse: **SalesAnalytics**

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
- After every join, immediately project alias-prefixed columns into flat names before downstream use.
- Before every groupBy/agg, assert grouping and aggregation columns exist.
- For REST/API calls, use explicit `if x is None: raise RuntimeError(...)` checks before any `.get()`.
- Do not use `saveAsTable`; write Delta files directly to lakehouse paths.
- Every notebook must start with parameter cells for workspace, lakehouse, run_id, source paths, and target paths.
- Use idempotent overwrite patterns with `overwriteSchema=true`.
- All try/except blocks must call `_save_error(layer, e)` (or `_save_error(layer, e, table=tbl)` in loops) and then re-raise.
- Process source tables independently per layer and record results.
- Every code cell must begin with a short comment block explaining purpose and intent.

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

- Land each source table 1:1 into `Tables/bronze/<table_name_lower>`.
- Preserve source schema exactly; no business transformations.
- Add ingestion metadata:
  - `_run_id`
  - `_ingested_at`
  - `_source_table`
  - `_bronze_ts`
- Write mode: Delta overwrite with schema evolution enabled.
- Partitioning:
  - `salesorderheader`: partition by year derived from `OrderDate`.
  - `salesorderdetail`: partition by year derived from `ModifiedDate`.
  - Remaining tables: unpartitioned due to expected small size.
- Persist all audit columns (`rowguid`, `ModifiedDate`) unchanged.
- Record row counts written for each table and emit bronze summary JSON.

## Silver

Standard actions for all tables:
- Convert column names to snake_case.
- Preserve source business keys.
- Add `_silver_ts`, `_run_id`, `_is_current`.
- Remove exact duplicate rows.
- Validate required key columns before writes.
- OPTIMIZE output tables after successful write.

Per-table deduplication strategy:
- address: dedupe on `address_id`, latest `modified_date`.
- customer: dedupe on `customer_id`, latest `modified_date`.
- customer_address: dedupe on composite key (`customer_id`,`address_id`), latest `modified_date`.
- product: dedupe on `product_id`, latest `modified_date`.
- product_category: dedupe on `product_category_id`, latest `modified_date`.
- product_description: dedupe on `product_description_id`, latest `modified_date`.
- product_model: dedupe on `product_model_id`, latest `modified_date`.
- product_model_product_description: dedupe on (`product_model_id`,`product_description_id`,`culture`), latest `modified_date`.
- sales_order_header: dedupe on `sales_order_id`, latest `modified_date`.
- sales_order_detail: dedupe on (`sales_order_id`,`sales_order_detail_id`), latest `modified_date`.

Additional Silver enrichments:
- Derive product lifecycle flags:
  - `is_discontinued`
  - `is_currently_sellable`
- Derive discount percentage at line level:
  - `discount_pct = unit_price_discount / nullif(unit_price,0)`
- Derive order calendar attributes from `order_date`.
- Normalize salesperson source value into helper field for Gold dimension creation.
- NOTE: User requested ProductModel Name as model name. The provided schema contains only `ProductModelID`, `rowguid`, and `ModifiedDate`; no model name column exists. Gold will therefore expose ProductModelID unless an additional source containing model names is supplied.

## Gold

Target star schema per user requirements.

Dimensions

1. `dim_order_date`
- Source: `sales_order_header.order_date`
- One row per calendar date.
- Hierarchy:
  - Year
  - Quarter
  - Month
  - Date
- Include standard calendar attributes.

2. `dim_ship_date`
- Source: `sales_order_header.ship_date`
- One row per calendar date.
- Hierarchy:
  - Year
  - Quarter
  - Month
  - Date

3. `dim_customer`
- Source: Customer + Address.
- User requested bypassing CustomerAddress bridge.
- Because Customer and Address have no direct join key, create customer dimension using:
  - Customer attributes from Customer.
  - Preferred address from SalesOrderHeader joins to BillToAddressID and CustomerID where available.
- Keep relevant fields:
  - customer_id
  - company_name
  - email_address
  - city
  - postal_code
  - title
- Exclude password_hash, password_salt, rowguid.
- NOTE: Direct Customer→Address relationship does not exist in supplied schema. Address enrichment must be inferred through SalesOrderHeader address usage. CustomerAddress is intentionally not used per requirement.

4. `dim_salesperson`
- Source: Customer.SalesPerson.
- Distinct salesperson records.
- Transform values:
  - If format resembles `domain\username`, retain username portion.
  - Remove trailing numeric suffix where present for reporting-friendly display (example: `jillian0` → `jillian`).
- Hierarchy:
  - Domain (if present)
  - Salesperson

5. `dim_order`
- Source: SalesOrderHeader.
- Move descriptive order attributes out of fact:
  - sales_order_id
  - revision_number
  - status
  - ship_method
  - credit_card_approval_code
  - comment
- Retain customer and date keys only as dimension relationships.

6. `dim_product`
- Source:
  - Product
  - ProductCategory
  - ProductModel
  - ProductDescription
  - ProductModelProductDescription
- Filter ProductModelProductDescription to `culture='en'`.
- Product hierarchy:
  - Category
  - Subcategory
  - Product
- Parent-child category handling:
  - Parent category becomes Category.
  - Child category becomes Subcategory.
- Keep relevant attributes:
  - product_id
  - product_number
  - color
  - size
  - weight
  - standard_cost
  - list_price
  - description
  - product_model_id
  - model_name (only if future source supplies a name column)
  - category
  - subcategory
  - sellability flags
- NOTE: Category names are not present in supplied schema. Only category IDs are available. Use category/subcategory IDs unless additional descriptive source tables are added.

Fact

`fact_sales_order`
- Grain: one sales order line.
- Source: SalesOrderHeader + SalesOrderDetail.
- Keys:
  - order_date_key
  - ship_date_key
  - customer_key
  - salesperson_key
  - order_key
  - product_key
- Measures:
  - order_qty
  - unit_price
  - unit_price_discount
  - gross_sales_amount = order_qty * unit_price
  - discount_amount = order_qty * unit_price_discount
  - net_sales_amount = order_qty * (unit_price - unit_price_discount)
  - tax_amount
  - freight_amount
- Retain only analytical fields needed for reporting.

Regional reporting approach:
- Region is approximated using customer address city and postal code because no territory/region table exists.
- Map visuals should use city and postal code geography fields.
- If a future regional lookup table becomes available, replace geographic approximation with explicit region dimension.

## Test

All tests append results into:
- `Tables/test/test_results`

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
- Expected variance ≤ 1%.

2. Gold Dimension PK Null Check
- Verify no null keys in:
  - dim_customer
  - dim_product
  - dim_order
  - dim_salesperson
  - dim_order_date
  - dim_ship_date

3. Gold Dimension PK Uniqueness
- Verify dimension primary keys are unique.

4. Referential Integrity
- Every fact product key exists in dim_product.
- Every fact customer key exists in dim_customer.
- Every fact order key exists in dim_order.
- Every fact salesperson key exists in dim_salesperson.
- Every fact order date key exists in dim_order_date.
- Every fact ship date key exists in dim_ship_date.

5. Business Rule Validation
- Net sales amount >= 0.
- Discount percentage between 0 and 1.
- Gross sales amount >= net sales amount.
- Maximum discount percentage reported in semantic model equals fact calculation.

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

Hierarchies

dim_order_date:
- Year > Quarter > Month > Date

dim_ship_date:
- Year > Quarter > Month > Date

dim_product:
- Category > Subcategory > Product

dim_customer:
- City > Customer

dim_salesperson:
- Domain > Salesperson

Measures

- Total Sales = SUM(fact_sales_order[net_sales_amount])
- Gross Sales = SUM(fact_sales_order[gross_sales_amount])
- Total Discount Amount = SUM(fact_sales_order[discount_amount])
- Average Sales = AVERAGE(fact_sales_order[net_sales_amount])
- Maximum Sales = MAX(fact_sales_order[net_sales_amount])
- Total Orders = DISTINCTCOUNT(dim_order[sales_order_id])
- Total Quantity = SUM(fact_sales_order[order_qty])
- Average Discount % = AVERAGE(fact_sales_order[discount_pct])
- Maximum Discount % = MAX(fact_sales_order[discount_pct])
- Average Order Value = DIVIDE([Total Sales],[Total Orders])
- Sales per Customer = DIVIDE([Total Sales],DISTINCTCOUNT(dim_customer[customer_id]))

Geographic settings:
- City categorized as City.
- Postal Code categorized as Postal Code.

## Report

Page 1: Executive Sales Overview
- KPI cards:
  - Total Sales
  - Average Sales
  - Maximum Sales
  - Total Orders
- Monthly sales trend line chart.
- Sales by product hierarchy column chart.
- Top customers by sales table.

Page 2: Regional Performance
- Filled map or Azure Maps visual using city/postal code.
- Bubble size: Total Sales.
- Color scale: Average Sales.
- Tooltip:
  - Total Sales
  - Average Sales
  - Maximum Sales
  - Total Orders
- Bottom 10 and Top 10 locations visual.

Page 3: Orders & Discounts
- Monthly order trend.
- Salesperson ranking by Average Discount %.
- Salesperson ranking by Maximum Discount %.
- Scatter plot:
  - Total Sales vs Average Discount %
- Order detail matrix.

Page 4: Product Performance
- Product hierarchy drill-down.
- Gross vs Net Sales comparison.
- Top discounted products.
- Product profitability proxy:
  - Net Sales minus Standard Cost estimate.

Page 5: Data Quality
- Test pass/fail summary.
- Referential integrity status.
- Row-count reconciliation metrics.
- Latest refresh and run information.

## Data Agent

Role:
- Sales Performance Analytics Agent for SalesAnalytics semantic model.
- Specializes in sales trends, order analysis, discount behavior, customer performance, product performance, and geographic performance.

Domain hints:
- Use measures from the semantic model whenever available.
- Prefer net sales for revenue discussions.
- Geographic analysis is based on city and postal-code data.
- Salesperson performance is derived from Customer.SalesPerson.
- Product hierarchy uses category/subcategory/product structure.

Starter questions:
- Which cities generated the highest total sales?
- Which cities generated the lowest total sales?
- What are the monthly sales trends over time?
- What is the average and maximum sales amount by month?
- Which salespeople offer the largest average discounts?
- Which salespeople offer the largest maximum discounts?
- Which products generate the highest net sales?
- Which customers place the largest orders?
- How many orders were placed each month?
- Which geographic areas are underperforming compared to average?

Guardrails:
- Answer only from the semantic model.
- State when requested information is unavailable.
- Do not infer missing territories, regions, countries, or category names not present in the model.
- Prefer aggregated results over row-level disclosure.
- Do not expose password_hash, password_salt, rowguid, or other excluded technical fields.
- Explain calculation logic for sales, discount, average, and maximum metrics when requested.
- Identify whether geographic conclusions are based on city/postal-code approximations.
- When comparing periods, explicitly state date range and filter context used.
