# Run Spec 20260604-102523-3841e4

## Inputs
- Workspace: `d547615a-511c-436c-b6e0-c95f688a7ead`
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
- Target Lakehouse: **a**

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
- Apply alias-prefixed joins only within the join statement; materialize flat column names immediately after joins.
- Assert groupBy and aggregation columns exist before execution.
- For REST/API responses use `if x is None: raise RuntimeError(...)` before any `.get(...)`.
- Do not use `saveAsTable`; write Delta directly to Lakehouse paths.
- All notebooks must begin with parameter cells for workspace, lakehouse, paths, run_id, and layer.
- Use idempotent overwrite patterns with `mode("overwrite")` and `overwriteSchema=true`.
- Use explicit try/except blocks that call `_save_error(layer, e)` and re-raise after logging.
- Process source tables independently and persist outputs per table.
- Emit discoverable Delta outputs under Tables/bronze, Tables/silver, Tables/gold, and Tables/test.
- Every notebook cell must start with a short comment block describing intent.

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

ALSO REQUIRE for every generated notebook: EACH code cell must start with a short markdown comment block (Python `# ---` divider + 1-3 lines of `# ` comments) describing what the cell is doing and why — never emit a cell with no leading comment.

## Bronze

Land each source table unchanged into `Tables/bronze/<table_name_lower>`.

Source-to-bronze tables:
- address
- customer
- customeraddress
- product
- productcategory
- productdescription
- productmodel
- productmodelproductdescription
- salesorderdetail
- salesorderheader

Bronze standards:
- Preserve source schema and datatypes.
- Add metadata columns:
  - `_run_id`
  - `_ingested_at`
  - `_source_table`
  - `_bronze_ts`
- Overwrite write pattern with schema evolution enabled.
- Partition large transactional tables:
  - salesorderheader: partition by order year derived from OrderDate.
  - salesorderdetail: partition by SalesOrderID hash bucket or order year if available after enrichment.
- Small master tables remain unpartitioned.
- Persist all tables as Delta format.
- Capture row counts in notebook summary output.

## Silver

Standardize all tables:
- Rename columns to snake_case.
- Add:
  - `_silver_ts`
  - `_source_dt`
  - `_is_current`
- Remove duplicate records using latest modified_date where available.
- Retain rowguid only where useful for lineage.

Deduplication keys:
- address: address_id
- customer: customer_id
- customeraddress: (customer_id, address_id)
- product: product_id
- productcategory: product_category_id
- productdescription: product_description_id
- productmodel: product_model_id
- productmodelproductdescription: (product_model_id, product_description_id, culture)
- salesorderheader: sales_order_id
- salesorderdetail: sales_order_detail_id

Silver business preparation:
- Customer:
  - Standardize email casing.
  - Trim text fields.
- Address:
  - Normalize city values.
- Product:
  - Derive is_discontinued from discontinued_date.
  - Derive is_active_product.
- SalesOrderHeader:
  - Derive order_year, order_month, order_date_key.
  - Derive ship_date_key when ship_date exists.
- SalesOrderDetail:
  - Derive line_discount_amount = order_qty * unit_price * unit_price_discount.
  - Derive gross_line_amount = order_qty * unit_price.
  - Derive net_line_amount = order_qty * unit_price * (1 - unit_price_discount).

Performance:
- OPTIMIZE silver transactional tables.
- ZORDER:
  - sales_order_id
  - customer_id
  - product_id

## Gold

User-requested target schema.

Dimension: dim_order_date
- Source: sales_order_header.order_date.
- Grain: one row per calendar date.
- Include hierarchy:
  - Year
  - Quarter
  - Month
  - Day
- Date key based on order_date.

Dimension: dim_ship_date
- Source: sales_order_header.ship_date.
- Grain: one row per calendar date.
- Include hierarchy:
  - Year
  - Quarter
  - Month
  - Day

Dimension: dim_customer
- Source:
  - customer
  - address
- User requested direct combination without customeraddress bridge.
- NOTE: schema does not provide a direct Customer ↔ Address relationship. CustomerAddress contains the only relationship keys. To satisfy reporting requirements, build dim_customer primarily from Customer. Address attributes may only be attached if a deterministic relationship is later supplied. Otherwise retain customer attributes only:
  - customer_id
  - company_name
  - title
  - suffix
  - email_address
- If user chooses later, CustomerAddress can be used to enrich geography.

Dimension: dim_salesperson
- Source: customer.sales_person.
- Create unique salesperson dimension.
- Extract username:
  - If value follows `domain\username`, keep username portion only.
  - Remove trailing numeric suffix when present for display name (example: jillian0 → jillian).
- Attributes:
  - salesperson_key
  - salesperson_username
  - salesperson_display_name
- Hierarchy:
  - Domain (if available)
  - Salesperson

Dimension: dim_order
- Source: sales_order_header.
- Move descriptive order attributes out of fact:
  - sales_order_id
  - revision_number
  - status
  - ship_method
  - credit_card_approval_code
  - comment
- Exclude measures and dates.

Dimension: dim_product
- Source:
  - product
  - productcategory
  - productmodel
  - productmodelproductdescription
  - productdescription
- User-requested combination.
- Filter ProductModelProductDescription to culture='en'.
- Include:
  - product_id
  - product_number
  - color
  - size
  - weight
  - standard_cost
  - list_price
  - model identifier
  - description
  - category
  - subcategory
  - active/discontinued status
- Parent-child category handling:
  - Parent category becomes Category.
  - Child category becomes Subcategory.
- NOTE: ProductModel schema contains only ProductModelID, rowguid, ModifiedDate. Requested ProductModel.Name → modelname is not available in source. Use ProductModelID as model reference unless additional source columns are provided.
- Hierarchies:
  - Category → Subcategory → Product
  - Product status → Product

Fact: fact_sales_order
- Grain:
  - One row per sales order detail line.
- Source:
  - sales_order_detail
  - sales_order_header
- Join:
  - sales_order_id
- Foreign keys:
  - order_date_key
  - ship_date_key
  - customer_id
  - salesperson_key
  - product_id
  - sales_order_id (dim_order)
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
- Derived measures:
  - discount_percentage
  - average_order_value
  - maximum_order_value
- Reporting focus:
  - monthly trends
  - salesperson discount behavior
  - order performance
  - regional analysis where geography becomes available

## Test

Write all results to:
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

1. Row count reconciliation
- Bronze vs Silver.
- Variance tolerance: ±1%.

2. Gold dimension PK not null
- dim_order_date.date_key
- dim_ship_date.date_key
- dim_customer.customer_id
- dim_salesperson.salesperson_key
- dim_order.sales_order_id
- dim_product.product_id

3. Gold dimension PK uniqueness
- Validate unique primary keys for all dimensions.

4. Referential integrity
- fact_sales_order.customer_id exists in dim_customer.
- fact_sales_order.product_id exists in dim_product.
- fact_sales_order.salesperson_key exists in dim_salesperson.
- fact_sales_order.sales_order_id exists in dim_order.
- fact_sales_order.order_date_key exists in dim_order_date.
- fact_sales_order.ship_date_key exists in dim_ship_date when populated.

5. Business-rule sanity checks
- Net sales amount >= 0.
- Discount percentage between 0 and 100.
- Order quantity > 0.
- Maximum sales amount >= average sales amount.
- No future order_date beyond execution date.

## Semantic model

Mode:
- Direct Lake

Tables:
- Fact Sales Order
- Customer
- Product
- SalesPerson
- Order
- OrderDate
- ShipDate

Relationships:
- Fact Sales Order → Customer
- Fact Sales Order → Product
- Fact Sales Order → SalesPerson
- Fact Sales Order → Order
- Fact Sales Order → OrderDate
- Fact Sales Order → ShipDate

Hierarchies:
- OrderDate: Year → Quarter → Month → Day
- ShipDate: Year → Quarter → Month → Day
- Product: Category → Subcategory → Product
- SalesPerson: Domain → SalesPerson
- Customer: CompanyName → CustomerID

Measures:
- Total Sales = SUM(net_sales_amount)
- Gross Sales = SUM(gross_sales_amount)
- Total Discount Amount = SUM(discount_amount)
- Discount % = DIVIDE([Total Discount Amount],[Gross Sales])
- Average Sales = AVERAGE(fact_sales_order[net_sales_amount])
- Maximum Sales = MAX(fact_sales_order[net_sales_amount])
- Total Orders = DISTINCTCOUNT(fact_sales_order[sales_order_id])
- Average Order Value = DIVIDE([Total Sales],[Total Orders])
- Maximum Order Value = MAXX(VALUES(fact_sales_order[sales_order_id]), CALCULATE(SUM(fact_sales_order[net_sales_amount])))
- Average Discount % = AVERAGE(fact_sales_order[discount_percentage])
- Maximum Discount % = MAX(fact_sales_order[discount_percentage])
- Total Quantity = SUM(order_qty)

## Report

Page 1: Executive Sales Overview
- KPI cards:
  - Total Sales
  - Average Sales
  - Maximum Sales
  - Total Orders
  - Average Order Value
- Monthly sales trend line chart.
- Sales by product category column chart.
- Top products by sales bar chart.

Page 2: Regional Performance
- Filled map and bubble map visuals.
- Color by Total Sales.
- Tooltips:
  - Average Sales
  - Maximum Sales
  - Order Count
- Regional ranking table.
- NOTE: Current schema lacks a valid customer-to-address relationship without CustomerAddress. Regional visuals require geography enrichment. If CustomerAddress is later approved for geography mapping, use City and PostalCode from Address.

Page 3: Salesperson Discount Analysis
- Top discounting salespeople.
- Discount % by salesperson.
- Average Discount % KPI.
- Maximum Discount % KPI.
- Scatter chart:
  - Sales vs Discount %.
- Matrix:
  - Salesperson
  - Orders
  - Sales
  - Discount Amount

Page 4: Orders and Fulfillment
- Orders by status.
- Ship method analysis.
- Order value distribution.
- Monthly order counts.
- Average and maximum order value visuals.

Page 5: Data Quality
- Test result summary.
- Failed test detail table.
- Record counts by layer.
- Referential integrity status.

## Data Agent

Role:
- Enterprise Sales Performance Analyst for the SalesLT reporting platform.

Domain instructions:
- Answer questions using only the Direct Lake semantic model.
- Prioritize sales, order, product, customer, salesperson, discount, and date analysis.
- Explain calculations using business terminology.
- Always reference measures rather than raw fact aggregations when available.
- Highlight data limitations when geography cannot be reliably derived.
- Distinguish between gross sales, discount amount, and net sales.
- Use date hierarchies when discussing trends.
- Surface significant outliers in sales or discounts.

Starter questions:
- What were total, average, and maximum sales this month?
- Which products generated the highest net sales?
- Which salespeople offered the largest discounts?
- How has sales performance changed month over month?
- What are the top categories by revenue?
- Which orders had the highest sales value?
- What is the average discount percentage by salesperson?
- Which products have the highest discount rates?
- What are the trends in order volume over time?
- Which customers generated the most revenue?

Guardrails:
- Do not invent geographic relationships not present in the model.
- Do not infer customer addresses unless they exist in the published dimensions.
- Clearly state when requested attributes are unavailable.
- Do not expose PasswordHash or PasswordSalt fields.
- Do not return row-level sensitive values unless explicitly requested and permitted.
- Use only semantic-model entities and measures.
- Prefer aggregated insights over raw record dumps.
- Identify whether a result is based on OrderDate or ShipDate when relevant.
