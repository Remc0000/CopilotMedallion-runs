# Run Spec 20260604-082256-3144db

## Inputs
- Workspace: `fa4681d4-fabe-41bc-b3c8-d6daa3601f10`
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
- Use defensive column references and validate schema before transformations.
- Apply alias-prefixed joins and immediately flatten required columns after joins.
- Assert required columns exist before every join, filter, groupBy, agg, Window, or withColumn operation.
- Use defensive REST handling with `if x is None: raise RuntimeError(...)` before any `.get()` access.
- Do not use `saveAsTable`; write Delta files directly to lakehouse paths.
- All notebooks must begin with parameter cells for workspace, source, target, layer, and run_id.
- Use idempotent overwrite patterns with `overwriteSchema=true`.
- Wrap table processing in error-loud try/except blocks that call `_save_error(layer, e)` and re-raise.
- Process source tables independently per-table wherever possible.
- Every code cell must start with a short markdown-style Python comment block describing purpose and intent.

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
  
- Pick the `dtype` to match the surrounding expression (`'timestamp'` for date/time coalesces, `'string'` for text, `'double'` for numeric, etc.) so Spark can resolve the result type without ambiguity.
- Alternative pattern (when the helper genuinely cannot know the dtype) — filter `None`s at the call site BEFORE invoking the Spark function:
  
  candidates = [c for c in (_maybe(df, 'modified_date'), _maybe(df, 'order_date')) if c is not None]
  df = df.withColumn('source_dt', F.to_date(F.coalesce(*candidates, F.current_timestamp())))
  
- Either approach is acceptable, but never pass Python `None` directly into a Spark function.
- Applies to ALL optional-column lookups across Bronze, Silver, Gold — including audit-timestamp coalesces, optional-key joins, fallback string formatting, etc. This is a layer-agnostic rule.

Rule H — Per-table isolation; one table's failure must not cancel the Spark session for the rest.
- Spark cancels the entire session when one statement crashes. If your notebook builds a single chained plan that touches every source table (one big SELECT, one big DataFrame, one big SQL script), any one table's failure kills ALL tables.
- ALWAYS process source tables in a `for tbl in source_tables:` loop where each iteration is a SELF-CONTAINED unit: read → transform → write → record-result → recover. Wrap the loop body in `try/except` that calls `_save_error(layer, e, table=tbl)` and APPENDS the failure to a results dict, then re-raises only AFTER the loop has attempted all tables.
- Do NOT build a single multi-CTE Spark SQL statement that joins/transforms many source tables in one shot.
- Do NOT share intermediate temp views across tables.
- If cross-table joins are required, perform them after all prerequisite layer outputs exist.

Rule I — Optional audit columns on junction / bridge / view tables.
- Use column existence checks before assuming audit columns exist.
- Deduplicate junction tables using composite foreign keys.
- Project only columns actually present.

Rule J — Validate column existence BEFORE the expensive transform.
- Assert required columns exist before joins, filters, aggregations, windows, and derived columns.

Rule K — Resilience to partial output: Bronze MUST write Delta tables that the next layer can discover.
- Write discoverable Delta outputs under Tables/bronze, Tables/silver, and Tables/gold.
- Emit completion summaries.
- Raise an error if no tables were written.

## Bronze

Land each source table unchanged into `Tables/bronze/<table_name>` with:
- Source metadata columns:
  - `_run_id`
  - `_ingested_at`
  - `_source_table`
  - `_bronze_ts`
- Write mode: overwrite
- Format: Delta
- Schema evolution enabled via overwriteSchema=true

Bronze tables:
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

Partitioning:
- SalesOrderHeader: partition by OrderDate year/month helper columns.
- SalesOrderDetail: partition by SalesOrderID hash bucket helper.
- Remaining tables: unpartitioned due to expected small dimension size.

## Silver

Common standards:
- Rename all columns to snake_case.
- Preserve business keys.
- Add:
  - `_silver_ts`
  - `_source_modified_date`
  - `_is_current`
- Remove duplicate records.
- Retain rowguid only for lineage, not business modeling.
- OPTIMIZE after write.

Deduplication keys:
- address: AddressID
- customer: CustomerID
- customeraddress: CustomerID + AddressID
- product: ProductID
- productcategory: ProductCategoryID
- productdescription: ProductDescriptionID
- productmodel: ProductModelID
- productmodelproductdescription: ProductModelID + ProductDescriptionID + Culture
- salesorderheader: SalesOrderID
- salesorderdetail: SalesOrderID + SalesOrderDetailID

Silver business enhancements:
- Customer:
  - Normalize EmailAddress.
  - Trim text fields.
- Product:
  - Create is_discontinued flag from DiscontinuedDate.
  - Create is_active_product flag from SellStartDate/SellEndDate.
- SalesOrderHeader:
  - Create order_year, order_month, order_date_key.
  - Create ship_year, ship_month, ship_date_key.
- SalesOrderDetail:
  - Create line_discount_amount = OrderQty * UnitPrice * UnitPriceDiscount.
  - Create gross_line_amount = OrderQty * UnitPrice.
  - Create net_line_amount = OrderQty * UnitPrice * (1 - UnitPriceDiscount).

Important modeling note:
- User requested Customer dimension by combining Customer and Address without using CustomerAddress.
- The available schema contains no direct Customer-to-Address relationship. CustomerAddress is the only relationship table available.
- To satisfy reporting requirements while preserving data correctness, Silver should retain CustomerAddress and Gold should use it internally to obtain the current customer-address association. The junction table will not be exposed as a Gold dimension.

Important modeling note:
- User requested ProductModel Name. The provided ProductModel schema contains only ProductModelID, rowguid, and ModifiedDate.
- No Name column exists in source data.
- Gold Product dimension will expose ProductModelID and English product description attributes instead unless additional source columns become available.

## Gold

Target star schema optimized for Direct Lake.

Dimensions

### dim_order_date
Source:
- SalesOrderHeader.OrderDate

Attributes:
- Date
- Year
- Quarter
- Month
- Month Name
- Week
- Day

Hierarchy:
- Year → Quarter → Month → Date

Key:
- order_date_key

### dim_ship_date
Source:
- SalesOrderHeader.ShipDate

Attributes:
- Date
- Year
- Quarter
- Month
- Month Name
- Week
- Day

Hierarchy:
- Year → Quarter → Month → Date

Key:
- ship_date_key

### dim_customer
Source:
- Customer
- Address
- CustomerAddress (internal join mechanism only)

Relevant attributes:
- CustomerID
- CompanyName
- Title
- EmailAddress
- City
- PostalCode

Hierarchy:
- City → PostalCode

Regional reporting:
- City serves as the highest available geographic attribute because no State/Country/Region columns exist.

### dim_salesperson
Source:
- Customer.SalesPerson

Transformation:
- Split values on "\".
- Keep username component only.
- Remove trailing numeric suffix when pattern resembles user examples (e.g. jillian0 → jillian).

Attributes:
- salesperson_key
- salesperson_username
- salesperson_source_value

Hierarchy:
- Single level dimension.

### dim_order
Source:
- SalesOrderHeader

Attributes:
- SalesOrderID
- RevisionNumber
- Status
- ShipMethod
- CreditCardApprovalCode
- Comment

Hierarchy:
- Status → SalesOrderID

### dim_product
Source:
- Product
- ProductCategory
- ProductModelProductDescription
- ProductDescription
- ProductModel

Join rules:
- Product → ProductCategory via ProductCategoryID
- Product → ProductModelProductDescription via ProductModelID
- ProductModelProductDescription filtered to Culture='en'
- ProductModelProductDescription → ProductDescription via ProductDescriptionID

Relevant attributes:
- ProductID
- ProductNumber
- Description
- Color
- Size
- Weight
- StandardCost
- ListPrice
- CategoryID
- ParentCategoryID
- is_active_product
- is_discontinued

Category handling:
- Expose category and subcategory from ProductCategory parent-child structure.
- If parent category exists:
  - ParentProductCategoryID = category
  - ProductCategoryID = subcategory

Hierarchy:
- Category → Subcategory → Product

Note:
- ProductModel Name cannot be included because the column is not present in source data.

### fact_sales_order

Source:
- SalesOrderHeader
- SalesOrderDetail

Join:
- SalesOrderHeader.SalesOrderID = SalesOrderDetail.SalesOrderID

Foreign keys:
- order_date_key
- ship_date_key
- customer_id
- salesperson_key
- order_id
- product_id

Measures stored as additive columns:
- order_qty
- unit_price
- unit_price_discount
- gross_sales_amount
- discount_amount
- net_sales_amount
- tax_amount
- freight_amount

Business grain:
- One row per SalesOrderDetail line.

Reporting focus:
- Sales performance
- Monthly sales trends
- Discount analysis
- Product performance
- Geographic performance by city

## Test

All test results append to:
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

Standard tests:

1. Row Count Reconciliation
- Compare Bronze vs Silver row counts.
- Expected variance <= 1%.

2. Gold Dimension PK Not Null
- dim_customer.customer_id
- dim_product.product_id
- dim_order.order_id
- dim_salesperson.salesperson_key
- dim_order_date.order_date_key
- dim_ship_date.ship_date_key

3. Gold Dimension PK Uniqueness
- Validate unique keys in all dimensions.

4. Referential Integrity
- fact_sales_order.customer_id exists in dim_customer
- fact_sales_order.product_id exists in dim_product
- fact_sales_order.order_id exists in dim_order
- fact_sales_order.order_date_key exists in dim_order_date
- fact_sales_order.ship_date_key exists in dim_ship_date
- fact_sales_order.salesperson_key exists in dim_salesperson

5. Business Rule Sanity Checks
- Net sales amount >= 0
- Gross sales amount >= net sales amount
- Discount percentage between 0 and 100
- Order quantity > 0
- Average sales <= maximum sales for every reporting period

## Semantic model

Storage mode:
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
- fact_sales_order.order_date_key → dim_order_date
- fact_sales_order.ship_date_key → dim_ship_date
- fact_sales_order.customer_id → dim_customer
- fact_sales_order.salesperson_key → dim_salesperson
- fact_sales_order.order_id → dim_order
- fact_sales_order.product_id → dim_product

Hierarchies:
- Order Date: Year → Quarter → Month → Date
- Ship Date: Year → Quarter → Month → Date
- Product: Category → Subcategory → Product
- Customer: City → PostalCode

Measures:

- Total Sales =
  SUM(net_sales_amount)

- Gross Sales =
  SUM(gross_sales_amount)

- Total Discount Amount =
  SUM(discount_amount)

- Discount Percentage =
  DIVIDE([Total Discount Amount],[Gross Sales])

- Total Orders =
  DISTINCTCOUNT(order_id)

- Total Quantity =
  SUM(order_qty)

- Average Sales =
  AVERAGE(fact_sales_order[net_sales_amount])

- Maximum Sales =
  MAX(fact_sales_order[net_sales_amount])

- Average Order Value =
  DIVIDE([Total Sales],[Total Orders])

- Maximum Order Value =
  MAXX(VALUES(dim_order[SalesOrderID]),CALCULATE([Total Sales]))

- Average Discount Percentage =
  AVERAGE(fact_sales_order[unit_price_discount])

- Maximum Discount Percentage =
  MAX(fact_sales_order[unit_price_discount])

- Active Products =
  COUNTROWS(FILTER(dim_product, dim_product[is_active_product]=TRUE()))

Key report scenarios supported:
- Regional performance
- Monthly sales trends
- Largest discounts by salesperson
- Product category performance
- Order analysis

## Report

### Page 1 — Executive Sales Overview

Visuals:
- KPI: Total Sales
- KPI: Average Sales
- KPI: Maximum Sales
- KPI: Total Orders
- Line chart: Monthly Sales Trend
- Clustered column chart: Sales by Product Category
- Slicer: Date

### Page 2 — Regional Performance

Visuals:
- Map visual using Customer City
- Bubble size: Total Sales
- Color saturation: Average Sales
- Tooltip: Maximum Sales, Orders, Discount Percentage
- Bar chart: Top and Bottom Cities by Sales
- Matrix: City performance summary

Note:
- True geographic regions are unavailable in source data; City-based geography is the highest supported granularity.

### Page 3 — Salesperson Discount Analysis

Visuals:
- Ranked bar chart: Salesperson by Discount Percentage
- Ranked bar chart: Salesperson by Total Discount Amount
- Scatter plot:
  - X = Discount Percentage
  - Y = Total Sales
  - Size = Orders
- Table: Largest discounted orders

### Page 4 — Orders and Product Performance

Visuals:
- Orders by Status
- Average Order Value by Month
- Product Category/Subcategory hierarchy drilldown
- Top Products by Sales
- Top Products by Discount Amount

### Page 5 — Data Quality

Visuals:
- PASS/FAIL summary
- Test execution history
- Failed tests table
- Layer row count comparison
- Referential integrity status

## Data Agent

Role:
- Sales Analytics Copilot for SalesAnalytics.
- Expert in sales performance, customer geography, product performance, order analysis, discount behavior, and data quality monitoring.
- Grounded exclusively on the semantic model and approved business measures.

Domain hints:
- Sales is measured using net sales amount.
- Geographic analysis is city-based.
- Product hierarchy is Category → Subcategory → Product.
- Salesperson usernames are normalized from source login values.
- Discount analysis should use Discount Percentage and Total Discount Amount measures.

Starter questions:
- Which cities generated the highest total sales?
- Which cities generated the lowest total sales?
- What are the monthly sales trends over time?
- What is the average and maximum sales value by month?
- Which salespeople offered the largest discounts?
- Which salespeople generated the highest sales after discounts?
- Which product categories contribute most revenue?
- What is the average discount percentage by salesperson?
- Which orders have the highest net sales value?
- How many active products are currently being sold?

Guardrails:
- Use only semantic model data.
- Do not fabricate geographic regions not present in the model.
- Treat City as the highest available geographic level.
- Prefer business measures over raw column aggregation.
- Explain calculations using semantic model measures.
- Surface uncertainty when requested data is unavailable.
- Never infer missing ProductModel names.
- Do not expose technical lineage or audit fields unless specifically requested.
- When comparing performance, provide average, maximum, and total metrics where applicable.
