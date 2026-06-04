# Run Spec 20260604-114516-1d5df3

## Inputs
- Workspace: `1f02de75-3d95-4694-b090-c7cecaae69bf`
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
- Use defensive column references and validate existence before use.
- Use alias-prefixed joins only within the join/select scope and materialize flat column names immediately after joins.
- Assert required columns before every join, filter, groupBy, agg, Window, and withColumn operation.
- For REST/API calls: use `if x is None: raise RuntimeError(...)` before any `.get()` access.
- Do not use `saveAsTable`; write Delta directly to managed lakehouse paths.
- Every notebook must begin with parameter cells for workspace, source, target, run_id, layer, and paths.
- Use idempotent overwrite patterns with `overwriteSchema=true`.
- Wrap per-table processing in error-loud try/except blocks that call `_save_error(layer, e)` and re-raise after recording failures.
- Process source tables independently; avoid session-wide dependency chains.
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
  def _maybe(df, name, dtype='timestamp'):
      return F.col(name) if name in df.columns else F.lit(None).cast(dtype)
- Pick the dtype to match the surrounding expression.
- Never pass Python None directly into Spark functions.

Rule H — Per-table isolation; one table's failure must not cancel the Spark session for the rest.
- Process source tables independently in loops.
- Record failures and continue processing remaining tables.
- Perform cross-table Gold joins only after Silver outputs exist.

Rule I — Optional audit columns on junction / bridge / view tables.
- Do not assume ModifiedDate exists on all tables.
- Use composite-key deduplication for junction tables.
- Project only actual source columns.

Rule J — Validate column existence BEFORE the expensive transform.
- Assert required columns before joins, filters, aggregations, and derivations.

Rule K — Resilience to partial output: Bronze MUST write Delta tables that the next layer can discover.
- Write discoverable Delta tables under Tables/bronze, Tables/silver, and Tables/gold.
- Emit summary output describing written tables.
- Raise an error if a layer produces zero output tables.

## Bronze

Land each source table unchanged into `Tables/bronze/<table_name_lower>`.

For every table:
- Preserve original source schema and datatypes.
- Add metadata columns:
  - `_run_id`
  - `_ingested_at`
  - `_source_table`
  - `_bronze_ts`
- Write as Delta using overwrite mode with overwriteSchema=true.
- Partition by ingestion date derived from `_ingested_at`.

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

Primary keys identified:
- Address: AddressID
- Customer: CustomerID
- CustomerAddress: (CustomerID, AddressID)
- Product: ProductID
- ProductCategory: ProductCategoryID
- ProductDescription: ProductDescriptionID
- ProductModel: ProductModelID
- ProductModelProductDescription: (ProductModelID, ProductDescriptionID, Culture)
- SalesOrderHeader: SalesOrderID
- SalesOrderDetail: SalesOrderDetailID

## Silver

Standard processing:
- Convert all column names to snake_case.
- Preserve business keys.
- Add:
  - `_silver_ts`
  - `_record_source`
  - `_run_id`
- Remove exact duplicates.
- Apply table-specific deduplication.
- OPTIMIZE and V-ORDER after write.

Deduplication strategy:
- silver.address → AddressID
- silver.customer → CustomerID
- silver.customeraddress → (CustomerID, AddressID)
- silver.product → ProductID
- silver.productcategory → ProductCategoryID
- silver.productdescription → ProductDescriptionID
- silver.productmodel → ProductModelID
- silver.productmodelproductdescription → (ProductModelID, ProductDescriptionID, Culture)
- silver.salesorderheader → SalesOrderID
- silver.salesorderdetail → SalesOrderDetailID

Business cleansing:
- Parse customer.sales_person into normalized username.
- If value follows `<domain>\username`, keep only username.
- Remove password_hash and password_salt from downstream Gold outputs.
- Create product status flags:
  - is_discontinued
  - is_active_for_sale
- Create order lifecycle flags:
  - is_shipped
  - is_open_order

Important modeling note:
- User requested ProductModel.Name as modelname, but ProductModel contains only ProductModelID, rowguid, and ModifiedDate. No Name column exists in the provided schema. Gold Product dimension will therefore include ProductModelID but cannot expose modelname unless the source schema is expanded.

## Gold

Target star schema for sales analytics.

Dimension: dim_order_date
- Source: SalesOrderHeader.OrderDate
- One row per calendar date.
- Hierarchy:
  - Year
  - Quarter
  - Month
  - Day
- Include fiscal-friendly attributes where derivable.

Dimension: dim_ship_date
- Source: SalesOrderHeader.ShipDate
- One row per ship date.
- Hierarchy:
  - Year
  - Quarter
  - Month
  - Day

Dimension: dim_customer
- User requirement: combine Customer and Address and do not use CustomerAddress.
- Limitation: Customer and Address have no direct join key.
- CustomerAddress is the only available relationship table connecting CustomerID and AddressID.
- Gold implementation should therefore use CustomerAddress internally to establish the relationship, while exposing a final combined Customer dimension.
- Keep relevant fields:
  - customer_id
  - company_name
  - sales_person_username
  - email_address
  - title
  - city
  - postal_code
- Exclude:
  - password_hash
  - password_salt
  - rowguid fields

Hierarchy:
- City
- Customer

Dimension: dim_salesperson
- Source: Customer.SalesPerson
- Extract username from domain-qualified values.
- One row per salesperson.
- Attributes:
  - salesperson_key
  - salesperson_username

Hierarchy:
- Salesperson

Dimension: dim_order
- Source: SalesOrderHeader
- Move non-measure attributes out of fact:
  - sales_order_id
  - status
  - revision_number
  - ship_method
  - credit_card_approval_code
  - comment
- Keep fact table lean.

Hierarchy:
- Status
- Order

Dimension: dim_product
- Source combination:
  - Product
  - ProductCategory
  - ProductModelProductDescription
  - ProductDescription
  - ProductModel
- Filter ProductModelProductDescription to Culture='en'.

Product category handling:
- Resolve parent-child ProductCategory structure.
- Flatten into:
  - category_id
  - category
  - subcategory_id
  - subcategory

Relevant attributes:
- product_id
- product_number
- color
- size
- weight
- standard_cost
- list_price
- description
- product_model_id
- sell_start_date
- sell_end_date
- discontinued_date

Hierarchy:
- Category
- Subcategory
- Product

Modeling limitation:
- ProductCategory table contains IDs only and no category name column.
- ProductModel table contains no Name column.
- Category names, subcategory names, and modelname cannot be populated from the supplied schema. Use available IDs unless richer source data is provided.

Fact: fact_sales_order
- Grain:
  - One row per SalesOrderDetail line.
- Source:
  - SalesOrderHeader joined to SalesOrderDetail.

Foreign keys:
- order_date_key
- ship_date_key
- customer_key
- salesperson_key
- product_key
- order_key

Measures stored in fact:
- order_qty
- unit_price
- unit_price_discount
- extended_amount = order_qty * unit_price
- discount_amount = order_qty * unit_price * unit_price_discount
- net_sales_amount = extended_amount - discount_amount
- subtotal
- tax_amt
- freight

Regional analysis:
- Region represented by customer city/postal code from address data.
- Map visuals will use city and postal code geography.

## Test

All tests append results into:
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

Test 1: Row count reconciliation
- Bronze vs Silver counts for every table.
- PASS when variance <= 1%.

Test 2: Gold dimension PK null check
- Verify no null PKs in:
  - dim_customer
  - dim_product
  - dim_salesperson
  - dim_order
  - dim_order_date
  - dim_ship_date

Test 3: Gold dimension PK uniqueness
- Verify unique business keys in every dimension.

Test 4: Referential integrity
- fact_sales_order.customer_key exists in dim_customer
- fact_sales_order.product_key exists in dim_product
- fact_sales_order.salesperson_key exists in dim_salesperson
- fact_sales_order.order_key exists in dim_order
- fact_sales_order.order_date_key exists in dim_order_date
- fact_sales_order.ship_date_key exists in dim_ship_date

Test 5: Business-rule validation
- net_sales_amount >= 0
- discount percentage between 0 and 100
- shipped orders must have ship_date populated

## Semantic model

Mode:
- Direct Lake

Tables:
- fact_sales_order
- dim_order_date
- dim_ship_date
- dim_customer
- dim_salesperson
- dim_product
- dim_order

Relationships:
- fact_sales_order → dim_order_date
- fact_sales_order → dim_ship_date
- fact_sales_order → dim_customer
- fact_sales_order → dim_salesperson
- fact_sales_order → dim_product
- fact_sales_order → dim_order

Hierarchies:
- Order Date: Year > Quarter > Month > Day
- Ship Date: Year > Quarter > Month > Day
- Product: Category > Subcategory > Product
- Customer: City > Customer
- SalesPerson: SalesPerson
- Order: Status > Order

Measures:
- Total Sales = SUM(net_sales_amount)
- Gross Sales = SUM(extended_amount)
- Total Discount Amount = SUM(discount_amount)
- Average Sales = AVERAGE(net_sales_amount)
- Maximum Sale = MAX(net_sales_amount)
- Total Orders = DISTINCTCOUNT(order_key)
- Total Quantity = SUM(order_qty)
- Average Discount % = AVERAGE(unit_price_discount) * 100
- Maximum Discount % = MAX(unit_price_discount) * 100
- Average Order Value = DIVIDE([Total Sales],[Total Orders])
- Sales per Customer = DIVIDE([Total Sales], DISTINCTCOUNT(customer_key))

## Report

Page 1: Executive Sales Overview
- KPI cards:
  - Total Sales
  - Average Sales
  - Maximum Sale
  - Total Orders
- Monthly sales trend line chart.
- Sales by salesperson bar chart.
- Sales by product category chart.

Page 2: Regional Performance
- Filled map or Azure Maps visual using city and postal code.
- Bubble size: Total Sales.
- Color scale: Average Sales.
- Tooltip:
  - Maximum Sale
  - Total Orders
  - Average Discount %
- Top and bottom performing regions visual.
- Regional monthly trend chart.

Page 3: Orders & Discounts
- Order status distribution.
- Discount leaderboard by salesperson.
- Maximum Discount % by salesperson.
- Order quantity distribution.
- Detail table of largest discounted orders.

Page 4: Product Performance
- Sales by product hierarchy.
- Top products by sales.
- Top products by discount amount.
- Product profitability view using sales versus standard cost.

Page 5: Data Quality
- Test status summary.
- Failed test details.
- Row count reconciliation matrix.
- Referential integrity results.

## Data Agent

Role:
- AI Sales Performance Analyst for the SalesLT sales reporting solution.

Domain instructions:
- Answer questions only using the semantic model.
- Prioritize business outcomes, sales performance, customer trends, regional performance, product performance, and discount analysis.
- Always use measures rather than raw aggregation when available.
- Explain drivers of high and low regional performance.
- Compare average, maximum, and total metrics when relevant.
- Highlight data quality issues when test results indicate failures.
- If category names or product model names are unavailable, explain that the source data only contains IDs.

Starter questions:
- Which regions generate the highest total sales?
- Which regions have the lowest sales performance?
- What are the monthly sales trends?
- Which salesperson provides the largest discounts?
- Which salesperson drives the highest net sales?
- What is the average sale value by region?
- What is the maximum sale recorded by region?
- Which products contribute the most revenue?
- Which products receive the largest discounts?
- How do shipped and unshipped orders compare?
- What is the average discount percentage by salesperson?
- Which customers generate the highest revenue?

Guardrails:
- Do not fabricate missing category names or model names.
- Do not answer using data outside the semantic model.
- Distinguish between averages, maxima, and totals.
- State filters and time periods used in every analytical answer.
- Surface uncertainty when source attributes are unavailable.
- Respect row-level aggregation and avoid exposing sensitive source fields such as password hashes or salts.
