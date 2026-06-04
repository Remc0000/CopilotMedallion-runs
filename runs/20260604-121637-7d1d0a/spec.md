# Run Spec 20260604-121533-af0e87

## Inputs
- Workspace: `120db309-94d0-4c4a-9183-504d81b9a3bf`
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
- Target Lakehouse: **e**

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
- Use defensive column references and validate required columns before every join, filter, groupBy, agg, window, and withColumn.
- After every join, immediately project alias-prefixed columns into flat names before downstream transformations.
- Assert aggregation and groupBy columns exist before execution.
- For REST/API responses, use `if x is None: raise RuntimeError(...)` before any `.get(...)` access.
- Never use `saveAsTable`; write Delta directly to lakehouse paths.
- Every notebook must begin with parameter cells for workspace, source lakehouse, target lakehouse, run_id, layer, and paths.
- Use idempotent overwrite patterns with `mode("overwrite")` and `overwriteSchema=true`.
- All exception handling must be error-loud: call `_save_error(layer, e)` (or `_save_error(layer, e, table=tbl)` in loops) and re-raise.
- Each code cell must start with a short comment block explaining purpose and business intent.
- Process source tables independently per layer wherever possible.
- Validate outputs exist before allowing downstream layers to execute.

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
  def _maybe(df, name, dtype='timestamp'):
      return F.col(name) if name in df.columns else F.lit(None).cast(dtype)
  Pick the `dtype` to match the surrounding expression (`'timestamp'` for date/time coalesces, `'string'` for text, `'double'` for numeric, etc.) so Spark can resolve the result type without ambiguity.
- Alternative pattern (when the helper genuinely cannot know the dtype) — filter `None`s at the call site BEFORE invoking the Spark function:
  candidates = [c for c in (_maybe(df, 'modified_date'), _maybe(df, 'order_date')) if c is not None]
  df = df.withColumn('source_dt', F.to_date(F.coalesce(*candidates, F.current_timestamp())))
  Either approach is acceptable, but never pass Python `None` directly into a Spark function.
- Applies to ALL optional-column lookups across Bronze, Silver, Gold — including audit-timestamp coalesces, optional-key joins, fallback string formatting, etc. This is a layer-agnostic rule.

Rule H — Per-table isolation; one table's failure must not cancel the Spark session for the rest.
- Spark cancels the entire session when one statement crashes. If your notebook builds a single chained plan that touches every source table (one big SELECT, one big DataFrame, one big SQL script), any one table's failure kills ALL tables.
- ALWAYS process source tables in a for-loop with per-table try/except handling and isolated writes.
- Do NOT build a single multi-table transformation chain for Bronze or Silver.
- Cross-table Gold joins occur only after all required Silver tables are successfully materialized.

Rule I — Optional audit columns on junction / bridge / view tables.
- Do not assume ModifiedDate or rowguid exists on every table.
- Junction-table deduplication must use composite business keys.
- Project only columns actually present in source objects.

Rule J — Validate column existence BEFORE the expensive transform.
- Assert required columns exist before joins, windows, aggregations, filters, and derived calculations.
- Revalidate after every select, rename, or drop operation.

Rule K — Resilience to partial output: Bronze MUST write Delta tables that the next layer can discover.
- Bronze writes Delta tables to `Tables/bronze/<table>`.
- Silver writes Delta tables to `Tables/silver/<table>`.
- Gold writes Delta tables to `Tables/gold/<table>`.
- Emit completion summaries and fail if zero discoverable tables are produced.

## Bronze

Land each source table unchanged into `Tables/bronze/<table_name>`.

Per-table actions:
- Preserve source schema exactly.
- Add metadata columns:
  - `_run_id`
  - `_ingested_at`
  - `_source_table`
  - `_source_lakehouse`
- Write Delta format with overwrite and overwriteSchema enabled.
- Partition large transactional tables by year/month derived from:
  - SalesOrderHeader.OrderDate
  - SalesOrderDetail.ModifiedDate
- Smaller master tables remain unpartitioned.
- Preserve binary content in Product.ThumbNailPhoto.
- Record row counts written for each table.

Expected Bronze outputs:
- bronze_address
- bronze_customer
- bronze_customeraddress
- bronze_product
- bronze_productcategory
- bronze_productdescription
- bronze_productmodel
- bronze_productmodelproductdescription
- bronze_salesorderdetail
- bronze_salesorderheader

## Silver

Common Silver standards:
- Rename all columns to snake_case.
- Add `_silver_loaded_at`.
- Trim string columns.
- Deduplicate using business keys and latest modified_date when available.
- Standardize timestamps.
- Remove exact duplicate records.
- OPTIMIZE/VACUUM according to Fabric guidance.

Table-specific deduplication:
- address: key = address_id
- customer: key = customer_id
- customeraddress: key = (customer_id, address_id)
- product: key = product_id
- productcategory: key = product_category_id
- productdescription: key = product_description_id
- productmodel: key = product_model_id
- productmodelproductdescription: key = (product_model_id, product_description_id, culture)
- salesorderheader: key = sales_order_id
- salesorderdetail: key = (sales_order_id, sales_order_detail_id)

Silver business enhancements:
- customer:
  - Derive cleaned_salesperson_raw from sales_person.
  - Derive salesperson_username by extracting username after "\" when present.
- product:
  - Derive is_discontinued from discontinued_date.
  - Derive is_currently_sellable from sell_start_date, sell_end_date, and discontinued_date.
- salesorderheader:
  - Derive order_year, order_month, order_date_key.
  - Derive ship_date_key when ship_date exists.
- salesorderdetail:
  - Derive line_discount_amount = order_qty * unit_price * unit_price_discount.
  - Derive gross_line_amount = order_qty * unit_price.

NOTE:
- User requested ProductModel.Name → modelname. The provided schema contains no Name column on ProductModel. Gold can only expose ProductModelID unless a model-name attribute is added to the source later.

## Gold

Create a business-facing star schema aligned to the requested sales reporting requirements.

Dimensions

1. dim_order_date
- Source: salesorderheader.order_date
- Grain: one row per calendar date.
- Attributes:
  - date
  - year
  - quarter
  - month
  - month_name
  - week
  - day
- Hierarchy:
  - Year → Quarter → Month → Date

2. dim_ship_date
- Source: salesorderheader.ship_date
- Grain: one row per calendar date.
- Attributes similar to dim_order_date.
- Hierarchy:
  - Year → Quarter → Month → Date

3. dim_customer
- User requirement: combine Customer and Address and do not use CustomerAddress as an intermediary.
- Since Customer and Address have no direct join key in the provided schemas, a direct customer-address relationship cannot be reliably established.
- Fallback implementation:
  - Base dimension from Customer.
  - Expose company_name, title, suffix, email_address.
  - If business later confirms a direct relationship, enrich with address attributes.
- Hierarchy:
  - Company Name

4. dim_salesperson
- Source: customer.sales_person
- One row per distinct salesperson_username.
- Extract username from values like domain\username.
- Keep:
  - salesperson_key
  - salesperson_username
  - salesperson_full_value
- Hierarchy:
  - Salesperson Username

5. dim_order
- Source: salesorderheader
- Grain: sales_order_id
- Move non-measure descriptive attributes out of fact:
  - sales_order_id
  - revision_number
  - status
  - ship_method
  - credit_card_approval_code
  - comment
- Hierarchy:
  - Status → Order

6. dim_product
- Source:
  - Product
  - ProductCategory
  - ProductModelProductDescription
  - ProductDescription
  - ProductModel
- Join path:
  - Product.product_model_id = ProductModelProductDescription.product_model_id
  - ProductModelProductDescription.product_description_id = ProductDescription.product_description_id
  - Filter ProductModelProductDescription where culture = 'en'
  - Product.product_category_id = ProductCategory.product_category_id
- Category modeling:
  - Build category and subcategory from ProductCategory parent-child structure.
- Keep relevant attributes:
  - product_id
  - product_number
  - color
  - size
  - weight
  - standard_cost
  - list_price
  - description
  - category
  - subcategory
  - product_model_id
- NOTE:
  - No ProductModel.Name exists in source schema; expose product_model_id instead.
- Hierarchy:
  - Category → Subcategory → Product

Facts

fact_sales_order
- Source:
  - SalesOrderHeader
  - SalesOrderDetail
- Grain:
  - One row per sales order line.
- Joins:
  - sales_order_id
  - customer_id
  - product_id
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
  - gross_sales_amount
  - discount_amount
  - net_sales_amount
  - subtotal
  - tax_amt
  - freight
- Calculations:
  - gross_sales_amount = order_qty * unit_price
  - discount_amount = order_qty * unit_price * unit_price_discount
  - net_sales_amount = gross_sales_amount - discount_amount

Regional reporting note:
- Requested regional performance and maps.
- Available geographic data contains only Address.City and PostalCode.
- No state, province, territory, country, or sales region columns exist.
- Gold should therefore treat City as the highest available geography level and use it for map-based regional analysis unless additional geographic reference data is supplied.

## Test

Store all results in:
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
- Compare Bronze vs Silver counts.
- Pass when variance <= 1%.

2. Gold Dimension PK Not Null
- dim_order_date.date_key
- dim_ship_date.date_key
- dim_customer.customer_key
- dim_salesperson.salesperson_key
- dim_order.order_key
- dim_product.product_key

3. Gold Dimension PK Uniqueness
- Validate uniqueness of every dimension primary key.

4. Referential Integrity
- fact_sales_order.product_key exists in dim_product.
- fact_sales_order.customer_key exists in dim_customer.
- fact_sales_order.salesperson_key exists in dim_salesperson.
- fact_sales_order.order_key exists in dim_order.
- fact_sales_order.order_date_key exists in dim_order_date.
- fact_sales_order.ship_date_key exists in dim_ship_date.

5. Business Rule Validation
- net_sales_amount <= gross_sales_amount.
- discount_amount >= 0.
- unit_price >= 0.
- order_qty > 0.
- average discount percentage between 0 and 100%.

## Semantic model

Mode:
- Direct Lake

Model tables:
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
- Product: Category → Subcategory → Product
- Customer: Company Name
- Order: Status → Order
- SalesPerson: Username

Measures:
- Total Sales = SUM(net_sales_amount)
- Gross Sales = SUM(gross_sales_amount)
- Total Discount Amount = SUM(discount_amount)
- Average Discount Amount = AVERAGE(discount_amount)
- Maximum Discount Amount = MAX(discount_amount)
- Discount % = DIVIDE([Total Discount Amount],[Gross Sales])
- Average Sales = AVERAGE(net_sales_amount)
- Maximum Sales = MAX(net_sales_amount)
- Total Orders = DISTINCTCOUNT(sales_order_id)
- Average Order Value = DIVIDE([Total Sales],[Total Orders])
- Average Order Quantity = AVERAGE(order_qty)
- Maximum Order Quantity = MAX(order_qty)

Geographic settings:
- Set City as geographic category for map visuals.
- Set PostalCode as postal code category where available.

## Report

Page 1 — Sales Executive Overview
- KPI cards:
  - Total Sales
  - Average Sales
  - Maximum Sales
  - Total Orders
- Monthly sales trend line chart.
- Sales by category bar chart.
- Top products by sales.

Page 2 — Geographic Performance
- Bubble map using City.
- Color scale highlighting high and low performing locations.
- Sales by City ranked bar chart.
- Average Sales by City.
- Maximum Sales by City.
- Drill-through to product/category performance by city.

Page 3 — Orders & Discounts
- Order status distribution.
- Sales by ship method.
- Top salespeople by discount percentage.
- Top salespeople by total discount amount.
- Scatter chart:
  - Discount % vs Total Sales.
- Detailed order table.

Page 4 — Product Performance
- Category/Subcategory hierarchy matrix.
- Product sales trend.
- Average sales by product.
- Maximum sales by product.

Page 5 — Data Quality
- Test result summary.
- Pass/fail counts.
- Failed test details.
- Refresh timestamp.
- Row counts by layer.

## Data Agent

Role:
- Sales Performance Analytics Agent for the SalesLT reporting platform.

Domain Instructions:
- Answer questions using only the semantic model.
- Prioritize facts from fact_sales_order and approved dimensions.
- Explain calculations when measures are referenced.
- Surface trends, outliers, top performers, and discount behavior.
- Use City as the available geographic proxy for regional analysis.
- Clearly state when requested geography exceeds available source data.
- Distinguish gross sales, discount amount, and net sales.
- When comparing periods, use Order Date unless the user explicitly requests Ship Date.
- Provide concise summaries first, then supporting detail.

Starter Questions:
- Which cities generate the highest sales?
- Which cities generate the lowest sales?
- What is the average sales amount by month?
- What is the maximum sales value recorded by month?
- Which products drive the most revenue?
- Which product categories are growing fastest?
- Which salespeople offer the largest discounts?
- What is the average discount percentage by salesperson?
- How many orders were placed each month?
- Which customers contribute the most net sales?

Guardrails:
- Do not invent regions, countries, or territories not present in the model.
- Do not infer customer demographics.
- Do not expose PasswordHash or PasswordSalt fields.
- Do not answer with data outside the semantic model.
- If data quality tests fail, mention potential impact on conclusions.
- Identify when a requested attribute is unavailable in the model.
- Always use approved semantic measures where available instead of recreating calculations ad hoc.
