# Run Spec 20260604-100109-de5088

## Inputs
- Workspace: `1779f71c-1dd7-4707-af35-94419229e9ac`
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
- Use defensive column references and validate required columns before every join, filter, aggregation, window, and derived-column calculation.
- Use alias-prefixed joins during join construction and materialize flat column names immediately after joins.
- Assert required columns exist before every groupBy/agg operation.
- For REST/API responses, use `if x is None: raise RuntimeError(...)` before any `.get(...)` call.
- Never use `saveAsTable`; write Delta directly to lakehouse paths.
- Every notebook must begin with parameter cells for workspace, lakehouse IDs, run_id, source/target paths, and configuration.
- Use idempotent overwrite patterns with `overwriteSchema=true`.
- Wrap processing in error-loud try/except blocks that call `_save_error(layer, e)` and re-raise.
- Process source tables independently and record per-table outcomes.
- Every notebook code cell must start with a short comment block describing intent.

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

Land each source table 1:1 into `Tables/bronze/<table_name>`.

For every table:
- Preserve source schema exactly.
- Add metadata columns:
  - `_run_id`
  - `_ingested_at`
  - `_source_table`
  - `_bronze_ts`
- Write as Delta overwrite with schema evolution enabled.
- Partition large transactional tables by year extracted from business date:
  - salesorderheader: orderdate year
  - salesorderdetail: salesorderid hash bucket or unpartitioned if volume is small
- Dimension-style tables may remain unpartitioned.

Bronze outputs:
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

Standardize all tables:
- Convert column names to snake_case.
- Add `_silver_ts`, `_source_dt`, `_run_id`.
- Trim strings and normalize empty strings to null where appropriate.
- Remove exact duplicate rows.
- Preserve rowguid for lineage but exclude from Gold unless needed.

Deduplication keys:
- address: `address_id`
- customer: `customer_id`
- customeraddress: (`customer_id`, `address_id`)
- product: `product_id`
- productcategory: `product_category_id`
- productdescription: `product_description_id`
- productmodel: `product_model_id`
- productmodelproductdescription: (`product_model_id`, `product_description_id`, `culture`)
- salesorderheader: `sales_order_id`
- salesorderdetail: (`sales_order_id`, `sales_order_detail_id`)

Business standardization:
- Derive `sales_person_username` from `sales_person`.
- When value contains `\`, retain only username portion and remove numeric suffix when present (example: `adventure-works\jillian0` → `jillian`).
- Create product lifecycle flags:
  - `is_discontinued`
  - `is_currently_sellable`
- Create order timing metrics:
  - `days_to_ship`
  - `days_until_due`

NOTE:
- User requested ProductModel Name as model name, but ProductModel schema only contains `product_model_id`, `rowguid`, and `modified_date`.
- Gold product dimension will therefore not contain a model name unless the source schema is expanded. Retain `product_model_id` as the model reference.

Optimize:
- OPTIMIZE silver.salesorderheader
- OPTIMIZE silver.salesorderdetail
- OPTIMIZE major Gold source dimensions after write

## Gold

Target star schema required by user.

Dimensions

1. dim_order_date
- Source: salesorderheader.order_date
- Surrogate key: order_date_key
- Attributes:
  - date
  - year
  - quarter
  - month
  - month_name
  - week
  - day
  - day_name
- Hierarchy:
  - Year → Quarter → Month → Date

2. dim_ship_date
- Source: salesorderheader.ship_date
- Surrogate key: ship_date_key
- Attributes similar to order date.
- Hierarchy:
  - Year → Quarter → Month → Date

3. dim_customer
- Source: customer + address
- User explicitly requested direct combination without CustomerAddress.
- Join approach:
  - Use SalesOrderHeader.CustomerID → Customer.CustomerID
  - Use SalesOrderHeader.ShipToAddressID → Address.AddressID
  - Build customer-address combinations actually used in orders.
- Retain relevant fields:
  - customer_id
  - company_name
  - title
  - email_address
  - city
  - postal_code
- Geography available only at city/postal-code level.
- NOTE:
  - No state, country, region, latitude, or longitude exists in source.
  - Regional reporting will therefore be modeled using city as the lowest available geography unless enrichment data is added.

4. dim_sales_person
- Source: customer.sales_person
- Distinct salesperson records.
- Attributes:
  - sales_person_key
  - sales_person_username
  - sales_person_original
- Hierarchy:
  - Sales Person

5. dim_order
- Source: salesorderheader
- Move non-measure order attributes from fact:
  - sales_order_id
  - revision_number
  - status
  - ship_method
  - credit_card_approval_code
  - comment
- Keep transactional amounts out of the dimension.

6. dim_product
- Source:
  - product
  - productcategory
  - productmodel
  - productmodelproductdescription (culture='en')
  - productdescription
- ProductCategory parent-child handling:
  - Parent category becomes Category.
  - Child category becomes Subcategory.
- Retain relevant attributes:
  - product_id
  - product_number
  - color
  - size
  - weight
  - category
  - subcategory
  - description
  - product_model_id
  - list_price
  - standard_cost
  - is_discontinued
- Hierarchy:
  - Category → Subcategory → Product
- NOTE:
  - ProductModel does not contain a Name column in supplied schema.
  - model_name cannot be populated.

Fact

fact_sales_order
- Grain:
  - One row per SalesOrderDetail line.
- Sources:
  - salesorderdetail
  - salesorderheader
- Joins:
  - salesorderdetail.sales_order_id = salesorderheader.sales_order_id
- Foreign keys:
  - order_date_key
  - ship_date_key
  - customer_key
  - sales_person_key
  - order_key
  - product_key
- Measures stored:
  - order_qty
  - unit_price
  - unit_price_discount
  - gross_sales_amount = order_qty * unit_price
  - discount_amount = order_qty * unit_price * unit_price_discount
  - net_sales_amount = gross_sales_amount - discount_amount
  - tax_amt
  - freight
- Regional performance:
  - Aggregate by customer city.
- Monthly trend support:
  - OrderDate relationship drives trend analysis.
- Discount analysis:
  - SalesPerson linked through customer dimension path.

## Test

All tests append results to:
- gold.test_results

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
- Bronze vs Silver counts per table.
- PASS when variance <= 1%.

2. Gold Dimension PK Not Null
- Validate surrogate/business keys populated.
- Tables:
  - dim_customer
  - dim_product
  - dim_order
  - dim_sales_person
  - dim_order_date
  - dim_ship_date

3. Gold Dimension PK Uniqueness
- Validate unique keys in all dimensions.

4. Referential Integrity
- fact_sales_order.product_key exists in dim_product
- fact_sales_order.customer_key exists in dim_customer
- fact_sales_order.sales_person_key exists in dim_sales_person
- fact_sales_order.order_key exists in dim_order
- fact_sales_order.order_date_key exists in dim_order_date
- fact_sales_order.ship_date_key exists in dim_ship_date

5. Business Rule Validation
- net_sales_amount <= gross_sales_amount
- discount_amount >= 0
- unit_price >= 0
- order_qty > 0
- ship_date >= order_date when ship_date is not null

## Semantic model

Mode:
- Direct Lake

Tables:
- Fact Sales Order
- Customer
- Product
- Sales Person
- Order
- Order Date
- Ship Date

Relationships:
- Fact Sales Order → Customer
- Fact Sales Order → Product
- Fact Sales Order → Sales Person
- Fact Sales Order → Order
- Fact Sales Order → Order Date
- Fact Sales Order → Ship Date

Dimension hierarchies:

Customer
- City → Company Name

Product
- Category → Subcategory → Product ID

Order Date
- Year → Quarter → Month → Date

Ship Date
- Year → Quarter → Month → Date

Measures:

Sales
- Total Sales = SUM(net_sales_amount)
- Gross Sales = SUM(gross_sales_amount)
- Total Discount Amount = SUM(discount_amount)
- Average Sales = AVERAGE(net_sales_amount)
- Maximum Sales = MAX(net_sales_amount)

Discounts
- Discount % = DIVIDE([Total Discount Amount],[Gross Sales])
- Average Discount % = AVERAGE(unit_price_discount)
- Maximum Discount % = MAX(unit_price_discount)

Orders
- Order Count = DISTINCTCOUNT(sales_order_id)
- Average Order Quantity = AVERAGE(order_qty)
- Maximum Order Quantity = MAX(order_qty)

Customers
- Customer Count = DISTINCTCOUNT(customer_id)

Products
- Product Count = DISTINCTCOUNT(product_id)

Time Intelligence
- Sales MTD
- Sales QTD
- Sales YTD

Geography
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
  - Order Count
  - Discount %
- Monthly sales trend line chart.
- Sales by category clustered column chart.
- Top customers table.

Page 2 — Regional Performance
- Filled map or bubble map using customer city.
- Color by Total Sales.
- Tooltips:
  - Total Sales
  - Average Sales
  - Maximum Sales
  - Order Count
- Top and bottom performing cities visual.
- Regional sales trend by month.

NOTE:
- True geographic regions are unavailable in provided schema.
- City is the only geography available without external enrichment.

Page 3 — Discount & Salesperson Analysis
- Salesperson ranking by Discount %.
- Salesperson ranking by Total Discount Amount.
- Scatter chart:
  - Discount % vs Total Sales.
- Product category discount analysis.
- Top discounted products.

Page 4 — Orders & Fulfillment
- Orders by status.
- Ship method distribution.
- Days to Ship analysis.
- Order quantity distribution.
- Monthly order volume trend.

Page 5 — Data Quality
- Test pass/fail summary.
- Row count reconciliation matrix.
- Referential integrity results.
- Failed test details table.

## Data Agent

Role:
- AI Sales Performance Analyst for the SalesAnalytics semantic model.
- Specializes in sales trends, customer performance, discount analysis, product performance, fulfillment metrics, and city-level geographic reporting.

Domain guidance:
- Use measures from the semantic model whenever available.
- Prefer net sales metrics over raw line values.
- Explain calculations clearly.
- Distinguish between gross sales, discount amount, and net sales.
- Treat city as the available geographic level.
- Do not infer countries, states, territories, or regions not present in the data.

Starter questions:
- Which cities generate the highest total sales?
- Which cities have the lowest sales performance?
- What are the average and maximum sales values by city?
- How have monthly sales trends changed over time?
- Which salespeople provide the largest discounts?
- Which salespeople have the highest discount percentages?
- Which product categories generate the most revenue?
- Which products receive the largest discounts?
- What is the average order value by month?
- Which ship methods are associated with the largest sales volume?

Guardrails:
- Answer only using the semantic model.
- Do not fabricate geography beyond city and postal code.
- Clearly state when requested information is unavailable.
- Use aggregate results instead of exposing sensitive customer-level details unless explicitly requested.
- Prefer validated measures over ad-hoc calculations.
- Surface data-quality issues when relevant tests fail.
- Explain filters and time periods used in any result.
- Never invent missing ProductModel names because the source schema does not contain them.
