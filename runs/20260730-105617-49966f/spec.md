# Run Spec 20260730-105433-edc8b4

## Inputs
- Workspace: `692949a8-3b8c-41ea-9617-5280c45b1a0f`
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

### Cross-cutting code rules
- Use defensive column references and assert required columns exist immediately before every join, filter, derived expression, `groupBy`, and aggregation.
- After every join, project alias-qualified columns and rename them to unique flat names before subsequent transformations.
- Before every `groupBy(...).agg(...)`, assert that all grouping and aggregation input columns exist.
- Handle REST responses defensively: use `if x is None: raise RuntimeError(...)` before any `.get()` call.
- Create target schemas with `CREATE SCHEMA IF NOT EXISTS bronze`, `silver`, `gold`, and `test`.
- Write schema-qualified Delta tables with `saveAsTable('<schema>.<table>')`; never use raw ABFSS `.save()` for the schema-enabled target Lakehouse.
- Put workspace, source Lakehouse, target Lakehouse, run ID, table list, and configurable behavior in notebook parameter cells.
- Use idempotent overwrite patterns, including `mode('overwrite')` and `option('overwriteSchema','true')`; replace only the current build's intended tables.
- Use error-loud `try/except` handling that calls `_save_error(layer, e)` or `_save_error(layer, e, table=tbl)` and then re-raises according to the per-table isolation policy.
- Every generated notebook code cell must start with a short human-readable comment block consisting of a Python `# ---` divider and one to three `# ` comment lines explaining what the cell does and why. Never emit an uncommented code cell.

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

Rule K — Resilience to partial output: every layer MUST write Delta tables the next layer can discover.
- The build pipeline runs each layer's notebook then inspects the lakehouse for the layer's output Delta tables before generating the next layer. If Bronze runs "successfully" (Spark Completed) but writes zero discoverable tables in the `bronze` schema, the build hard-fails with "prior layer produced no discoverable tables".
- To guarantee discoverability, the Bronze notebook MUST:
  - Write via `df.write.format('delta').mode('overwrite').option('overwriteSchema','true').partitionBy(...).saveAsTable(f"bronze.<flat>")` (after `spark.sql('CREATE SCHEMA IF NOT EXISTS bronze')`) for every source table, where `<flat>` is the lowercased last segment of `table_relative_path`. NEVER write target tables with abfss `.save(path)` — on this SCHEMA-ENABLED lakehouse a raw .save() to `Tables/bronze/<t>` lands at a broken nested `Tables/Tables/bronze/<t>` path the discovery + SQL endpoint cannot see.
  - Print a final summary line `print(json.dumps({{"bronze_results": {{<table>: {{"rows": N, "path": ...}}, ...}}}}))` listing every table actually written. Use this as a self-check.
  - Raise (not just log) if zero tables were written by the end of the notebook.
- Same rule applies recursively to Silver (`silver.<table>`) and Gold (`gold.<table>` + `test.test_results`) — each via `CREATE SCHEMA IF NOT EXISTS` + saveAsTable.

Rule L — Disambiguate shared columns in join projections (avoid AMBIGUOUS_REFERENCE).
- When you `select(...)` directly off a join whose sides share a column name, selecting that column as a BARE string raises `[AMBIGUOUS_REFERENCE]` and cancels the whole Spark session. Typical Gold offenders: joining product `p` with product_model `m` (both expose `product_model_id`), or product `p` with product_category `pc` (both expose `product_category_id`), or any dimension built from several aliased source tables that carry the same key.
- Inside the SAME join+select expression the alias scope is still live, so reference EVERY shared/overlapping column with its alias and rename it explicitly: `F.col('p.product_model_id').alias('product_model_id')`, `F.col('pc.parent_product_category_id').alias('category_id')`. A bare string in a join `select` is ONLY safe for a column that exists on EXACTLY ONE side of the join.
- Before emitting a join's select list, enumerate the columns on each side; for any name present on more than one side, alias-qualify the side you want. When in doubt in a multi-table join, alias-qualify ALL columns in the select list — it is always safe and never ambiguous.
- This is the in-join counterpart to Rule A: dotted alias references (`F.col('p.col')`) are valid ONLY inside the join/select that introduces the alias; once the joined DataFrame has been materialized by that select, switch back to plain, already-renamed column names (Rule A).
- Concrete failure to avoid: `.select(F.col('p.product_id').alias('product_id'), 'product_number', 'product_model_id', ...)` after `p.join(m, ...)` — `product_model_id` exists on both `p` and `m`, so it MUST be `F.col('p.product_model_id').alias('product_model_id')` (or the `m.` side), never the bare `'product_model_id'`.

### Notebook cell documentation
Every generated notebook must apply this pattern to every code cell:
```
# ---
# Read raw bronze table and apply schema enforcement.
# Drops rows where any required key is null.
df = spark.read.table(...)
```

## Bronze
- Create `bronze` and land one Delta table per selected source table, preserving source names and source data types.
- Add `_run_id`, `_ingested_at`, `_source_lakehouse`, `_source_table`, and `_source_modified_at`; populate the last field from `ModifiedDate`.
- Preserve source column names in Bronze so it remains a faithful raw landing layer. Snake-case conversion occurs in Silver.
- Process all ten tables independently and record source and written row counts in the notebook result summary.
- Use full-table idempotent overwrite because no source change-tracking or ingestion watermark column was supplied. `ModifiedDate` can support a later incremental design, but is not sufficient by itself to detect deletions.
- Do not partition the relatively small master/junction tables: `customeraddress`, `productdescription`, `customer`, `productcategory`, `productmodel`, `productmodelproductdescription`, `product`, and `address`.
- Do not partition `salesorderdetail`; its available columns do not provide a natural date partition.
- Partition `salesorderheader` by a derived `_order_year` based on `OrderDate` only if volume justifies partitioning; otherwise leave it unpartitioned to avoid small files. Retain the original `OrderDate`.
- Write each result with schema-qualified `saveAsTable('bronze.<table>')`.
- Table roles observed from the supplied columns:
  - Transaction header: `salesorderheader`, keyed by `SalesOrderID`.
  - Transaction lines: `salesorderdetail`, keyed by `SalesOrderDetailID` and linked by `SalesOrderID`.
  - Master data: `customer`, `address`, `product`, `productcategory`, `productmodel`, and `productdescription`.
  - Junctions: `customeraddress` and `productmodelproductdescription`.

## Silver
- Create cleaned, typed, snake-case Delta tables with the same table names under `silver`.
- Trim strings, convert blank strings to null, retain decimal precision, normalize booleans, and preserve source timestamps.
- Retain `rowguid` as a lineage attribute but do not use it as the business key.
- Add `_silver_ts`, `_run_id`, and `source_dt`, using `modified_date` when present.
- Deduplicate by the following keys, keeping the greatest non-null `modified_date`, then deterministic `rowguid`:
  - `customeraddress`: composite `(customer_id, address_id, address_type)`.
  - `salesorderdetail`: `sales_order_detail_id`; verify `(sales_order_id, sales_order_detail_id)` is also unique.
  - `productdescription`: `product_description_id`.
  - `customer`: `customer_id`.
  - `productcategory`: `product_category_id`.
  - `productmodel`: `product_model_id`.
  - `salesorderheader`: `sales_order_id`.
  - `productmodelproductdescription`: composite `(product_model_id, product_description_id, culture)`.
  - `product`: `product_id`.
  - `address`: `address_id`.
- Validate that quantities and monetary fields can be interpreted without coercion: `order_qty`, `unit_price`, `unit_price_discount`, `standard_cost`, `list_price`, `sub_total`, `tax_amt`, and `freight`.
- Treat `unit_price_discount` as a fractional rate for downstream calculations; retain the original value and flag values outside `[0,1]`.
- Normalize `culture` to lowercase for the English-description filter.
- Preserve customer `sales_person` for Gold parsing. Do not expose `password_hash` or `password_salt` beyond Silver.
- Run `OPTIMIZE` after writes where supported, prioritizing `salesorderheader`, `salesorderdetail`, and `product`; avoid unnecessary compaction of tiny tables.

## Gold
- Create a Direct Lake star schema under `gold`, using one fact row per sales-order detail.
- `gold.dim_order_date`:
  - Generate a continuous calendar covering the minimum through maximum non-null `salesorderheader.order_date`.
  - Use an integer `date_key` in `yyyyMMdd` form and include calendar year, quarter, month number/name, year-month, week, and day attributes.
- `gold.dim_ship_date`:
  - Generate a continuous calendar covering the minimum through maximum non-null `salesorderheader.ship_date`.
  - Use the same date-key convention and expose a Ship Date hierarchy: Year → Quarter → Month → Date.
  - Include an unknown member for unshipped orders with null `ship_date`.
- `gold.dim_customer`:
  - Use `customer_id` as the durable source key and retain relevant name, company, email, and phone attributes; exclude password and row GUID fields.
  - The requested direct `customer`-to-`address` join is not possible because neither table contains the other's key. To satisfy the requirement without using `customeraddress`, derive each customer's reporting address through `salesorderheader`: select the customer's latest order by `order_date`, then `sales_order_id`, and join its `ship_to_address_id` to `address.address_id`.
  - Include city, state/province, country/region, and postal code from that latest shipping address. Customers without orders receive unknown geography values.
  - This address is explicitly a current reporting location inferred from the latest order, not a historical customer master address.
- `gold.dim_sales_person`:
  - Derive distinct salespeople from `customer.sales_person`.
  - Remove the domain prefix through the final backslash and remove trailing numeric characters so `adventure-works\jillian0` becomes `jillian`, as explicitly requested.
  - Use a deterministic salesperson key based on the normalized username and include an `Unknown` member for null or blank values.
  - Note that removing trailing digits can merge distinct source usernames; retain the original value only as an internal lineage field for collision testing.
- `gold.dim_order`:
  - Use `sales_order_id` as the key.
  - Move suitable header-level descriptors here: revision number, status, online-order flag, purchase-order number, account number, ship method, and comment.
  - Do not place additive monetary values such as subtotal, tax, or freight in this dimension.
- `gold.dim_product`:
  - Start from `product`, keyed by `product_id`.
  - Join `productcategory` as the child/subcategory and self-join its `parent_product_category_id` to obtain the parent category.
  - Expose parent category as Category and child category as Subcategory. If no parent exists, treat the product's category name as Category and set Subcategory to `Uncategorized`.
  - Join `productmodel` by `product_model_id`, retaining only its `name` as `model_name`.
  - Filter `productmodelproductdescription` to `culture = 'en'`, deduplicate by `product_model_id` using latest `modified_date` then lowest `product_description_id`, and join `productdescription`, retaining only `description`.
  - Retain relevant product descriptors and pricing/cost attributes; omit thumbnail binary content and row GUID.
  - Expose Product hierarchy: Category → Subcategory → Model → Product.
- `gold.fact_sales_order`:
  - Inner join `salesorderdetail` to `salesorderheader` on `sales_order_id`.
  - Grain: one row per `sales_order_detail_id`.
  - Include foreign keys for order date, ship date, customer, salesperson, order, and product.
  - Derive the salesperson key through the order's `customer_id` and the normalized `customer.sales_person`.
  - Retain measures: order quantity, unit price, unit-price discount rate, gross line amount, discount amount, and net sales amount.
  - Calculate `gross_line_amount = order_qty * unit_price`, `discount_amount = gross_line_amount * unit_price_discount`, and `net_sales_amount = gross_line_amount - discount_amount`.
  - Do not repeat header `sub_total`, `tax_amt`, or `freight` on every line because summing them would overstate totals. If allocated totals are later required, add an explicit proportional allocation design.
- Add unknown dimension members and map unresolved/null fact foreign keys to those members.
- Write dimensions before the fact, then validate the written dimension keys before constructing fact foreign keys.
- Optimize the fact and large dimensions after successful schema-qualified writes.

## Test
- Create `test.test_results` with: `run_id`, `test_name`, `layer`, `table_name`, `status`, `actual`, `expected`, `details`, and `checked_at`.
- Append exactly one result row per individual table/test assertion; use `ERROR` when execution fails and preserve the exception in `details`.
- Row-count reconciliation:
  - Compare every Bronze table with its corresponding Silver table.
  - Expect Silver counts to be within 1% of Bronze after cleaning and deduplication; failures must report both counts and percentage variance.
  - Compare `fact_sales_order` with the count of valid Silver detail rows that match a Silver header.
- No-null dimension PKs:
  - Test keys in `dim_order_date`, `dim_ship_date`, `dim_customer`, `dim_sales_person`, `dim_order`, and `dim_product`.
- Unique dimension PKs:
  - Test that each Gold dimension has exactly one row per primary key, including only one unknown member.
- Referential integrity:
  - Test every fact order-date, ship-date, customer, salesperson, order, and product key against its corresponding dimension.
  - Unresolved source values must map to an unknown member rather than remain null.
- Business-rule sanity:
  - Assert `order_qty > 0`, `unit_price >= 0`, `unit_price_discount` is between `0` and `1`, and `net_sales_amount` is between `0` and `gross_line_amount`.
  - Assert non-null `ship_date` is not before `order_date`.
  - Assert Gold net sales summed by `sales_order_id` agrees with the recomputed Silver line calculation within decimal rounding tolerance.
  - Assert normalized salesperson usernames are not blank and report collisions where multiple non-null original salesperson values normalize to one username.

## Semantic model
- Create a Direct Lake semantic model over `gold.fact_sales_order` and all six Gold dimensions.
- Use single-direction, one-to-many relationships from each dimension to the fact.
- Relationships:
  - `dim_order_date[date_key]` → `fact_sales_order[order_date_key]`.
  - `dim_ship_date[date_key]` → `fact_sales_order[ship_date_key]`.
  - Customer, SalesPerson, Order, and Product keys → corresponding fact foreign keys.
- Mark `dim_order_date` and `dim_ship_date` as date tables.
- Create hierarchies:
  - Order Date: Year → Quarter → Month → Date.
  - Ship Date: Year → Quarter → Month → Date.
  - Customer Geography: Country/Region → State/Province → City → Customer.
  - Product: Category → Subcategory → Model → Product.
  - SalesPerson: Salesperson.
  - Order: Online Order Flag → Status → Order.
- Hide technical keys, lineage columns, row GUIDs, and raw fact calculation inputs not useful to report consumers.
- Format currency and percentage measures consistently and assign meaningful display folders.
- Explicit DAX measures:
  - `Total Sales = SUM(fact_sales_order[net_sales_amount])`
  - `Average Sales = AVERAGE(fact_sales_order[net_sales_amount])`
  - `Maximum Sale = MAX(fact_sales_order[net_sales_amount])`
  - `Order Count = DISTINCTCOUNT(fact_sales_order[order_key])`
  - `Average Discount % = DIVIDE(SUM(fact_sales_order[discount_amount]), SUM(fact_sales_order[gross_line_amount]))`
  - `Maximum Discount % = MAX(fact_sales_order[unit_price_discount])`
- Use weighted `Average Discount %` for commercial analysis rather than a simple average of line-level percentages.
- Add measure descriptions clarifying that sales metrics are net line sales and exclude header tax and freight.

## Report
- Apply a polished merchandise-oriented theme derived from the actual `productcategory` Category and Subcategory values. Use product imagery only as subtle design accents; do not expose the binary thumbnail field.
- Page 1 — **Sales Overview**:
  - KPI cards for Total Sales, Average Sales, Maximum Sale, and Order Count.
  - Monthly line chart using the Order Date hierarchy and Total Sales.
  - Category/Subcategory treemap or clustered bar chart using the Product hierarchy.
  - Online-order and order-status slicers.
- Page 2 — **Regional Performance**:
  - Azure Maps visual using country/region, state/province, city, and postal code from `dim_customer`; bubble size represents Total Sales and color scale identifies high- and low-performing regions.
  - Ranked regional bar chart for Total Sales, Average Sales, and Maximum Sale.
  - Drillable matrix using Customer Geography with Order Count and discount measures.
  - Clearly label geography as the customer's latest known shipping region inferred from order history.
- Page 3 — **Products, Discounts, and Salespeople**:
  - Product hierarchy decomposition tree for Total Sales.
  - Ranked salesperson bar chart by Average Discount % with Total Sales in the tooltip.
  - Matrix by salesperson and product category showing Total Sales, Maximum Sale, Average Discount %, and Maximum Discount %.
  - Scatter chart with Total Sales versus Average Discount %, sized by Order Count, to highlight high-discount salespeople.
- Page 4 — **Data Quality**:
  - PASS/FAIL/ERROR status cards sourced from `test.test_results`.
  - Test-result matrix by layer, table, and test name.
  - Failure-details table with checked timestamp and run ID.
  - Row-count variance chart for Bronze-to-Silver reconciliation.
- Use a consistent grid, restrained category-inspired palette, accessible contrast, dynamic titles, informative tooltips, synced slicers, and drill-through from region, salesperson, and product views.

## Data Agent
- Create an AISkill grounded only on the SalesAnalytics semantic model.
- Role: “You are a sales-performance analyst who explains regional, product, order, customer, and salesperson performance using governed Direct Lake measures.”
- Domain instructions:
  - Interpret Sales as net line sales after discount and before header tax and freight.
  - Use Order Date for sales trends unless the user explicitly requests shipping analysis.
  - Use Ship Date only for shipping questions.
  - Treat customer geography as the latest shipping location inferred through SalesOrderHeader, not a permanent or historical customer address.
  - Analyze category and subcategory through the parent-child product-category hierarchy.
  - Use weighted Average Discount % and distinguish it from Maximum Discount %.
  - Rank salespeople by normalized username.
- Starter questions:
  - Which regions have the highest and lowest total sales?
  - What are average and maximum sales by country, state, and city?
  - How are monthly sales and order counts trending?
  - Which product categories and subcategories generate the most sales?
  - Which products have the highest average discount percentage?
  - Which salespeople offer the largest discounts?
  - Are high salesperson discounts associated with higher sales?
  - Which regions have strong order volume but below-average sales?
  - How long does it typically take orders to ship?
  - Which data-quality tests failed in the latest run?
- Guardrails:
  - Use only certified semantic-model tables, relationships, hierarchies, and explicit measures.
  - Never expose password hashes, password salts, row GUIDs, email addresses, phone numbers, or other row-level personal data.
  - Do not infer a customer's residence from the reporting geography.
  - Do not sum header subtotal, tax, or freight across fact lines.
  - Do not claim causal relationships between discounts and sales; describe associations only.
  - State active filters, date basis, measure definition, and geography level in answers.
  - Refuse requests to identify individual customers; aggregate customer results to an appropriate regional or commercial level.
  - If data is missing, relationships are unresolved, or a test has failed, say so explicitly rather than fabricating an answer.
