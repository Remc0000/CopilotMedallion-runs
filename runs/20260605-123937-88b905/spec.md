# Run Spec 20260605-123906-ec8a95

## Inputs
- Workspace: `db12fd40-6fa7-4998-821c-6ce8e3590ad0`
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
- Target Lakehouse: **c**

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
- Use defensive column references and validate column existence before every join, filter, aggregation, window, and derived-column expression.
- Alias-qualify columns inside join projections and rename immediately after joins.
- Assert required columns exist before every groupBy/agg.
- For REST/API calls use defensive handling: `if x is None: raise` before any `.get()` access.
- Create schemas with `CREATE SCHEMA IF NOT EXISTS` and write only through `saveAsTable('<schema>.<table>')`.
- Never write target layer outputs through raw abfss `.save()` paths.
- Use parameter cells at notebook start for workspace, lakehouse, run_id, source table, and load mode.
- Use idempotent overwrite patterns with `overwriteSchema=true`.
- Use error-loud try/except patterns that call `_save_error(layer, e)` and re-raise.
- Process source tables independently in loops to avoid session-wide failures.
- Every notebook cell must begin with a short comment header block using `# ---` and human-readable comments describing purpose.

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
- Alternative pattern (when the helper genuinely cannot know the dtype) — filter `None`s at the call site BEFORE invoking the Spark function.
- Either approach is acceptable, but never pass Python `None` directly into a Spark function.

Rule H — Per-table isolation; one table's failure must not cancel the Spark session for the rest.

Rule I — Optional audit columns on junction / bridge / view tables.

Rule J — Validate column existence BEFORE the expensive transform.

Rule K — Resilience to partial output: every layer MUST write Delta tables the next layer can discover.

Rule L — Disambiguate shared columns in join projections (avoid AMBIGUOUS_REFERENCE).

## Bronze

Landing strategy:
- Create schema `bronze`.
- Ingest each source table 1:1 with original structure preserved.
- Add metadata columns:
  - `_run_id`
  - `_ingested_at`
  - `_source_table`
  - `_bronze_ts`
- Write as Delta tables using overwrite mode and overwriteSchema=true.

Bronze tables:
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

Partitioning:
- salesorderheader: partition by year derived from OrderDate.
- salesorderdetail: partition by SalesOrderID hash bucket or unpartitioned if volume is small.
- Remaining master-data tables: unpartitioned.

Primary business keys observed:
- Address: AddressID
- Customer: CustomerID
- CustomerAddress: CustomerID + AddressID
- Product: ProductID
- ProductCategory: ProductCategoryID
- ProductDescription: ProductDescriptionID
- ProductModel: ProductModelID
- ProductModelProductDescription: ProductModelID + ProductDescriptionID + Culture
- SalesOrderHeader: SalesOrderID
- SalesOrderDetail: SalesOrderID + SalesOrderDetailID

## Silver

Common transformations:
- Convert all column names to snake_case.
- Preserve business keys.
- Standardize timestamps.
- Remove duplicate rows.
- Add:
  - _silver_ts
  - _source_modified_date
  - _is_current

Deduplication keys:
- silver.address → address_id
- silver.customer → customer_id
- silver.customeraddress → customer_id + address_id
- silver.product → product_id
- silver.productcategory → product_category_id
- silver.productdescription → product_description_id
- silver.productmodel → product_model_id
- silver.productmodelproductdescription → product_model_id + product_description_id + culture
- silver.salesorderheader → sales_order_id
- silver.salesorderdetail → sales_order_id + sales_order_detail_id

Business cleansing:
- Customer:
  - Trim company_name, email_address, sales_person.
  - Exclude password_hash and password_salt from downstream gold models.
- Product:
  - Create is_discontinued flag from discontinued_date.
  - Create is_active_product based on sell dates and discontinuation.
- SalesOrderHeader:
  - Validate order_date <= due_date where both exist.
- SalesOrderDetail:
  - Create line_discount_amount = order_qty * unit_price * unit_price_discount.
  - Create line_gross_amount = order_qty * unit_price.
  - Create line_net_amount = order_qty * unit_price * (1 - unit_price_discount).

Performance:
- OPTIMIZE silver.salesorderheader
- OPTIMIZE silver.salesorderdetail
- OPTIMIZE major gold source dimensions after write

User-request note:
- User requested Product dimension include ProductModel Name. The provided ProductModel schema contains only ProductModelID, rowguid, and ModifiedDate. No Name column exists. Gold Product dimension will therefore include ProductModelID and English description linkage, but model name cannot be populated unless an additional source is supplied.

## Gold

Create schema `gold`.

Dimension: gold.dim_order_date
- Source: SalesOrderHeader.OrderDate
- One row per calendar date.
- Hierarchy:
  - Year
  - Quarter
  - Month
  - Day
- Date intelligence attributes.

Dimension: gold.dim_ship_date
- Source: SalesOrderHeader.ShipDate
- Separate role-playing date dimension.
- Hierarchy:
  - Year
  - Quarter
  - Month
  - Day

Dimension: gold.dim_customer
- Source: Customer + Address.
- User explicitly requested bypassing CustomerAddress.
- Use billing address from SalesOrderHeader.BillToAddressID to associate customer and address.
- Retain relevant fields:
  - customer_id
  - company_name
  - title
  - sales_person reference
  - email_address
  - city
  - postal_code
- Geography fields support map visuals.
- Alternative association is possible through CustomerAddress, but user requested not to use it.

Dimension: gold.dim_salesperson
- Source: Customer.SalesPerson
- Extract username portion.
- Example transformation:
  - adventure-works\jillian0 → jillian
- Deduplicate by cleaned salesperson value.
- Hierarchy:
  - Domain
  - Salesperson

Dimension: gold.dim_order
- Source: SalesOrderHeader
- Move descriptive order attributes out of fact:
  - sales_order_id
  - revision_number
  - status
  - ship_method
  - credit_card_approval_code
  - comment
- Keep fact table focused on measures and keys.

Dimension: gold.dim_product
- Source:
  - Product
  - ProductCategory
  - ProductModelProductDescription
  - ProductDescription
  - ProductModel
- Filter ProductModelProductDescription to Culture='en'.
- Parent-child category flattening:
  - category_id
  - category
  - subcategory_id
  - subcategory
- Include relevant product attributes:
  - product_id
  - product_number
  - color
  - size
  - weight
  - standard_cost
  - list_price
  - description
  - product_model_id
  - active/discontinued indicators
- ProductModel Name unavailable in source schema and therefore cannot be included.

Fact: gold.fact_sales_order
- Grain:
  - One row per SalesOrderDetail line.
- Join:
  - SalesOrderHeader
  - SalesOrderDetail
- Foreign keys:
  - order_date_key
  - ship_date_key
  - customer_key
  - salesperson_key
  - product_key
  - order_key
- Measures stored:
  - order_qty
  - unit_price
  - unit_price_discount
  - gross_sales_amount
  - discount_amount
  - net_sales_amount
  - freight
  - tax_amount
- Derived metrics:
  - discount_pct
  - average_order_line_value
- Regional analysis enabled through customer geography attributes.

Fact/Dimension rationale:
- SalesOrderDetail is transactional and therefore fact-like.
- SalesOrderHeader contributes order-level attributes and dates.
- Customer, Product, SalesPerson, Order, and Date entities are dimensions.
- CustomerAddress is treated as a relationship table and not exposed in Gold.

## Test

Write all results into:
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
- Bronze vs Silver for every table.
- PASS when variance <= 1%.

2. Gold Dimension PK Not Null
- dim_customer.customer_id
- dim_salesperson.salesperson_key
- dim_product.product_id
- dim_order.sales_order_id
- dim_order_date.date_key
- dim_ship_date.date_key

3. Gold Dimension PK Uniqueness
- Validate uniqueness of all dimension business keys.

4. Referential Integrity
- fact_sales_order.product_key exists in dim_product.
- fact_sales_order.customer_key exists in dim_customer.
- fact_sales_order.salesperson_key exists in dim_salesperson.
- fact_sales_order.order_key exists in dim_order.
- fact_sales_order.order_date_key exists in dim_order_date.
- fact_sales_order.ship_date_key exists in dim_ship_date.

5. Business Rule Sanity Check
- discount_pct between 0 and 1.
- net_sales_amount >= 0.
- gross_sales_amount >= net_sales_amount.
- ship_date is null or ship_date >= order_date.
- Regional sales aggregation returns at least one populated geography.

## Semantic model

Mode:
- Direct Lake

Tables:
- dim_order_date
- dim_ship_date
- dim_customer
- dim_salesperson
- dim_product
- dim_order
- fact_sales_order

Relationships:
- fact_sales_order → dim_order_date
- fact_sales_order → dim_ship_date
- fact_sales_order → dim_customer
- fact_sales_order → dim_salesperson
- fact_sales_order → dim_product
- fact_sales_order → dim_order

Hierarchies:

Order Date:
- Year > Quarter > Month > Day

Ship Date:
- Year > Quarter > Month > Day

Customer Geography:
- City > Postal Code

Product:
- Category > Subcategory > Product

SalesPerson:
- Domain > SalesPerson

Measures:
- Total Sales = SUM(net_sales_amount)
- Gross Sales = SUM(gross_sales_amount)
- Total Discount Amount = SUM(discount_amount)
- Discount % = DIVIDE([Total Discount Amount],[Gross Sales])
- Average Sales = AVERAGE(net_sales_amount)
- Maximum Sales = MAX(net_sales_amount)
- Total Orders = DISTINCTCOUNT(sales_order_id)
- Average Order Value = DIVIDE([Total Sales],[Total Orders])
- Maximum Order Value = MAXX(VALUES(dim_order[sales_order_id]), CALCULATE([Total Sales]))
- Average Discount % = AVERAGE(discount_pct)
- Maximum Discount % = MAX(discount_pct)
- Average Quantity = AVERAGE(order_qty)

Regional reporting support:
- Map visuals sourced from customer city and postal code.
- Sales aggregation by geography.

## Report

Page 1: Executive Sales Overview
- KPI cards:
  - Total Sales
  - Gross Sales
  - Average Sales
  - Maximum Sales
  - Total Orders
- Monthly sales trend line chart.
- Sales by category bar chart.
- Sales by salesperson ranking.

Page 2: Regional Performance
- Filled map or bubble map by city.
- Color scale for Total Sales.
- Tooltip:
  - Total Sales
  - Average Sales
  - Maximum Sales
  - Total Orders
- Bottom 10 regions visual.
- Top 10 regions visual.

Page 3: Orders and Discounts
- Monthly order trend.
- Discount % by salesperson.
- Top salespeople by maximum discount percentage.
- Top salespeople by total discount amount.
- Order status breakdown.

Page 4: Product Performance
- Category/subcategory hierarchy drill-down.
- Product sales ranking.
- Average sales by product.
- Maximum sales by product.

Page 5: Data Quality
- Test execution summary.
- Pass/fail counts.
- Recent failed tests.
- Layer row-count comparison.

## Data Agent

Role:
- AI Sales Performance Analyst for the SalesLT reporting environment.
- Answers business questions using only the approved semantic model.
- Focuses on sales performance, regional trends, customer activity, product performance, order behavior, and discount analysis.

Domain hints:
- Sales fact grain is sales-order line level.
- Regional analysis comes from customer geography.
- Product hierarchy uses category and subcategory.
- Salesperson dimension is derived from Customer.SalesPerson.
- Date analysis can use either Order Date or Ship Date perspectives.

Starter questions:
- Which regions generated the highest sales this month?
- Which regions have the lowest sales performance?
- What is the monthly sales trend over the last year?
- What are the average and maximum sales values by region?
- Which salespeople offer the largest discounts?
- What is the average discount percentage by salesperson?
- Which products generate the highest revenue?
- Which product categories are growing fastest?
- How many orders were placed each month?
- What is the average order value by region?

Guardrails:
- Use only measures and dimensions from the semantic model.
- Do not fabricate unavailable geographic levels beyond city and postal code.
- Do not infer product model names because the source does not contain them.
- Clearly distinguish Order Date analysis from Ship Date analysis.
- Prefer governed measures over ad hoc calculations.
- Cite filters and time periods used in every answer.
- If data is unavailable in the model, state the limitation explicitly.
- Do not expose technical columns such as rowguid, password_hash, or password_salt.
