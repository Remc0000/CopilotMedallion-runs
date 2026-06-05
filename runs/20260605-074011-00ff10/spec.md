# Run Spec 20260605-073923-154f5e

## Updated specs

### Iteration 1 — 2026-06-05 07:42:33Z — failed layer: bronze (run: 20260605-074011-00ff10)
- **Root cause (1-line summary)**: Bronze completed without producing discoverable schema-backed Delta tables, causing Silver to fail with "prior layer produced no discoverable tables".
- **Cross-table audit**:
  - Address: yes — subject to the same write/discovery mechanism as all Bronze tables.
  - Customer: yes — subject to the same write/discovery mechanism as all Bronze tables.
  - CustomerAddress: yes — subject to the same write/discovery mechanism as all Bronze tables.
  - Product: yes — subject to the same write/discovery mechanism as all Bronze tables.
  - ProductCategory: yes — subject to the same write/discovery mechanism as all Bronze tables.
  - ProductDescription: yes — subject to the same write/discovery mechanism as all Bronze tables.
  - ProductModel: yes — subject to the same write/discovery mechanism as all Bronze tables.
  - ProductModelProductDescription: yes — subject to the same write/discovery mechanism as all Bronze tables.
  - SalesOrderDetail: yes — subject to the same write/discovery mechanism as all Bronze tables.
  - SalesOrderHeader: yes — subject to the same write/discovery mechanism as all Bronze tables.
- **Fix approach**: GENERALIZE — the failure is systemic and affects discoverability for every Bronze table, not a single table-specific schema issue.
- **What was changed**:
  - Tightened Bronze write requirements to mandate one successful `saveAsTable('bronze.<table>')` per source table.
  - Added post-write validation that each expected Bronze table is discoverable via Spark catalog metadata before notebook completion.
  - Added a hard-fail condition if any expected Bronze table is missing or if fewer than 10 Bronze tables are discoverable.

## Inputs
- Workspace: `373889eb-1531-49df-9b0c-474976350c90`
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
- Target Lakehouse: **o**

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
- Use defensive column references and assert required columns exist before use.
- Use alias-prefixed joins only inside the immediate join/select scope; materialize flat columns afterward.
- Assert groupBy/agg columns exist before aggregation.
- For REST/API calls, use `if x is None: raise RuntimeError(...)` before any `.get()` access.
- Execute `CREATE SCHEMA IF NOT EXISTS bronze`, `silver`, `gold`, and `test` before writes.
- Write all outputs with schema-qualified `saveAsTable('<schema>.<table>')`.
- Never write target lakehouse outputs via raw abfss `.save()`.
- Parameterize workspace, lakehouse, schema, run_id, and table names in notebook parameter cells.
- Use idempotent overwrite patterns with Delta and `overwriteSchema=true`.
- Use error-loud try/except blocks that call `_save_error(layer, e)` and re-raise.
- Process tables in isolated loops with independent read-transform-write logic.
- Every notebook code cell must begin with a short explanatory comment block.

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

ALSO REQUIRE for every generated notebook: EACH code cell must start with a short markdown comment block (Python `# ---` divider + 1-3 lines of `# ` comments) describing what the cell is doing and why — never emit a cell with no leading comment. Example:
```
# ---
# Read raw bronze table and apply schema enforcement.
# Drops rows where any required key is null.
df = spark.read.table(...)
```
Keep the comments human-readable, not the code repeated in prose. The goal: someone scrolling through the notebook in Fabric can understand each block at a glance.

## Bronze

- Land each source table 1:1 into the `bronze` schema:
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
- Preserve source schema and datatypes.
- Add ingestion metadata:
  - `_run_id`
  - `_ingested_at`
  - `_source_table`
  - `_bronze_ts`
- Partitioning:
  - salesorderheader: partition by year(orderdate)
  - salesorderdetail: partition by salesorderid hash/bucket strategy if supported
  - remaining tables: no partitioning due to small dimension size
- Write mode:
  - Delta overwrite with schema evolution enabled.
- Capture row counts per table in notebook summary output.
- Discovery requirements (mandatory):
  - Create schema with `CREATE SCHEMA IF NOT EXISTS bronze` before any write.
  - Read each source table independently and write exactly one corresponding managed table using `saveAsTable('bronze.<table_name>')`.
  - Expected discoverable output tables are:
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
  - After each write, validate discoverability using catalog metadata (`spark.catalog.tableExists('bronze.<table_name>')` or equivalent).
  - Before notebook completion, verify all 10 expected Bronze tables exist and are queryable.
  - If any expected Bronze table is missing, raise a RuntimeError naming the missing table(s).
  - Do not treat the Bronze layer as successful unless at least 10 discoverable Bronze tables were written.

## Silver

Standard transformations for all tables:
- Rename columns to snake_case.
- Preserve business keys.
- Add:
  - `_silver_ts`
  - `_record_source`
  - `_run_id`
- Remove exact duplicates.
- Standardize timestamps to UTC-compatible timestamp type.
- Retain rowguid and modified_date for lineage unless explicitly excluded in Gold.

Deduplication strategy:
- address: dedupe on `address_id`, keep latest `modified_date`.
- customer: dedupe on `customer_id`, keep latest `modified_date`.
- customeraddress: dedupe on composite (`customer_id`,`address_id`), keep latest `modified_date`.
- product: dedupe on `product_id`, keep latest `modified_date`.
- productcategory: dedupe on `product_category_id`, keep latest `modified_date`.
- productdescription: dedupe on `product_description_id`, keep latest `modified_date`.
- productmodel: dedupe on `product_model_id`, keep latest `modified_date`.
- productmodelproductdescription: dedupe on (`product_model_id`,`product_description_id`,`culture`), keep latest `modified_date`.
- salesorderheader: dedupe on `sales_order_id`, keep latest `modified_date`.
- salesorderdetail: dedupe on `sales_order_detail_id`, keep latest `modified_date`.

Silver business enhancements:
- Derive product status flags:
  - is_discontinued
  - is_currently_sellable
- Create normalized salesperson helper field from customer.sales_person:
  - extract username portion from `<domain>\username`
  - remove trailing numeric suffix where present (example: `jillian0` → `jillian`)
  - store as `sales_person_clean`
- Create product category helper columns:
  - category_id
  - subcategory_id
  - parent_product_category_id
- OPTIMIZE and VACUUM according to Fabric best practices after successful writes.

## Gold

Target star schema required by user request.

Dimensions:

1. gold.dim_order_date
- Source: salesorderheader.order_date
- Grain: one row per calendar date.
- Include hierarchy:
  - Year
  - Quarter
  - Month
  - Date
- Attributes:
  - date_key
  - full_date
  - day/month/quarter/year fields
  - month_name
  - fiscal placeholders if future expansion is needed

2. gold.dim_ship_date
- Source: salesorderheader.ship_date
- Grain: one row per ship date.
- Include hierarchy:
  - Year
  - Quarter
  - Month
  - Date

3. gold.dim_customer
- Source: customer + address
- User requirement: do not use customeraddress as intermediary.
- NOTE: schema does not contain a direct relationship between Customer and Address. CustomerAddress is the only available bridge. Requested design cannot be implemented exactly from available keys.
- Implemented approach:
  - Build customer dimension from Customer.
  - Optionally enrich with address only if business confirms a deterministic join path.
  - Current gold model should expose customer attributes:
    - customer_id
    - company_name
    - title
    - suffix
    - email_address
- Regional reporting limitation:
  - Address contains City and PostalCode but no customer-to-address relationship exists without CustomerAddress.
  - To support regional analysis, use CustomerAddress bridge during ETL while presenting a flattened customer dimension to report consumers.

4. gold.dim_sales_person
- Source: customer.sales_person
- One row per cleaned salesperson.
- Attributes:
  - sales_person_key
  - sales_person_name
  - original_sales_person
- Hierarchy:
  - SalesPerson

5. gold.dim_order
- Source: salesorderheader
- Move descriptive attributes out of fact:
  - sales_order_id
  - revision_number
  - status
  - ship_method
  - comment
  - credit_card_approval_code
- Keep transactional amounts in fact.

6. gold.dim_product
- Source:
  - product
  - productcategory
  - productmodelproductdescription (culture='en')
  - productdescription
  - productmodel
- NOTE:
  - ProductModel table only contains ProductModelID and does not contain a Name column.
  - Requested ModelName cannot be populated from available schema.
  - Use ProductDescription.Description as the primary descriptive text.
- Category handling:
  - Resolve parent-child category structure.
  - Expose:
    - category_id
    - subcategory_id
    - category_name_placeholder
    - subcategory_name_placeholder
  - Because ProductCategory contains IDs only and no category names, names cannot be populated without additional source data.
- Relevant attributes:
  - product_id
  - product_number
  - description
  - color
  - size
  - weight
  - standard_cost
  - list_price
  - sell_start_date
  - sell_end_date
  - discontinued_date
  - is_discontinued

Fact:

7. gold.fact_sales_order
- Grain: one sales order line.
- Source:
  - salesorderheader
  - salesorderdetail
- Join:
  - sales_order_id
- Foreign keys:
  - order_date_key
  - ship_date_key
  - customer_id
  - product_id
  - sales_person_key
  - order_key
- Measures stored:
  - order_qty
  - unit_price
  - unit_price_discount
  - extended_sales_amount = order_qty * unit_price
  - discount_amount = order_qty * unit_price * unit_price_discount
  - net_sales_amount = order_qty * unit_price * (1 - unit_price_discount)
  - subtotal
  - tax_amt
  - freight
- Regional reporting:
  - Region is represented by City and PostalCode from Address via CustomerAddress enrichment during ETL.
  - Surface city/postal geography in customer dimension for map visuals.

Fact vs dimension rationale:
- SalesOrderDetail is the transactional fact source.
- SalesOrderHeader provides order-level context.
- Customer, Product, SalesPerson, Order, and Date entities are dimensions.
- CustomerAddress and ProductModelProductDescription are bridge/junction tables used during transformation, not exposed in the semantic layer.

## Test

Write all results to `test.test_results`:
- Columns:
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

1. Row Count Reconciliation
- Bronze vs Silver counts for each table.
- PASS when variance <= 1%.

2. Gold Dimension PK Not Null
- dim_customer.customer_id
- dim_product.product_id
- dim_order.sales_order_id
- dim_sales_person.sales_person_key
- dim_order_date.date_key
- dim_ship_date.date_key

3. Gold Dimension PK Uniqueness
- Validate uniqueness for all dimension primary keys.

4. Referential Integrity
- fact_sales_order.customer_id -> dim_customer.customer_id
- fact_sales_order.product_id -> dim_product.product_id
- fact_sales_order.order_key -> dim_order.sales_order_id
- fact_sales_order.order_date_key -> dim_order_date.date_key
- fact_sales_order.ship_date_key -> dim_ship_date.date_key
- fact_sales_order.sales_person_key -> dim_sales_person.sales_person_key

5. Business Rule: Discount Percentage Range
- Calculated discount percentage must be between 0 and 100%.

6. Business Rule: Net Sales Validation
- net_sales_amount <= extended_sales_amount.

7. Business Rule: Geographic Coverage
- At least one populated city value exists in customer geography attributes.

8. Business Rule: Order Date Logic
- ship_date is null or ship_date >= order_date.

## Semantic model

Storage mode:
- Direct Lake

Model tables:
- dim_order_date
- dim_ship_date
- dim_customer
- dim_sales_person
- dim_order
- dim_product
- fact_sales_order

Relationships:
- fact_sales_order.order_date_key -> dim_order_date.date_key
- fact_sales_order.ship_date_key -> dim_ship_date.date_key
- fact_sales_order.customer_id -> dim_customer.customer_id
- fact_sales_order.product_id -> dim_product.product_id
- fact_sales_order.sales_person_key -> dim_sales_person.sales_person_key
- fact_sales_order.order_key -> dim_order.sales_order_id

Hierarchies:
- Order Date: Year > Quarter > Month > Date
- Ship Date: Year > Quarter > Month > Date
- Customer Geography: City > Postal Code
- Product Category: Category > Subcategory
- SalesPerson hierarchy (single level)
- Order Status hierarchy where useful

Explicit measures:
- Total Sales = SUM(fact_sales_order[net_sales_amount])
- Gross Sales = SUM(fact_sales_order[extended_sales_amount])
- Total Discount Amount = SUM(fact_sales_order[discount_amount])
- Discount % = DIVIDE([Total Discount Amount],[Gross Sales])
- Average Sales = AVERAGE(fact_sales_order[net_sales_amount])
- Maximum Sales = MAX(fact_sales_order[net_sales_amount])
- Total Orders = DISTINCTCOUNT(fact_sales_order[sales_order_id])
- Average Order Value = DIVIDE([Total Sales],[Total Orders])
- Maximum Order Value = MAXX(VALUES(fact_sales_order[sales_order_id]),CALCULATE(SUM(fact_sales_order[net_sales_amount])))
- Average Discount % = AVERAGE(fact_sales_order[unit_price_discount])
- Products Sold = SUM(fact_sales_order[order_qty])

Required reporting attributes:
- Month-over-month sales trend.
- Regional performance.
- Salesperson discount performance.
- Product performance.

## Report

Page 1 — Executive Sales Overview
- KPI cards:
  - Total Sales
  - Average Sales
  - Maximum Sales
  - Total Orders
  - Average Order Value
- Monthly sales trend line chart.
- Sales by product category visual.
- Top products by sales bar chart.

Page 2 — Regional Performance
- Filled map or Azure map.
- Bubble size: Total Sales.
- Color scale: Average Sales.
- Tooltip:
  - Total Sales
  - Average Sales
  - Maximum Sales
  - Total Orders
- City ranking table.
- High-performing vs low-performing regions chart.

Page 3 — Orders and Operations
- Order status distribution.
- Orders by ship method.
- Monthly order volume trend.
- Order detail matrix with drill-through.

Page 4 — Discount and Salesperson Analysis
- Salesperson ranking by Discount %.
- Salesperson ranking by Discount Amount.
- Scatter plot:
  - X = Discount %
  - Y = Total Sales
  - Details = SalesPerson
- Table highlighting largest discounts.

Page 5 — Product Performance
- Product sales ranking.
- Category/Subcategory drill-down.
- Gross vs Net Sales comparison.
- Product lifecycle analysis using sell/discontinued dates.

Page 6 — Data Quality
- Test result summary.
- Pass/fail counts.
- Referential integrity status.
- Row count reconciliation results.

## Data Agent

Role:
- Enterprise Sales Performance Analyst for the SalesLT reporting environment.
- Answer questions using only the approved semantic model.
- Focus on sales performance, customer activity, product performance, order trends, discounts, geography, and salesperson effectiveness.

Domain instructions:
- Use measures from the semantic model whenever possible.
- Prefer net sales for revenue discussions.
- Explain calculations in business language.
- Highlight trends, anomalies, and outliers.
- Compare current selections against overall averages when meaningful.
- Surface both average and maximum values when discussing performance.
- For geography questions, use city and postal-code-based regional views.
- When discussing discounts, always include Discount % and Discount Amount where available.
- Distinguish gross sales from net sales.

Starter questions:
- Which regions generated the highest and lowest sales?
- What are the average and maximum sales by city?
- How have sales trended month over month?
- Which salespeople provide the largest discounts?
- Which salespeople generate the highest net sales after discounts?
- What products contribute most to total sales?
- Which product categories are growing fastest?
- What is the average order value by month?
- Which orders generated the highest revenue?
- Are discounts increasing or decreasing over time?

Guardrails:
- Only answer from semantic model data.
- Do not fabricate category names, model names, customer locations, or missing attributes.
- If requested data is unavailable because it is not present in the model, explicitly state that limitation.
- Do not expose password_hash or password_salt fields under any circumstance.
- Never infer customer identity beyond available business attributes.
- Use aggregate analysis rather than row-level disclosure when possible.
- Explain uncertainty when dimensional relationships are incomplete.
- Refuse requests to reveal credentials, hashes, salts, or hidden system metadata.