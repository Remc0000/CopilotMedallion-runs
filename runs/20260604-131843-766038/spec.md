# Run Spec 20260604-131743-8f2e67

## Updated specs

### Iteration 1 — 2026-06-04 13:22:16Z — failed layer: silver (run: 20260604-131843-766038)
- **Root cause (1-line summary)**: Silver execution terminated with `System_Cancelled_Session_Statements_Failed`, indicating one Silver-table failure cancelled the Spark session and prevented completion of remaining Silver tables.
- **Cross-table audit**:
  - Address: yes — any unhandled Silver transformation error can cancel the shared Spark session.
  - Customer: yes — derived-column logic increases risk of a table-specific failure stopping the run.
  - CustomerAddress: yes — junction-table processing can fail independently and cancel the session.
  - Product: yes — derived sellable/discontinued logic can fail independently and cancel the session.
  - ProductCategory: yes — any schema or transformation issue can cancel the session.
  - ProductDescription: yes — any schema or transformation issue can cancel the session.
  - ProductModel: yes — schema-shape differences can fail independently and cancel the session.
  - ProductModelProductDescription: yes — junction-table processing can fail independently and cancel the session.
  - SalesOrderDetail: yes — calculated metrics can fail independently and cancel the session.
  - SalesOrderHeader: yes — date-key derivations can fail independently and cancel the session.
- **Fix approach**: GENERALIZE — the failure mode is session-wide and can affect every Silver table, so a single per-table-isolation requirement is safer than table-specific fixes.
- **What was changed**:
  - Tightened the Silver section to require independent read-transform-write execution per Silver table.
  - Required per-table result tracking and deferred failure reporting after all Silver tables are attempted.
  - Required Silver-table schema validation and write completion checks before advancing to the next table.

### Iteration 2 — 2026-06-04 13:29:48Z — failed layer: reporting (run: 20260604-131843-766038)
- **Root cause (1-line summary)**: Reporting stage failed with `System_Cancelled_Session_Statements_Failed`, indicating one reporting artifact build failure cancelled the Spark session and prevented remaining semantic-model/reporting tasks from completing.
- **Cross-table audit**:
  - Address: yes — reporting datasets may ultimately depend on dimensions sourced from Address.
  - Customer: yes — reporting relationships and visuals depend on customer-derived dimensions.
  - CustomerAddress: yes — customer geography reporting depends on this bridge-derived logic.
  - Product: yes — product reporting artifacts depend on product-derived dimensions.
  - ProductCategory: yes — category hierarchies feed reporting artifacts.
  - ProductDescription: yes — product descriptions may be surfaced in semantic models.
  - ProductModel: yes — model attributes participate in reporting dimensions.
  - ProductModelProductDescription: yes — reporting joins may depend on this bridge table.
  - SalesOrderDetail: yes — fact measures originate from detail records.
  - SalesOrderHeader: yes — date dimensions and order attributes originate here.
- **Fix approach**: GENERALIZE — the failure mode is reporting-stage orchestration and can affect any semantic-model, relationship, measure, hierarchy, or report artifact regardless of source table.
- **What was changed**:
  - Added reporting-stage isolation requirements so semantic model, relationships, measures, hierarchies, and report pages are built and validated independently.
  - Required existence validation of all Gold tables before creating reporting artifacts.
  - Required per-artifact status tracking and deferred failure reporting after all reporting artifacts are attempted.

### Iteration 3 — 2026-06-04 13:31:45Z — failed layer: reporting (run: 20260604-131843-766038)
- **Root cause (1-line summary)**: Reporting stage again ended with `System_Cancelled_Session_Statements_Failed`; underlying artifact failure was masked by session cancellation and likely occurred during semantic-model object creation against missing or mismatched model objects.
- **Cross-table audit**:
  - Address: yes — contributes attributes to dim_customer and can indirectly affect reporting fields.
  - Customer: yes — semantic model references customer attributes and relationships.
  - CustomerAddress: yes — customer geography depends on this bridge-derived logic.
  - Product: yes — report visuals and measures reference product dimensions.
  - ProductCategory: yes — product hierarchy depends on category fields.
  - ProductDescription: yes — description attributes may be surfaced in reports.
  - ProductModel: yes — model-related attributes participate in dim_product.
  - ProductModelProductDescription: yes — product-description joins affect report fields.
  - SalesOrderDetail: yes — fact measures originate from this table.
  - SalesOrderHeader: yes — date dimensions and order attributes originate here.
- **Fix approach**: GENERALIZE — the failure is plausibly caused by reporting artifacts referencing objects before they are successfully created, which can affect all reporting assets regardless of source table.
- **What was changed**:
  - Tightened Semantic model requirements to enforce object-existence validation before relationships, hierarchies, and measures are created.
  - Tightened Report requirements to require dependency-aware creation order and skipping dependent artifacts when prerequisite objects are unavailable.
  - Added explicit validation of tables, columns, measures, and relationships before report-page generation.

## Inputs
- Workspace: `0258bc57-3512-47aa-a7fb-6e3930af0f5d`
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
- Target Lakehouse: **f**

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
- For REST/API responses, use `if x is None: raise RuntimeError(...)` before any `.get(...)`.
- Do not use `saveAsTable`; write Delta directly to Lakehouse paths.
- All notebooks must begin with parameter cells.
- Use idempotent overwrite patterns with `overwriteSchema=true`.
- Wrap table processing in try/except that calls `_save_error(layer, e)` and re-raises after logging.
- Use per-table isolation processing patterns.
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
- ALWAYS process source tables in a `for tbl in source_tables:` loop where each iteration is a SELF-CONTAINED unit: read → transform → write → record-result → recover. Wrap the loop body in `try/except` that calls `_save_error(layer, e, table=tbl)` and APPENDS the failure to a results dict, then re-raises only AFTER the loop has attempted all tables (or, if your spec says "fail-fast-first-table", re-raise immediately — but per-table-isolated by default).
- Do NOT build a single multi-CTE Spark SQL statement that joins/transforms many source tables in one shot. Each table's transform is its own DataFrame chain with its own `.write` call.
- Do NOT share intermediate temp views across tables. Temp views from one iteration must not be assumed to exist in the next. If you need cross-table joins (typical for Gold), do them in a SECOND loop AFTER all per-table Silver/Gold writes are complete.

Rule I — Optional audit columns on junction / bridge / view tables.
- In this source, ProductModelProductDescription and CustomerAddress must be treated as junction tables and handled defensively.
- Use composite-key deduplication on junction tables.
- Do not assume any additional audit columns beyond those physically present.

Rule J — Validate column existence BEFORE the expensive transform.
- Assert all required columns exist before joins, filters, windows, aggregations, and derived metrics.

Rule K — Resilience to partial output: Bronze MUST write Delta tables that the next layer can discover.
- Bronze writes Delta tables to `Tables/bronze/<table>`.
- Silver writes Delta tables to `Tables/silver/<table>`.
- Gold writes Delta tables to `Tables/gold/<table>`.
- Emit completion summaries and raise if no discoverable tables are written.

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
  - `_bronze_ingested_at`
  - `_bronze_source_table`
  - `_bronze_run_id`
- Write mode: overwrite with schema overwrite enabled.
- Partition large transactional tables:
  - salesorderheader by year(OrderDate)
  - salesorderdetail by SalesOrderID
- Small master tables remain unpartitioned.
- Retain binary column `ThumbNailPhoto` in Bronze only.

## Silver

Common standards:
- Convert all column names to snake_case.
- Deduplicate using latest `modified_date` where available.
- Add:
  - `_silver_loaded_at`
  - `_silver_run_id`
  - `_source_modified_date`
- Remove obvious technical duplicates.
- OPTIMIZE all Silver Delta tables after write.
- REQUIRED EXECUTION PATTERN: process each Silver table independently in this order: read Bronze table → validate required columns → apply table-specific transformations → write Silver table → verify write success → record status.
- Maintain a per-table results collection containing table name, status, row count, and error details.
- If one Silver table fails, log the failure with `_save_error('silver', e, table=<table_name>)`, continue attempting the remaining Silver tables, and raise a consolidated failure only after all Silver tables have been attempted.
- Do not build a single Silver job, SQL statement, or DataFrame lineage that depends on multiple source tables being transformed together.
- After each Silver write, confirm the target Delta path `Tables/silver/<table_name>` exists and is readable before starting the next table.

Silver table intent and dedup keys:
- address
  - Dedup key: `address_id`
- customer
  - Dedup key: `customer_id`
  - Create cleaned `sales_person_username`:
    - Extract username from values formatted as `domain\username`.
    - Remove trailing numeric suffix when present (example: jillian0 → jillian).
- customeraddress
  - Dedup key: (`customer_id`,`address_id`)
- product
  - Dedup key: `product_id`
  - Derive:
    - `is_discontinued`
    - `is_currently_sellable`
- productcategory
  - Dedup key: `product_category_id`
- productdescription
  - Dedup key: `product_description_id`
- productmodel
  - Dedup key: `product_model_id`
  - NOTE: schema does not contain a Name column. User requested ProductModel Name → modelname. This is not possible from current source. Use ProductModelID as the only available model attribute unless a Name column is added later.
- productmodelproductdescription
  - Dedup key: (`product_model_id`,`product_description_id`,`culture`)
  - Filter culture-specific joins later in Gold.
- salesorderheader
  - Dedup key: `sales_order_id`
  - Derive order_date_key and ship_date_key helper fields.
- salesorderdetail
  - Dedup key: (`sales_order_id`,`sales_order_detail_id`)
  - Derive line_discount_amount and line_extended_amount.

## Gold

Target star schema aligned to requested reporting requirements.

Dimensions

1. dim_order_date
- Source: SalesOrderHeader.OrderDate
- Grain: one row per calendar date.
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
  - Year > Quarter > Month > Date

2. dim_ship_date
- Source: SalesOrderHeader.ShipDate
- Grain: one row per calendar date.
- Attributes similar to dim_order_date.
- Hierarchy:
  - Year > Quarter > Month > Date

3. dim_customer
- Source: Customer + Address
- User requested direct combination without CustomerAddress bridge.
- NOTE: no direct Customer-to-Address relationship exists in the provided schema. CustomerAddress is the only available linkage table.
- Gold implementation should use CustomerAddress internally to perform the join, but expose a final combined customer dimension.
- Keep relevant fields:
  - customer_id
  - company_name
  - title
  - email_address
  - city
  - postal_code
- Geography hierarchy:
  - City > Postal Code

4. dim_salesperson
- Source: Customer.sales_person
- One row per cleaned salesperson username.
- Attributes:
  - salesperson_key
  - salesperson_username
- Hierarchy:
  - Salesperson

5. dim_order
- Source: SalesOrderHeader
- Grain: one row per sales_order_id.
- Attributes:
  - sales_order_id
  - revision_number
  - status
  - ship_method
  - comment
- Keeps non-measure order attributes out of fact.

6. dim_product
- Source:
  - Product
  - ProductCategory
  - ProductModel
  - ProductModelProductDescription
  - ProductDescription
- Join logic:
  - Product → ProductCategory on product_category_id
  - Product → ProductModel on product_model_id
  - ProductModel → ProductModelProductDescription on product_model_id
  - Filter ProductModelProductDescription to culture='en'
  - ProductModelProductDescription → ProductDescription on product_description_id
- Parent-child category flattening:
  - category = parent category
  - subcategory = child category
- Keep relevant fields:
  - product_id
  - product_number
  - color
  - size
  - weight
  - standard_cost
  - list_price
  - category
  - subcategory
  - description
  - modelname
- NOTE: modelname cannot be populated from current schema because ProductModel contains only ProductModelID. Use ProductModelID as placeholder model attribute.
- Product hierarchy:
  - Category > Subcategory > Product

Fact

fact_sales_order
- Source:
  - SalesOrderHeader
  - SalesOrderDetail
- Grain:
  - One row per sales_order_detail_id.
- Join:
  - SalesOrderHeader.sales_order_id = SalesOrderDetail.sales_order_id
- Foreign keys:
  - order_date_key
  - ship_date_key
  - customer_id
  - salesperson_key
  - sales_order_id
  - product_id
- Measures retained:
  - order_qty
  - unit_price
  - unit_price_discount
  - line_sales_amount = order_qty * unit_price
  - discount_amount = order_qty * unit_price * unit_price_discount
  - net_sales_amount = order_qty * unit_price * (1 - unit_price_discount)
  - subtotal
  - tax_amt
  - freight

Reporting focus supported:
- Regional performance by city/postal code.
- Monthly sales trends.
- Average sales.
- Maximum sales.
- Largest discounts by salesperson.
- Order analysis.

## Test

Write all results to `Tables/test/test_results` with:
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
- Bronze vs Silver row counts for every table.
- Expected variance <= 1%.

2. Gold Dimension PK Not Null
- dim_customer.customer_id
- dim_product.product_id
- dim_order.sales_order_id
- dim_order_date.date_key
- dim_ship_date.date_key
- dim_salesperson.salesperson_key

3. Gold Dimension PK Uniqueness
- Validate uniqueness for all dimension primary keys.

4. Referential Integrity
- fact_sales_order.product_id exists in dim_product.
- fact_sales_order.customer_id exists in dim_customer.
- fact_sales_order.sales_order_id exists in dim_order.
- fact_sales_order.order_date_key exists in dim_order_date.
- fact_sales_order.ship_date_key exists in dim_ship_date.
- fact_sales_order.salesperson_key exists in dim_salesperson.

5. Business Rule Sanity Check
- Net sales amount >= 0.
- Discount percentage between 0 and 100%.
- Order quantity > 0.
- Average sales amount <= maximum sales amount.

## Semantic model

Storage mode:
- Direct Lake

Model build order (mandatory):
1. Create semantic model shell.
2. Add and validate all tables.
3. Add and validate all relationships.
4. Add and validate all hierarchies.
5. Add and validate all measures.
6. Publish/save model.
- Do not create relationships, hierarchies, or measures until all referenced tables are confirmed present in the semantic model.
- Before creating any relationship, verify both tables and relationship columns exist.
- Before creating any hierarchy, verify all hierarchy columns exist on the target table.
- Before creating any measure, verify all referenced columns and referenced measures exist.
- Record success/failure for each semantic-model object independently.

Tables:
- fact_sales_order
- dim_customer
- dim_product
- dim_order
- dim_salesperson
- dim_order_date
- dim_ship_date

Relationships:
- fact_sales_order → dim_customer
- fact_sales_order → dim_product
- fact_sales_order → dim_order
- fact_sales_order → dim_salesperson
- fact_sales_order → dim_order_date
- fact_sales_order → dim_ship_date

Hierarchies:
- Order Date: Year > Quarter > Month > Date
- Ship Date: Year > Quarter > Month > Date
- Product: Category > Subcategory > Product
- Customer Geography: City > Postal Code

Measures:
- Total Sales = SUM(net_sales_amount)
- Gross Sales = SUM(line_sales_amount)
- Total Discount Amount = SUM(discount_amount)
- Average Sales = AVERAGE(net_sales_amount)
- Maximum Sale = MAX(net_sales_amount)
- Total Orders = DISTINCTCOUNT(sales_order_id)
- Average Order Value = DIVIDE([Total Sales],[Total Orders])
- Discount % = DIVIDE([Total Discount Amount],[Gross Sales])
- Average Discount % = AVERAGE(unit_price_discount)
- Maximum Discount % = MAX(unit_price_discount)
- Products Sold = SUM(order_qty)

Model features:
- Mark date dimensions as date tables.
- Hide surrogate and technical columns.
- Format currency measures appropriately.

## Report

Reporting execution requirements:
- Treat each reporting artifact as an independent unit: semantic model creation, relationship creation, hierarchy creation, measure creation, and each report page build must run separately.
- Before creating any reporting artifact, validate that all referenced Gold tables exist and are readable:
  - fact_sales_order
  - dim_customer
  - dim_product
  - dim_order
  - dim_salesperson
  - dim_order_date
  - dim_ship_date
- Maintain a reporting results collection containing artifact name, artifact type, status, and error details.
- If one reporting artifact fails, log the failure, continue attempting remaining reporting artifacts, and raise a consolidated reporting failure only after all reporting artifacts have been attempted.
- Do not build the entire semantic model and report in a single monolithic operation without intermediate validation checkpoints.
- After creating each relationship, hierarchy, measure, or report page, validate successful creation before proceeding to the next artifact.
- Dependency enforcement:
  - Do not create report pages until the semantic model is successfully published.
  - Do not place a visual on a page unless every referenced table, column, hierarchy, and measure has been validated in the semantic model.
  - If a required dependency is missing, mark the artifact as skipped with a detailed reason and continue processing remaining independent artifacts.
  - Validate each report page after creation and before adding the next page.

Page 1 — Executive Sales Overview
- KPI cards:
  - Total Sales
  - Average Sales
  - Maximum Sale
  - Total Orders
- Monthly sales trend line chart.
- Sales by product category column chart.
- Discount % KPI.

Page 2 — Regional Performance
- Filled map and bubble map using customer city/postal code.
- Color by Total Sales.
- Tooltip:
  - Total Sales
  - Average Sales
  - Maximum Sales
  - Total Orders
- Top-performing regions chart.
- Lowest-performing regions chart.

Page 3 — Orders & Discounts
- Salesperson ranking table.
- Largest discount by salesperson visual.
- Discount % trend over time.
- Order status distribution.
- Product-level discount analysis.

Page 4 — Product Performance
- Category/subcategory hierarchy drilldown.
- Top products by sales.
- Bottom products by sales.
- Sales versus discount scatter plot.

Page 5 — Data Quality
- Test result summary.
- PASS/FAIL counts.
- Referential integrity status.
- Row count reconciliation status.
- Latest refresh details.

## Data Agent

Role:
- Sales Performance Intelligence Agent for the SalesLT reporting model.
- Answers questions about sales, orders, products, customers, geography, discounts, and salesperson performance using only the semantic model.

Domain hints:
- Regional analysis is based on customer geography from Address data.
- Sales metrics are derived from sales order detail transactions.
- Net sales account for discounts.
- Product hierarchy uses category and subcategory rollups.
- Date analysis can use both order date and ship date perspectives.

Starter questions:
- Which regions generated the highest total sales?
- Which regions generated the lowest total sales?
- What is the monthly sales trend over the last year?
- Which salesperson offered the largest total discounts?
- Which salesperson has the highest average discount percentage?
- What are the top-selling product categories?
- Which products have the highest net sales?
- What is the average sale amount by region?
- What is the maximum sale recorded and when did it occur?
- How many orders were placed each month?

Guardrails:
- Use only approved semantic model tables and measures.
- Prefer explicit measures over raw column aggregation.
- State when a requested attribute is unavailable in the source.
- Do not infer geography beyond available city and postal code fields.
- Do not fabricate product model names because the source schema does not contain them.
- Distinguish gross sales, discount amount, and net sales in all answers.
- When ranking performance, specify the date context used.
- Surface data-quality concerns if relevant tests fail.