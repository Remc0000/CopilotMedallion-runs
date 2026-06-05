# Run Spec 20260605-065435-02ea02

## Inputs
- Workspace: `1dbcce9f-93a7-4f2d-b6a7-fc408557bfee`
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
- Target Lakehouse: **n**

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
- Use defensive column references and validate required columns before every join, filter, aggregation, window, and derived-column calculation.
- After every join, immediately project alias-prefixed columns into flat column names.
- Assert groupBy and aggregation columns exist before execution.
- For REST/API calls, use explicit `if x is None: raise RuntimeError(...)` before any `.get()` access.
- Do not use `saveAsTable`; write Delta files directly to Lakehouse paths.
- Every notebook must begin with parameter cells for workspace, source, target, run_id, and layer configuration.
- Use idempotent overwrite patterns with `mode("overwrite")` and `overwriteSchema=true`.
- All try/except blocks must call `_save_error(layer, e)` (or `_save_error(layer, e, table=tbl)` for table loops) and re-raise appropriately.
- Process source tables independently in Bronze and Silver.
- All notebook code cells must begin with a short human-readable comment block.

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
  - Print a final summary line `print(json.dumps({"bronze_results": {<table>: {"rows": N, "path": ...}, ...}}))` listing every table actually written. Use this as a self-check.
  - Raise (not just log) if zero tables were written by the end of the notebook.
- Same rule applies recursively to Silver (`Tables/silver/<table>`) and Gold (`Tables/gold/<table>` + `Tables/test/test_results`).

ALSO REQUIRE for every generated notebook: EACH code cell must start with a short markdown comment block (Python `# ---` divider + 1-3 lines of `# ` comments) describing what the cell is doing and why — never emit a cell with no leading comment.

## Bronze

- Land each source table independently into `Tables/bronze/<table_name_lower>`.
- Preserve source schema exactly.
- Add metadata columns:
  - `_run_id`
  - `_bronze_ingested_at`
  - `_source_table`
- Partition large transactional tables by year derived from:
  - SalesOrderHeader.OrderDate
  - SalesOrderDetail.ModifiedDate
- Non-transactional master tables may be written without partitioning.
- Write mode: overwrite with schema overwrite enabled.
- Expected Bronze outputs:
  - bronze/address
  - bronze/customer
  - bronze/customeraddress
  - bronze/product
  - bronze/productcategory
  - bronze/productdescription
  - bronze/productmodel
  - bronze/productmodelproductdescription
  - bronze/salesorderdetail
  - bronze/salesorderheader

## Silver

Common standards:
- Rename all columns to snake_case.
- Retain source business keys.
- Add:
  - `_silver_loaded_at`
  - `_source_dt`
  - `_run_id`
- Remove exact duplicates.
- Standardize text trimming and null handling.
- Exclude binary product image column (`thumbnail_photo`) from downstream analytical dimensions unless specifically required.

Deduplication strategy:
- address: dedupe on `address_id`
- customer: dedupe on `customer_id`
- customeraddress: dedupe on (`customer_id`, `address_id`)
- product: dedupe on `product_id`
- productcategory: dedupe on `product_category_id`
- productdescription: dedupe on `product_description_id`
- productmodel: dedupe on `product_model_id`
- productmodelproductdescription: dedupe on (`product_model_id`, `product_description_id`, `culture`)
- salesorderheader: dedupe on `sales_order_id`
- salesorderdetail: dedupe on (`sales_order_id`, `sales_order_detail_id`)

Silver business enrichment:
- Customer:
  - Create cleaned email field.
  - Normalize salesperson values.
- SalesPerson preparation:
  - Extract username from `sales_person`.
  - When value follows `domain\username`, retain only username portion.
  - Create normalized `sales_person_username`.
- Product:
  - Create flags:
    - `is_discontinued`
    - `is_sellable_currently`
  - Derive lifecycle indicators from sell_start_date, sell_end_date, discontinued_date.
- SalesOrderHeader:
  - Create order_year, order_month, order_date_key.
  - Create ship_date_key when ship_date exists.
- Optimize and vacuum Silver Delta outputs after successful write.

Note:
- User requested Product dimension include ProductModel.Name. The provided ProductModel schema contains only `ProductModelID`, `rowguid`, and `ModifiedDate`; no Name column exists. Product dimension will retain ProductModelID and description enrichment, unless ProductModel is expanded in source later.

## Gold

Star schema requested by user.

Dimensions:

1. dim_order_date
- Source: sales_order_header.order_date
- Grain: one row per calendar date.
- Include standard hierarchy:
  - Year
  - Quarter
  - Month
  - Date
- Surrogate key: order_date_key.

2. dim_ship_date
- Source: sales_order_header.ship_date
- Grain: one row per ship date.
- Include hierarchy:
  - Year
  - Quarter
  - Month
  - Date
- Surrogate key: ship_date_key.

3. dim_customer
- Source: customer + address.
- User explicitly requested bypassing CustomerAddress.
- Because Customer and Address have no direct join key in the provided schema, a valid relationship cannot be derived without CustomerAddress.
- Gold implementation:
  - Create customer dimension from Customer attributes:
    - customer_id
    - company_name
    - title
    - suffix
    - email_address
  - Optionally expose address attributes separately if user later permits CustomerAddress bridge usage.
- Modeling note:
  - Requested customer-address merge is not technically supported by available keys unless CustomerAddress participates in the join.

4. dim_salesperson
- Source: customer.sales_person.
- Grain: one row per salesperson username.
- Attributes:
  - salesperson_key
  - salesperson_username
  - original_sales_person
- Username extraction:
  - adventure-works\jillian0 → jillian0

5. dim_order
- Source: sales_order_header.
- Grain: one row per sales_order_id.
- Retain dimensional attributes:
  - sales_order_id
  - revision_number
  - status
  - ship_method
  - credit_card_approval_code
  - comment
- Remove additive financial measures from this dimension.

6. dim_product
- Source:
  - product
  - productcategory
  - productmodelproductdescription (culture='en')
  - productdescription
  - productmodel
- Grain: one row per product_id.
- Include:
  - product_id
  - product_number
  - color
  - size
  - weight
  - standard_cost
  - list_price
  - category_id
  - parent_category_id
  - description
  - product_model_id
  - lifecycle attributes
- Category hierarchy:
  - Category
  - Subcategory
- Because ProductCategory contains only IDs, expose category and parent category identifiers.
- If descriptive category names become available later, populate hierarchy labels from those columns.

Facts:

fact_sales_order
- Grain: one row per sales order detail line.
- Source:
  - sales_order_header
  - sales_order_detail
- Join:
  - sales_order_id
- Foreign keys:
  - customer_id
  - salesperson_key
  - product_id
  - order_date_key
  - ship_date_key
  - sales_order_id (to dim_order)
- Measures:
  - order_qty
  - unit_price
  - unit_price_discount
  - gross_sales_amount = order_qty * unit_price
  - discount_amount = order_qty * unit_price * unit_price_discount
  - net_sales_amount = gross_sales_amount - discount_amount
  - subtotal
  - tax_amt
  - freight

Regional reporting note:
- User requested regional performance and map visuals.
- Available Address schema contains City and PostalCode only.
- Region-level reporting will use City as the highest available geographic level.
- No state, province, country, or territory attributes exist in source.

## Test

All tests append results to:
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

Tests:

1. Row Count Reconciliation
- Compare Bronze vs Silver row counts.
- Expected variance <= 1%.

2. Gold Dimension PK Not Null
- dim_customer.customer_id
- dim_product.product_id
- dim_order.sales_order_id
- dim_order_date.order_date_key
- dim_ship_date.ship_date_key
- dim_salesperson.salesperson_key

3. Gold Dimension PK Uniqueness
- Validate uniqueness of every dimension primary key.

4. Referential Integrity
- fact_sales_order.product_id exists in dim_product.
- fact_sales_order.customer_id exists in dim_customer.
- fact_sales_order.salesperson_key exists in dim_salesperson.
- fact_sales_order.order_date_key exists in dim_order_date.
- fact_sales_order.ship_date_key exists in dim_ship_date.
- fact_sales_order.sales_order_id exists in dim_order.

5. Business Rule Sanity Check
- net_sales_amount >= 0.
- discount_amount <= gross_sales_amount.
- average discount percentage between 0 and 100.
- ship_date >= order_date when ship_date is populated.

## Semantic model

Storage mode:
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

OrderDate hierarchy:
- Year
- Quarter
- Month
- Date

ShipDate hierarchy:
- Year
- Quarter
- Month
- Date

Product hierarchy:
- Parent Category ID
- Category ID
- Product

Customer hierarchy:
- City (if customer-address relationship is later enabled)
- Company Name

Measures:

- Total Sales = SUM(net_sales_amount)
- Gross Sales = SUM(gross_sales_amount)
- Total Discount Amount = SUM(discount_amount)
- Average Sales = AVERAGE(net_sales_amount)
- Maximum Sales = MAX(net_sales_amount)
- Total Orders = DISTINCTCOUNT(sales_order_id)
- Average Order Value = DIVIDE([Total Sales],[Total Orders])
- Discount % = DIVIDE([Total Discount Amount],[Gross Sales])
- Average Discount % = AVERAGE(unit_price_discount)
- Maximum Discount % = MAX(unit_price_discount)
- Average Quantity = AVERAGE(order_qty)
- Maximum Quantity = MAX(order_qty)

## Report

Page 1 — Executive Sales Overview
- KPI cards:
  - Total Sales
  - Average Sales
  - Maximum Sales
  - Total Orders
  - Average Order Value
- Monthly sales trend line chart using OrderDate hierarchy.
- Sales by Product chart.
- Sales by SalesPerson chart.

Page 2 — Geographic Performance
- Bubble map using City from Address when customer geography becomes available.
- Total Sales by City.
- Average Sales by City.
- Maximum Sales by City.
- Top 10 and Bottom 10 performing cities.
- Geographic performance matrix.

Note:
- Customer geography depends on CustomerAddress relationship. Without it, city mapping cannot be connected to customer sales. If CustomerAddress usage is permitted, enable city-level reporting through CustomerAddress → Address joins.

Page 3 — Orders and Discounts
- Order status distribution.
- Sales by ship method.
- Discount % trend over time.
- SalesPerson ranking by:
  - Total Discount Amount
  - Average Discount %
  - Maximum Discount %
- Detailed order table.

Page 4 — Data Quality
- Test pass/fail summary.
- Referential integrity results.
- Row count reconciliation results.
- Failed test details.

## Data Agent

Role:
- Sales Performance Analytics Agent for the SalesLT reporting model. Answer questions about sales performance, order activity, discount behavior, product performance, customer activity, shipping trends, and geographic sales patterns using only the semantic model.

Domain hints:
- Fact table is Fact Sales Order.
- Financial analysis should prioritize Net Sales.
- Discount analysis should use Discount % and Discount Amount measures.
- Date trend analysis should default to OrderDate unless the user explicitly asks about shipping.
- Geographic analysis is based on City-level data when available.
- Salesperson analysis uses normalized usernames.

Starter questions:
- What were total, average, and maximum sales last month?
- Which salespeople provided the largest discounts?
- Which products generated the highest net sales?
- What are the monthly sales trends over the last year?
- Which cities have the highest sales performance?
- Which cities have the lowest sales performance?
- What is the average discount percentage by salesperson?
- Which orders generated the largest net sales?
- What products have been discontinued?
- How do shipping patterns compare by ship method?

Guardrails:
- Use only semantic model data.
- Never invent geographic regions not present in the model.
- Clearly distinguish Gross Sales, Discount Amount, and Net Sales.
- Use OrderDate for sales trends unless ShipDate is explicitly requested.
- If customer-to-address linkage is unavailable, state that geographic attribution is limited by source relationships.
- Explain calculations when returning rankings or discount percentages.
- Do not expose PasswordHash or PasswordSalt fields under any circumstance.
- Prefer aggregated reporting over row-level customer disclosure.
