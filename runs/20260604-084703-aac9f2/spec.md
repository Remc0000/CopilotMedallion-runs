# Run Spec 20260604-084600-e0d98a

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
- Target Lakehouse: **SalesLTAnalytics**

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
- Use defensive column references and assert required columns before every join, filter, groupBy, agg, Window, and withColumn operation.
- Use alias-prefixed joins during join construction, then materialize flat column names immediately after joins.
- Assert all groupBy and aggregation columns exist before execution.
- For REST/API responses, use explicit validation (`if x is None: raise RuntimeError(...)`) before any `.get()` access.
- Never use `saveAsTable`; write Delta directly to Lakehouse paths.
- All notebooks must begin with parameter cells for workspace, source, target, run_id, and layer configuration.
- Use idempotent overwrite patterns with `.mode("overwrite").option("overwriteSchema","true")`.
- Wrap per-table processing in error-loud try/except blocks that call `_save_error(layer, e)` (or `_save_error(layer, e, table=tbl)` inside loops) and re-raise appropriately.
- Enforce per-table isolation and resilient execution.
- Every code cell must start with a short comment block describing purpose and intent.
- Validate outputs after each layer and fail if no discoverable Delta tables were produced.

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
- Junction tables: dedupe on the COMPOSITE FK key.
- View tables: project ONLY the columns actually returned by the view. Do not assume any standard naming.

Rule J — Validate column existence BEFORE the expensive transform.
- For every join, withColumn, groupBy, agg, or filter that names a specific column, ASSERT the column exists in `df.columns` BEFORE the line that uses it.
- Especially important AFTER a select(), drop(), or rename() — re-validate before the next consumer of those columns.

Rule K — Resilience to partial output: Bronze MUST write Delta tables that the next layer can discover.
- Bronze, Silver, Gold, and Test outputs must be discoverable Delta tables under their respective layer folders.
- Emit final layer summaries and raise if no tables were written.

## Bronze

Land each source table unchanged into `Tables/bronze/<table_name>` with source fidelity preserved.

Source-to-Bronze tables:
- bronze.address
- bronze.customer
- bronze.customer_address
- bronze.product
- bronze.product_category
- bronze.product_description
- bronze.product_model
- bronze.product_model_product_description
- bronze.sales_order_detail
- bronze.sales_order_header

Standard Bronze metadata:
- ingest_run_id
- ingest_timestamp_utc
- source_table
- source_lakehouse
- bronze_loaded_at

Write pattern:
- Delta format
- Overwrite mode with schema overwrite
- Partition large transactional tables by year extracted from ModifiedDate when available:
  - sales_order_header
  - sales_order_detail
- Small dimension/master tables remain unpartitioned

Data handling:
- No business transformations
- Preserve binary ThumbnailPhoto in Product
- Preserve rowguid and ModifiedDate columns
- Record row counts per table

## Silver

Standard processing for all tables:
- Convert column names to snake_case
- Add silver_loaded_at
- Add source_modified_date from modified_date where present
- Remove exact duplicate rows
- Validate primary-key uniqueness after dedup
- Optimize and vacuum according to Fabric best practices

Table-specific deduplication keys:
- address: address_id
- customer: customer_id
- customer_address: (customer_id, address_id)
- product: product_id
- product_category: product_category_id
- product_description: product_description_id
- product_model: product_model_id
- product_model_product_description: (product_model_id, product_description_id, culture)
- sales_order_header: sales_order_id
- sales_order_detail: (sales_order_id, sales_order_detail_id)

Silver business enrichment:
- customer:
  - Create sales_person_username derived from sales_person.
  - If value contains "\" then retain username portion only.
  - Remove numeric suffixes where present (example: jillian0 → jillian).
  - Preserve original sales_person field for traceability.
- product:
  - Create is_discontinued flag from discontinued_date.
  - Create is_currently_sellable based on sell_start_date, sell_end_date, and discontinued_date.
- sales_order_detail:
  - Calculate discount_amount = unit_price * unit_price_discount * order_qty.
- sales_order_header:
  - Create order_date_key and ship_date_key helper values for downstream dimensions.

Note:
- User requested ProductModel.Name as modelname, but ProductModel schema contains only ProductModelID, rowguid, and ModifiedDate. No Name column exists. Gold will therefore use ProductModelID as the available model reference unless the source schema is expanded.

## Gold

Target star schema per user requirements.

Fact table:
- fact_sales_order
  - Grain: one sales order line item (SalesOrderDetail joined to SalesOrderHeader)
  - Join keys:
    - order_id
    - product_id
    - customer_key
    - sales_person_key
    - order_date_key
    - ship_date_key
  - Measures retained:
    - order_qty
    - unit_price
    - unit_price_discount
    - gross_sales_amount = order_qty * unit_price
    - discount_amount
    - net_sales_amount = gross_sales_amount - discount_amount
    - subtotal_allocated (optional proportional allocation)
    - tax_amount
    - freight_amount

Dimensions:

- dim_order_date
  - Source: SalesOrderHeader.OrderDate
  - Attributes:
    - date
    - day
    - month
    - month_name
    - quarter
    - year
  - Hierarchy:
    - Year → Quarter → Month → Date

- dim_ship_date
  - Source: SalesOrderHeader.ShipDate
  - Same attributes as order date
  - Hierarchy:
    - Year → Quarter → Month → Date

- dim_customer
  - Source: Customer joined directly to Address per user requirement
  - Note: Customer and Address have no direct join key in the provided schema. The available relationship exists through CustomerAddress.
  - Requested design cannot be implemented exactly from current schema.
  - Practical implementation:
    - Use CustomerAddress bridge to associate customers to addresses.
    - Retain only relevant fields:
      - customer_id
      - company_name
      - email_address
      - city
      - postal_code
  - Geography hierarchy:
    - City → Postal Code

- dim_sales_person
  - Source: Customer.sales_person
  - One row per normalized username
  - Attributes:
    - sales_person_key
    - sales_person_username
    - sales_person_original
  - Hierarchy:
    - Sales Person

- dim_order
  - Source: SalesOrderHeader
  - Move descriptive order attributes out of fact:
    - sales_order_id
    - revision_number
    - status
    - ship_method
    - credit_card_approval_code
    - comment
  - Hierarchy:
    - Status → Ship Method

- dim_product
  - Source integration:
    - Product
    - ProductCategory
    - ProductModel
    - ProductModelProductDescription (Culture='en')
    - ProductDescription (Description only)
  - Category modeling:
    - Resolve parent-child ProductCategory into:
      - category_id
      - category_parent_id
      - category_level
      - category
      - subcategory
  - Relevant attributes:
    - product_id
    - product_number
    - description
    - color
    - size
    - weight
    - standard_cost
    - list_price
    - model_id
    - category
    - subcategory
    - is_discontinued
    - is_currently_sellable
  - Hierarchies:
    - Category → Subcategory → Product
    - Category → Subcategory → Product Number

Modeling notes:
- ProductDescription join path:
  Product → ProductModel → ProductModelProductDescription (Culture='en') → ProductDescription.
- ProductModel name requested by user is unavailable in source data.
- Regional reporting will be based on City and PostalCode from customer-address geography because no state/province/country fields exist.

## Test

Write all results to:
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

1. Row count reconciliation
- Bronze vs Silver row counts
- Tolerance: ±1%
- Execute for every table

2. Gold dimension PK not null
- dim_customer.customer_id
- dim_product.product_id
- dim_order.sales_order_id
- dim_sales_person.sales_person_key
- dim_order_date.date_key
- dim_ship_date.date_key

3. Gold dimension PK uniqueness
- Validate uniqueness of each dimension primary key

4. Referential integrity
- fact_sales_order.product_id exists in dim_product
- fact_sales_order.customer_key exists in dim_customer
- fact_sales_order.sales_person_key exists in dim_sales_person
- fact_sales_order.order_date_key exists in dim_order_date
- fact_sales_order.ship_date_key exists in dim_ship_date
- fact_sales_order.sales_order_id exists in dim_order

5. Business-rule sanity checks
- net_sales_amount >= 0
- gross_sales_amount >= net_sales_amount
- unit_price >= 0
- order_qty > 0
- average discount percentage between 0 and 100
- maximum discount percentage between 0 and 100

Each test appends one result row and never overwrites prior test history.

## Semantic model

Mode:
- Direct Lake

Tables:
- fact_sales_order
- dim_customer
- dim_product
- dim_order
- dim_sales_person
- dim_order_date
- dim_ship_date

Relationships:
- fact_sales_order → dim_product
- fact_sales_order → dim_customer
- fact_sales_order → dim_sales_person
- fact_sales_order → dim_order
- fact_sales_order → dim_order_date
- fact_sales_order → dim_ship_date

Hierarchies:

- Order Date:
  - Year → Quarter → Month → Date

- Ship Date:
  - Year → Quarter → Month → Date

- Product:
  - Category → Subcategory → Product

- Customer Geography:
  - City → Postal Code

Measures:

- Total Sales =
  SUM(net_sales_amount)

- Gross Sales =
  SUM(gross_sales_amount)

- Total Discount Amount =
  SUM(discount_amount)

- Average Sales =
  AVERAGE(net_sales_amount)

- Maximum Sale =
  MAX(net_sales_amount)

- Total Orders =
  DISTINCTCOUNT(sales_order_id)

- Total Quantity =
  SUM(order_qty)

- Average Discount Percentage =
  DIVIDE(SUM(discount_amount), SUM(gross_sales_amount), 0)

- Maximum Discount Percentage =
  MAXX(
    fact_sales_order,
    DIVIDE(discount_amount, gross_sales_amount, 0)
  )

- Average Order Value =
  DIVIDE([Total Sales], [Total Orders], 0)

- Distinct Customers =
  DISTINCTCOUNT(customer_id)

- Active Products =
  DISTINCTCOUNT(product_id)

Regional analytics:
- Geography sourced from City and PostalCode.
- Use Customer Geography hierarchy for map visuals.

## Report

Page 1 — Executive Sales Overview
- KPI cards:
  - Total Sales
  - Gross Sales
  - Average Sales
  - Maximum Sale
  - Total Orders
- Monthly sales trend line chart
- Sales by product category column chart
- Top customers by sales bar chart
- Salesperson performance table

Page 2 — Regional Performance
- Bubble map using City location
- Filled map where supported by geography resolution
- Sales by City ranking chart
- Average Sales by City
- Maximum Sale by City
- Regional order count matrix
- High-performing vs low-performing region decomposition tree

Page 3 — Discounts & Salesperson Analysis
- Top salespeople by Total Discount Amount
- Top salespeople by Average Discount Percentage
- Scatter plot:
  - Discount Percentage
  - Sales Amount
- Product category discount analysis
- Maximum discount transactions table

Page 4 — Orders & Fulfillment
- Orders by status
- Orders by ship method
- Order volume trend
- Ship date vs order date analysis
- Freight and tax contribution visuals

Page 5 — Data Quality
- Test pass/fail summary
- Failed tests table
- Referential integrity status
- Row count reconciliation status
- Data refresh metadata

## Data Agent

Role:
- Sales Performance Intelligence Agent for SalesLTAnalytics.
- Specialized in sales performance, regional analysis, discount behavior, customer trends, product performance, order activity, and fulfillment insights.

Grounding:
- Use only the Direct Lake semantic model.
- Prefer measures over raw column aggregation.
- Use defined hierarchies when drilling through dimensions.

Domain hints:
- Regional analysis is based on customer City and PostalCode.
- Sales metrics come from fact_sales_order.
- Product rollups use Category and Subcategory hierarchies.
- Salesperson analysis uses normalized usernames from dim_sales_person.

Starter questions:
- Which regions generated the highest and lowest sales?
- What is the monthly sales trend over time?
- Which salespeople offered the largest discounts?
- What is the average discount percentage by salesperson?
- Which product categories generate the most revenue?
- Which customers generate the highest sales?
- What are the highest-value individual sales transactions?
- How do average and maximum sales vary by region?
- Which ship methods are associated with the most orders?
- Which products have declining sales trends?

Guardrails:
- Never answer using data outside the semantic model.
- Clearly distinguish sales amount, gross sales, and discount metrics.
- Use measures rather than ad hoc calculations whenever equivalent measures exist.
- When geography is requested, explain that available regional granularity is City and PostalCode.
- If requested attributes are unavailable in source data (for example ProductModel name), state that the field is not present in the model.
- Surface filter context and date range assumptions when reporting results.
- Prioritize dimensional hierarchies for drill-down recommendations.
- Do not expose technical columns such as rowguid, password_hash, password_salt, or ingestion metadata.
