# Run Spec 20260604-120003-8620a7

## Inputs
- Workspace: `1a90328c-54a4-4905-9c94-fa8e59b2ceb1`
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
- Target Lakehouse: **d**

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
- Use defensive column references and validate required columns before use.
- Use alias-prefixed joins only within the join expression; materialize flat columns immediately after joins.
- Assert column existence before every join, filter, withColumn, groupBy, agg, and Window operation.
- Before any REST response `.get()`, use `if x is None: raise RuntimeError(...)`.
- Do not use `saveAsTable`; write Delta directly to Lakehouse paths.
- All notebooks must begin with parameter cells for workspace, source, target, run_id, and layer settings.
- Use idempotent overwrite patterns with `.mode("overwrite").option("overwriteSchema","true")`.
- Wrap table processing in error-loud try/except blocks that call `_save_error(layer, e)` and re-raise appropriately.
- Process source tables independently per-table where possible.
- Every notebook cell must begin with a short comment block explaining purpose and intent.

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

ALSO REQUIRE for every generated notebook: EACH code cell must start with a short markdown comment block (Python `# ---` divider + 1-3 lines of `# ` comments) describing what the cell is doing and why — never emit a cell with no leading comment. Example:
```
# ---
# Read raw bronze table and apply schema enforcement.
# Drops rows where any required key is null.
df = spark.read.table(...)
```
Keep the comments human-readable, not the code repeated in prose. The goal: someone scrolling through the notebook in Fabric can understand each block at a glance.

## Bronze

- Land each source table unchanged into `Tables/bronze/<table_name_lower>`.
- Preserve source schema and datatypes.
- Add metadata columns:
  - `_run_id`
  - `_bronze_ingested_at`
  - `_source_table`
  - `_source_lakehouse`
- Write format: Delta.
- Write mode: overwrite with schema overwrite enabled.
- Partitioning:
  - `salesorderheader`: partition by year derived from `OrderDate`.
  - `salesorderdetail`: partition by year derived from `ModifiedDate`.
  - Remaining tables: no partitioning due to expected small dimension size.
- Record row counts for all landed tables.
- Preserve binary column `ThumbNailPhoto` without transformation.

## Silver

General transformations:
- Rename all columns to snake_case.
- Standardize timestamps.
- Add:
  - `_silver_ts`
  - `_run_id`
  - `_record_source`
- Remove exact duplicate records.
- Retain business and audit fields needed for Gold.

Per-table deduplication keys:
- address: `address_id`
- customer: `customer_id`
- customer_address: `(customer_id, address_id)`
- product: `product_id`
- product_category: `product_category_id`
- product_description: `product_description_id`
- product_model: `product_model_id`
- product_model_product_description: `(product_model_id, product_description_id, culture)`
- sales_order_header: `sales_order_id`
- sales_order_detail: `sales_order_detail_id`

Silver business enrichment:
- Address:
  - Standardize city and postal_code.
- Customer:
  - Normalize email address casing.
  - Create `salesperson_username` derived from `sales_person`.
  - If value contains `\`, keep username portion only.
  - Remove trailing numeric suffixes for reporting display (example: `jillian0` → `jillian`).
- Product:
  - Create `is_discontinued`.
  - Create `is_currently_sellable`.
- Product Category:
  - Materialize parent-child relationship helper columns.
- Sales Order Header:
  - Derive order_year, order_month, ship_year, ship_month.
- Sales Order Detail:
  - Derive line_sales_amount = order_qty * unit_price.
  - Derive line_discount_amount = order_qty * unit_price * unit_price_discount.
  - Derive line_net_sales_amount = line_sales_amount - line_discount_amount.

OPTIMIZE and VACUUM eligible Silver Delta tables after successful write.

Note:
- User requested ProductModel Name → modelname. The actual ProductModel table contains only `ProductModelID`, `rowguid`, and `ModifiedDate`; no Name column exists. Gold will expose ProductModelID and related description data unless a model name source is later provided.

## Gold

Star schema requested by user.

Dimensions:

1. DimOrderDate
- Source: `sales_order_header.order_date`
- Grain: one row per calendar date.
- Keys:
  - order_date_key
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

2. DimShipDate
- Source: `sales_order_header.ship_date`
- Grain: one row per calendar date.
- Keys:
  - ship_date_key
- Hierarchy:
  - Year → Quarter → Month → Date

3. DimCustomer
- User requirement: combine Customer and Address and do not use CustomerAddress as an intermediate business entity.
- Implementation:
  - Join customer directly to address using sales order relationships:
    - sales_order_header.customer_id
    - sales_order_header.ship_to_address_id
    - sales_order_header.bill_to_address_id
  - Build a customer-centric dimension using the address most recently associated with customer orders.
- Attributes:
  - customer_id
  - company_name
  - title
  - suffix
  - email_address
  - city
  - postal_code
- Hierarchy:
  - City → Customer

Note:
- Customer and Address are not directly related in the source schema. CustomerAddress exists but user requested not to use it. Therefore customer-to-address association will be inferred from SalesOrderHeader order activity.

4. DimSalesPerson
- Source: customer.sales_person
- One row per normalized salesperson username.
- Attributes:
  - salesperson_key
  - salesperson_username
  - salesperson_original
- Hierarchy:
  - SalesPerson

5. DimOrder
- Source: SalesOrderHeader
- Grain: one row per order.
- Attributes:
  - sales_order_id
  - revision_number
  - status
  - ship_method
  - credit_card_approval_code
  - comment
- Hierarchy:
  - Status → Order

6. DimProduct
- Sources:
  - Product
  - ProductCategory
  - ProductModel
  - ProductModelProductDescription
  - ProductDescription
- Required logic:
  - Filter ProductModelProductDescription to `culture='en'`.
  - Join Product → ProductCategory.
  - Join Product → ProductModel.
  - Join ProductModelProductDescription → ProductDescription.
  - Materialize category and subcategory from parent-child ProductCategory structure.
- Attributes:
  - product_id
  - product_number
  - color
  - size
  - weight
  - standard_cost
  - list_price
  - category_id
  - category
  - subcategory
  - description
  - product_model_id
  - thumbnail_photo_file_name
  - is_discontinued
  - is_currently_sellable
- Hierarchies:
  - Category → Subcategory → Product
  - Color → Product

Fact:

FactSalesOrder
- Grain:
  - One row per sales order detail line.
- Sources:
  - SalesOrderHeader
  - SalesOrderDetail
- Joins:
  - sales_order_id
- Foreign keys:
  - order_date_key
  - ship_date_key
  - customer_id
  - salesperson_key
  - order_key
  - product_id
- Measures stored:
  - order_qty
  - unit_price
  - unit_price_discount
  - gross_sales_amount
  - discount_amount
  - net_sales_amount
  - freight
  - tax_amt
- Calculations:
  - gross_sales_amount = order_qty * unit_price
  - discount_amount = order_qty * unit_price * unit_price_discount
  - net_sales_amount = gross_sales_amount - discount_amount

Regional reporting approach:
- No explicit region field exists in source data.
- Use City and PostalCode from Address as geographic reporting dimensions.
- Map visuals will be based on City geography.

## Test

All tests append one result row into:
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

1. Row Count Consistency
- Compare Bronze vs Silver row counts.
- Pass when variance ≤ 1%.

2. Gold Dimension PK Not Null
- DimCustomer.customer_id
- DimProduct.product_id
- DimOrder.order_key
- DimOrderDate.order_date_key
- DimShipDate.ship_date_key
- DimSalesPerson.salesperson_key

3. Gold Dimension PK Uniqueness
- Validate uniqueness of all dimension primary keys.

4. Referential Integrity
- FactSalesOrder.product_id exists in DimProduct.
- FactSalesOrder.customer_id exists in DimCustomer.
- FactSalesOrder.order_key exists in DimOrder.
- FactSalesOrder.order_date_key exists in DimOrderDate.
- FactSalesOrder.ship_date_key exists in DimShipDate.
- FactSalesOrder.salesperson_key exists in DimSalesPerson.

5. Business Rule Sanity Check
- net_sales_amount >= 0
- gross_sales_amount >= discount_amount
- unit_price >= 0
- order_qty > 0

## Semantic model

Mode:
- Direct Lake

Tables:
- FactSalesOrder
- DimCustomer
- DimProduct
- DimOrder
- DimOrderDate
- DimShipDate
- DimSalesPerson

Relationships:
- FactSalesOrder → DimCustomer
- FactSalesOrder → DimProduct
- FactSalesOrder → DimOrder
- FactSalesOrder → DimOrderDate
- FactSalesOrder → DimShipDate
- FactSalesOrder → DimSalesPerson

Hierarchies:
- Order Date: Year → Quarter → Month → Date
- Ship Date: Year → Quarter → Month → Date
- Product: Category → Subcategory → Product
- Customer: City → Customer
- SalesPerson: SalesPerson

Measures:
- Total Sales = SUM(FactSalesOrder[net_sales_amount])
- Gross Sales = SUM(FactSalesOrder[gross_sales_amount])
- Total Discount Amount = SUM(FactSalesOrder[discount_amount])
- Average Sales = AVERAGE(FactSalesOrder[net_sales_amount])
- Maximum Sales = MAX(FactSalesOrder[net_sales_amount])
- Total Orders = DISTINCTCOUNT(FactSalesOrder[sales_order_id])
- Total Quantity = SUM(FactSalesOrder[order_qty])
- Average Order Value = DIVIDE([Total Sales],[Total Orders])
- Discount % = DIVIDE([Total Discount Amount],[Gross Sales])
- Average Discount % = AVERAGE(FactSalesOrder[unit_price_discount])
- Maximum Discount % = MAX(FactSalesOrder[unit_price_discount])
- Average Freight = AVERAGE(FactSalesOrder[freight])
- Maximum Freight = MAX(FactSalesOrder[freight])

## Report

Page 1 — Executive Sales Overview
- KPI cards:
  - Total Sales
  - Average Sales
  - Maximum Sales
  - Total Orders
  - Discount %
- Monthly sales trend line chart using Order Date hierarchy.
- Sales by product category clustered column chart.
- Top products by net sales.

Page 2 — Geographic Performance
- Bubble map using City.
- Color scale by Total Sales.
- Size by Total Sales.
- Top performing cities table.
- Lowest performing cities table.
- Average Sales by City.
- Maximum Sales by City.

Page 3 — Orders & Discounts
- Salesperson ranking by Discount %.
- Salesperson ranking by Total Discount Amount.
- Order status breakdown.
- Order quantity distribution.
- Discount trend over time.

Page 4 — Product Performance
- Category/Subcategory/Product drill-down visual.
- Gross Sales vs Net Sales comparison.
- Average Sales by Product.
- Maximum Sales by Product.

Page 5 — Data Quality
- Test pass/fail summary.
- Failed test details table.
- Row counts by layer.
- Referential integrity status indicators.

## Data Agent

Role:
- Sales Performance Intelligence Agent for the SalesLT sales model.
- Answer business questions using the Direct Lake semantic model.
- Focus on sales performance, customer activity, product performance, discount behavior, order trends, and geographic analysis.

Domain hints:
- Geographic analysis is based on City and PostalCode from customer-associated addresses.
- Regional performance refers to available geographic attributes in the source system.
- Sales measures should prioritize net sales amount unless explicitly requested otherwise.
- Distinguish gross sales, discounts, and net sales in all explanations.
- Use date hierarchies when summarizing trends.

Starter questions:
- Which cities generated the highest net sales?
- Which cities generated the lowest net sales?
- What are the monthly sales trends?
- Which salespeople offered the largest discounts?
- What is the average sales amount by city?
- What is the maximum sales amount by city?
- Which product categories generate the most revenue?
- What is the average discount percentage by salesperson?
- Which products have the highest net sales?
- How many orders were placed each month?

Guardrails:
- Only answer using data available in the semantic model.
- Do not invent regions, territories, or countries that do not exist in the source data.
- Clearly state when requested information is unavailable.
- Use measures from the semantic model whenever possible.
- Prefer aggregated business insights over row-level disclosure.
- Explain whether figures are based on gross sales, discount amount, or net sales.
- Surface potential data-quality concerns when relevant.
- When comparing periods, use the date dimensions and active model relationships.
