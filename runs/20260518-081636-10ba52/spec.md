# Run Spec 20260518-081403-ebaded

## Inputs
- Workspace: `2242620d-b06e-48f8-a3ee-8e9ae6ffd17f`
- Source Lakehouse: **SalesLT** (`efe41f78-82b7-47ee-9780-2d78372bfdf3`)
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
  - `SalesLT/vGetAllCategories`
  - `SalesLT/vProductAndDescription`
  - `SalesLT/vProductModelCatalogDescription`
- Target Lakehouse: **e2e**

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
- Use defensive column references everywhere; never assume a column exists without checking `df.columns`.
- After every join, immediately project/select alias-prefixed columns into flat names so no downstream step depends on join aliases remaining in scope.
- Before every `groupBy`, `agg`, join, filter, window, or `withColumn`, assert required columns exist and fail fast with a table-specific error.
- For REST/API responses, use defensive handling and `if x is None: raise ...` BEFORE any `.get(...)`.
- Do not use `saveAsTable`; write Delta by path under `Tables/<layer>/...`.
- Use parameter cells for workspace, lakehouse IDs, run ID, and configurable table lists.
- Use idempotent overwrite patterns with `.format('delta').mode('overwrite').option('overwriteSchema','true')`.
- Use error-loud `try/except` blocks that call `_save_error(layer, e)` (and `_save_error(layer, e, table=tbl)` inside per-table loops) and then re-raise per the per-table isolation pattern.
- Every generated notebook cell must start with a short markdown-style Python comment block using a `# ---` divider and 1-3 explanatory comment lines.

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
- Ingest each listed source object 1:1 into `Tables/bronze/<table_name_lower>` using the lowercased last path segment:
  - `address`, `customer`, `customeraddress`, `product`, `productcategory`, `productdescription`, `productmodel`, `productmodelproductdescription`, `salesorderdetail`, `salesorderheader`, `vgetallcategories`, `vproductanddescription`, `vproductmodelcatalogdescription`.
- For each table:
  - Read from the source lakehouse path/object exactly as provided.
  - Preserve source business columns as-is in Bronze; do not apply business joins in this layer.
  - Add standard Bronze metadata columns:
    - `_run_id`
    - `_bronze_ingested_at`
    - `_source_table`
    - `_source_lakehouse_id`
    - `_ingest_date`
  - Normalize only minimally for operability:
    - retain original names in Bronze to mirror source shape
    - optionally flatten unsupported characters if needed, but no business renaming yet
    - preserve binary column `ThumbNailPhoto` in Bronze even if excluded later
- Write mode and partitioning:
  - Use overwrite with schema overwrite for idempotent reruns.
  - Partition Bronze tables by `_ingest_date`; for large transaction tables also keep data skippable by date filters downstream.
- Table-specific Bronze notes:
  - `SalesOrderHeader` and `SalesOrderDetail` are the main transactional sources and should be landed without filtering.
  - `CustomerAddress` is still ingested even though the requested Gold customer dimension does not use it; keep it available for audit/alternate modeling.
  - View objects (`vGetAllCategories`, `vProductAndDescription`, `vProductModelCatalogDescription`) must be treated as regular sources and landed exactly with the columns returned by the view.
- Bronze completion rule:
  - Raise an error if no Bronze tables are written.
  - Emit a JSON summary of written tables and row counts.

## Silver
- Silver applies schema standardization, quality filtering, deduplication, and light enrichment, writing to `Tables/silver/<table>`.
- Common Silver rules for all tables:
  - Rename columns to `snake_case`.
  - Add `_silver_loaded_at`, `_run_id`, and `source_dt`.
  - Use `modified_date` when present for `source_dt`; otherwise typed-null or current timestamp fallback per global rules.
  - Drop obvious technical columns from curated outputs only when not needed downstream; keep `rowguid` available on core entities for traceability.
  - `OPTIMIZE` Silver tables after writes where supported, especially for `sales_order_header`, `sales_order_detail`, `product`, and `customer`.
- Dedup keys by actual schema:
  - `address`: dedupe on `address_id`, keep latest `modified_date`.
  - `customer`: dedupe on `customer_id`, keep latest `modified_date`.
  - `customer_address`: dedupe on composite key `(customer_id, address_id, address_type)`, keep latest `modified_date`.
  - `product`: dedupe on `product_id`, keep latest `modified_date`.
  - `product_category`: dedupe on `product_category_id`, keep latest `modified_date`.
  - `product_description`: dedupe on `product_description_id`, keep latest `modified_date`.
  - `product_model`: dedupe on `product_model_id`, keep latest `modified_date`.
  - `product_model_product_description`: dedupe on `(product_model_id, product_description_id, culture)`, keep latest `modified_date`.
  - `sales_order_detail`: dedupe on `(sales_order_id, sales_order_detail_id)`, keep latest `modified_date`.
  - `sales_order_header`: dedupe on `sales_order_id`, keep latest `modified_date`.
  - `vgetallcategories`: dedupe on `product_category_id`; if duplicates remain, keep deterministic first by category names.
  - `vproductanddescription`: dedupe on `(product_id, culture)` to preserve multilingual granularity.
  - `vproductmodelcatalogdescription`: dedupe on `product_model_id`, keep latest `modified_date`.
- Table-specific Silver shaping:
  - `address`:
    - trim text fields and standardize empty strings to null where appropriate
    - derive a display geography helper such as `city_state_country`
  - `customer`:
    - create `full_name` from title/first/middle/last/suffix
    - lowercase and trim `email_address`
    - keep `sales_person` raw for later salesperson dimension derivation
    - exclude password fields from downstream Gold by default; keep in Silver only if governance requires traceability
  - `customer_address`:
    - retain as a bridge/audit table only; not used in the requested Gold customer design
  - `product`:
    - derive `is_discontinued`
    - derive `is_sellable_currently` using `sell_start_date`, `sell_end_date`, `discontinued_date`
    - keep standard cost, list price, category/model keys for product conformance
  - `product_category`:
    - preserve `parent_product_category_id` for parent-child category rollup
  - `product_model_product_description`:
    - create a Silver helper subset for `culture = 'en'` because Gold product explicitly requests English description preference
  - `sales_order_header`:
    - derive `order_date_key`, `ship_date_key`, and month-start helpers for trend reporting
    - standardize status and online flag into reporting-friendly fields
  - `sales_order_detail`:
    - derive `discount_amount = order_qty * unit_price * unit_price_discount`
    - derive `gross_amount = order_qty * unit_price`
    - retain `line_total` as authoritative net sales amount
- NOTE on requested customer dimension:
  - The user explicitly requested “Combine Customer and Address. Don’t use CustomerAddress in between.”
  - With the actual schema, `Customer` has no direct `AddressID`; the only relational path from customer to address is through `CustomerAddress`.
  - Therefore the requested join is not directly possible from the available columns.
  - Gold will implement a practical fallback: build `dim_customer` primarily from `Customer`, and keep region/geography on the Order/ship-to and bill-to address roles from `SalesOrderHeader` + `Address`, which fully supports regional sales reporting.
  - If the user wants a customer-home-address dimension specifically, they can edit the spec to allow `CustomerAddress`.

## Gold
- Gold implements the user-requested sales star schema, adapted to the actual available keys and columns.

- Proposed Gold tables:
  - `dim_order_date`
  - `dim_ship_date`
  - `dim_customer`
  - `dim_sales_person`
  - `dim_order`
  - `dim_product`
  - `fact_sales_order`

- Fact vs dimension identification from actual sources:
  - Facts:
    - `SalesOrderDetail` is line-grain transactional fact content.
    - `SalesOrderHeader` contributes order-level transaction attributes and additive totals.
  - Dimensions:
    - `Customer`, `Address`, `Product`, `ProductCategory`, `ProductDescription`, `ProductModel`
  - Junction/bridge:
    - `CustomerAddress`
    - `ProductModelProductDescription`
  - Views useful as optional enrichment/validation:
    - `vGetAllCategories`
    - `vProductAndDescription`
    - `vProductModelCatalogDescription`

- `dim_order_date`
  - Build a canonical calendar dimension from distinct `order_date` values in `sales_order_header`, expanded to full contiguous date range between min/max order dates.
  - Primary key: `date_key` as `yyyyMMdd` integer.
  - Include hierarchy-friendly attributes: date, day, month, month_name, quarter, year, year_month, week, day_of_week.
  - Create explicit hierarchy: Year > Quarter > Month > Date.

- `dim_ship_date`
  - Separate role-playing date dimension sourced from `ship_date` in `sales_order_header`.
  - Same structure and hierarchy as `dim_order_date`.
  - Include a default unknown row for orders not yet shipped (`ship_date` null).

- `dim_customer`
  - Requested intent: combine Customer and Address, keeping relevant fields only.
  - Actual-schema fallback:
    - Build the conformed customer dimension from `customer` only because no direct customer-to-address key exists without `customer_address`.
    - Keep relevant fields:
      - `customer_id`
      - formatted customer name fields / `full_name`
      - `company_name`
      - `email_address`
      - `phone`
      - `name_style`
      - cleaned salesperson reference key/value for relationship to `dim_sales_person`
    - Exclude passwords and salts from Gold.
  - If the user later permits `CustomerAddress`, geography can be added to this dimension; for now regional analysis will use order ship/bill geography instead.
  - Suggested hierarchy where applicable:
    - CompanyName > FullName
    - or CustomerID > FullName if company is sparse

- `dim_sales_person`
  - Source: `customer.sales_person`.
  - Split distinct salesperson values into their own dimension.
  - Transform rule:
    - if value is in `<domain>\username` form, keep only username
    - then proper-case for display, e.g. `adventure-works\jillian0` → `Jillian0`
  - Include fields:
    - `sales_person_key` surrogate or cleaned business key
    - `sales_person_raw`
    - `sales_person_username`
    - `sales_person_display_name`
    - optional `sales_person_domain`
  - Relationship path:
    - `dim_customer.sales_person_clean` → `dim_sales_person.sales_person_clean`
    - `fact_sales_order` can carry salesperson key via the joined customer at build time for simpler reporting.
  - Hierarchy where applicable:
    - Domain > SalesPersonDisplayName

- `dim_order`
  - Source primarily from `sales_order_header`.
  - Move suitable descriptive order attributes out of the fact:
    - `sales_order_id` as natural key / dimension PK
    - `sales_order_number`
    - `purchase_order_number`
    - `account_number`
    - `ship_method`
    - `status`
    - `online_order_flag`
    - `credit_card_approval_code` only if governance allows; otherwise exclude as sensitive/low-value
    - `comment`
  - Also add order-level geography helpers from the actual available address keys:
    - ship-to and bill-to address IDs
    - ship city/state/country/postal code from `address`
    - bill city/state/country/postal code from `address`
  - This is the preferred place to support the user’s regional reporting requirement, since `SalesOrderHeader` directly contains `ShipToAddressID` and `BillToAddressID`.
  - Hierarchies:
    - ShipCountry > ShipStateProvince > ShipCity
    - BillCountry > BillStateProvince > BillCity
    - Status groupings if desired

- `dim_product`
  - Build from actual tables per user instruction:
    - base: `product`
    - add category/subcategory from `product_category` parent-child structure
    - add description only from `product_description`
    - add model name only from `product_model.name` as `model_name`
    - use `product_model_product_description` filtered to `culture = 'en'` to determine English description linkage
  - Recommended join path from actual keys:
    - `product.product_category_id` → child `product_category.product_category_id`
    - child category’s `parent_product_category_id` → parent category for category/subcategory split
    - `product.product_model_id` → `product_model.product_model_id`
    - `product.product_model_id` → `product_model_product_description.product_model_id` filtered `culture='en'`
    - then `product_model_product_description.product_description_id` → `product_description.product_description_id`
  - Keep relevant attributes only:
    - `product_id`
    - `product_name`
    - `product_number`
    - `color`, `size`, `weight`
    - `standard_cost`, `list_price`
    - `model_name`
    - `description`
    - `category_name` (parent)
    - `subcategory_name` (child)
    - `sell_start_date`, `sell_end_date`, `discontinued_date`
    - `is_discontinued`, `is_sellable_currently`
  - Exclude binary thumbnail from Gold.
  - Hierarchies:
    - Category > Subcategory > ProductName
    - ModelName > ProductName where useful

- `fact_sales_order`
  - Grain: one row per sales order line (`sales_order_id`, `sales_order_detail_id`).
  - Build by joining `sales_order_detail` to `sales_order_header`, then conformed dimensions.
  - Keep only relevant keys and measures:
    - Foreign keys:
      - `order_date_key`
      - `ship_date_key`
      - `customer_id`
      - `sales_person_key`
      - `sales_order_id` (to `dim_order`)
      - `product_id`
    - Degenerate/retained identifiers:
      - `sales_order_detail_id`
    - Measures:
      - `order_qty`
      - `unit_price`
      - `unit_price_discount`
      - `discount_amount`
      - `gross_amount`
      - `line_total` as net sales
      - header-level amounts only if allocated carefully; otherwise do not repeat `sub_total`, `tax_amt`, `freight`, `total_due` on every line
  - Important modeling rule:
    - Do not duplicate header totals onto every line without allocation because this would overstate totals.
    - If order-level measures are required in reporting, expose them through `dim_order`/a separate order-level fact later; for this build, line-level net sales should be the primary sales measure.
  - Regional performance support:
    - derive geography through `dim_order` ship/bill address attributes
    - map visuals should use ship geography by default because it best represents delivery region
  - Discount analysis support:
    - include salesperson key on the fact by deriving from the customer attached to the order header
    - enables ranking salespeople by average/max discount and discount percentage
  - Alternative the user may edit later:
    - add a second fact at order-header grain for logistics/order-cycle analytics; not required for this first build.

## Test
- Write all test outputs to `Tables/test/test_results` with schema:
  - `run_id, test_name, layer, table_name, status, actual, expected, details, checked_at`
- Each test appends one result row; failures should be recorded before the notebook raises.

- Standard tests:
  1. Row counts per layer
     - Compare Bronze vs Silver row counts for each base table.
     - Expected: Silver row count is approximately Bronze row count within 1% for entity tables after dedup; for transactional tables, document exact variance if duplicates were removed.
     - Minimum required checks:
       - `salesorderheader` vs `sales_order_header`
       - `salesorderdetail` vs `sales_order_detail`
       - `customer` vs `customer`
       - `product` vs `product`

  2. No-null PKs in Gold dims
     - `dim_order_date.date_key`
     - `dim_ship_date.date_key`
     - `dim_customer.customer_id`
     - `dim_sales_person.sales_person_key`
     - `dim_order.sales_order_id`
     - `dim_product.product_id`

  3. Unique PKs in Gold dims
     - assert one row per PK in each dimension above

  4. Referential integrity
     - every `fact_sales_order.product_id` exists in `dim_product`
     - every `fact_sales_order.customer_id` exists in `dim_customer`
     - every `fact_sales_order.sales_order_id` exists in `dim_order`
     - every non-null `fact_sales_order.order_date_key` exists in `dim_order_date`
     - every non-null `fact_sales_order.ship_date_key` exists in `dim_ship_date`
     - every non-null `fact_sales_order.sales_person_key` exists in `dim_sales_person`

  5. Business-rule sanity checks tailored to this data
     - `line_total >= 0`
     - `order_qty > 0`
     - `unit_price >= 0`
     - `unit_price_discount` between 0 and 1 if stored as fractional discount; if actual values indicate amount-based discount instead, flag and document in test details
     - `ship_date >= order_date` for rows with both dates populated
     - `gross_amount >= line_total` for discounted lines
     - at least one mapped ship geography exists (non-null ship country/state/city) to support the requested map visuals

- Additional recommended tests:
  - `dim_product` category/subcategory completeness: non-null category or subcategory for most active products
  - `dim_sales_person` cleaning test: no retained backslash in display name after username extraction
  - customer security test: password fields are absent from Gold outputs
  - date dimension coverage: every header `order_date` and `ship_date` maps to a corresponding date dimension row

## Semantic model
- Storage mode: Direct Lake over Gold tables.
- Star schema:
  - `fact_sales_order` in center
  - dimensions:
    - `dim_order_date`
    - `dim_ship_date`
    - `dim_customer`
    - `dim_sales_person`
    - `dim_order`
    - `dim_product`
- Relationships:
  - `fact_sales_order[order_date_key]` → `dim_order_date[date_key]`
  - `fact_sales_order[ship_date_key]` → `dim_ship_date[date_key]`
  - `fact_sales_order[customer_id]` → `dim_customer[customer_id]`
  - `fact_sales_order[sales_person_key]` → `dim_sales_person[sales_person_key]`
  - `fact_sales_order[sales_order_id]` → `dim_order[sales_order_id]`
  - `fact_sales_order[product_id]` → `dim_product[product_id]`
- Date behavior:
  - Mark `dim_order_date` as the primary date table.
  - Keep `dim_ship_date` as a role-playing date table for shipping analysis.
- Hierarchies to create on all applicable dimensions:
  - `dim_order_date`: Year > Quarter > Month > Date
  - `dim_ship_date`: Year > Quarter > Month > Date
  - `dim_product`: Category > Subcategory > Product Name
  - `dim_order`: Ship Country > Ship StateProvince > Ship City; Bill Country > Bill StateProvince > Bill City
  - `dim_customer`: Company Name > Full Name where populated
  - `dim_sales_person`: Domain > SalesPersonDisplayName if domain exists
- Explicit measures aligned to user intent:
  - `Sales Amount = SUM(fact_sales_order[line_total])`
  - `Gross Sales Amount = SUM(fact_sales_order[gross_amount])`
  - `Discount Amount = SUM(fact_sales_order[discount_amount])`
  - `Discount % = DIVIDE([Discount Amount], [Gross Sales Amount])`
  - `Average Sales = AVERAGE(fact_sales_order[line_total])`
  - `Max Sales = MAX(fact_sales_order[line_total])`
  - `Order Lines = COUNTROWS(fact_sales_order)`
  - `Orders = DISTINCTCOUNT(fact_sales_order[sales_order_id])`
  - `Units Sold = SUM(fact_sales_order[order_qty])`
  - `Average Discount % = AVERAGE(fact_sales_order[unit_price_discount])`
  - `Max Discount % = MAX(fact_sales_order[unit_price_discount])`
  - `Average Order Value = DIVIDE([Sales Amount], [Orders])`
  - `Sales per Order Line = DIVIDE([Sales Amount], [Order Lines])`
- Calculation intent:
  - Use `dim_order` ship geography for regional maps and regional performance visuals.
  - Use `dim_sales_person` with discount measures for identifying salespeople offering the largest discounts.
  - Use `dim_order_date` for monthly sales trends; optionally use `dim_ship_date` for shipment trend comparisons.

## Report
- Build a Power BI report tailored to sales, regional performance, trends, discounts, and data quality.

- Page 1: Executive Sales Overview
  - KPI cards:
    - `Sales Amount`
    - `Average Sales`
    - `Max Sales`
    - `Orders`
    - `Units Sold`
    - `Discount %`
  - Line chart:
    - monthly `Sales Amount` by `dim_order_date[Year-Month]`
  - Clustered column chart:
    - `Sales Amount` by `dim_product[Category]`
  - Donut or bar chart:
    - `Sales Amount` by `dim_order[Ship Country]`
  - Slicers:
    - Order Date hierarchy
    - Ship Date hierarchy
    - Category/Subcategory
    - SalesPerson

- Page 2: Regional Performance
  - Filled map or bubble map:
    - location from ship geography on `dim_order` using Ship Country / StateProvince / City
    - size/color by `Sales Amount`
  - Secondary map or regional bar chart:
    - `Average Sales` by ship region
  - Bar chart:
    - `Max Sales` by ship region
  - Matrix:
    - Ship Country > Ship StateProvince > Ship City with
      - `Sales Amount`
      - `Average Sales`
      - `Max Sales`
      - `Orders`
      - `Discount %`
  - Purpose:
    - quickly identify high- and low-performing regions per user request

- Page 3: Orders, Trends, and Discounts
  - Line chart:
    - monthly `Sales Amount`, with optional comparison of order date vs ship date
  - Table or matrix:
    - top orders by `Sales Amount`, using `dim_order[sales_order_number]`, customer, ship region
  - Bar chart:
    - top salespeople by `Discount Amount`
  - Bar chart:
    - top salespeople by `Discount %`
  - Scatter chart:
    - SalesPerson on details, `Sales Amount` on X, `Discount %` on Y, `Orders` as size
  - Detail table:
    - product, order number, salesperson, discount %, line total

- Page 4: Product Performance
  - Treemap:
    - Category > Subcategory by `Sales Amount`
  - Bar chart:
    - top products by `Sales Amount`
  - Bar chart:
    - top products by `Average Sales`
  - Table:
    - Product, Model, Category, Subcategory, Gross Sales, Discount Amount, Net Sales

- Page 5: Data Quality
  - Card visuals:
    - total tests run
    - tests passed
    - tests failed
  - Table:
    - latest `test_results` with status, test name, table, details, checked_at
  - Stacked column:
    - PASS/FAIL/ERROR counts by layer
  - Optional slicers:
    - layer
    - status
    - run_id

## Data Agent
- Agent type:
  - AISkill grounded on the semantic model built over the Gold Direct Lake dataset.

- Role description:
  - You are a Sales Performance Analytics Agent for the `e2e` Fabric solution.
  - Your job is to help business users understand sales performance by region, product, customer, order, and salesperson using the certified semantic model.
  - You specialize in regional sales analysis, monthly trends, discount behavior, and identifying unusually high or low performance.
  - You should answer with concise business language first, then include the metric logic and filters used when helpful.

- Domain hints for grounding:
  - Sales facts are at sales-order-line grain.
  - Net sales are represented by `line_total`.
  - Gross sales are derived from quantity × unit price.
  - Discount analysis uses both `discount_amount` and `discount %`.
  - Regional analysis should prefer ship geography from the Order dimension.
  - Order trends should use Order Date unless the user explicitly asks about shipping timing.
  - Ship Date is a separate role-playing date dimension.
  - SalesPerson is derived from the `customer.sales_person` field and cleaned from `<domain>\username` format.

- Starter questions:
  - Which regions have the highest and lowest sales?
  - Show monthly sales trends for the last 12 months in the model.
  - What is the average and maximum sales amount by region?
  - Which salespeople are giving the largest discounts?
  - What is the discount percentage by salesperson and by product category?
  - Which products and subcategories drive the most sales?
  - Which customers generate the highest sales amounts?
  - How many orders and order lines were recorded by month?
  - Which regions have high order counts but low average sales?
  - Compare order-date sales trends versus ship-date shipment trends.

- Guardrails:
  - Use only the grounded semantic model; do not invent data, columns, relationships, or external business context.
  - Prefer measures already defined in the semantic model over ad hoc calculations when possible.
  - For “region,” default to ship geography unless the user explicitly asks for billing geography.
  - For “sales,” default to net sales (`line_total` / `Sales Amount`) unless the user explicitly asks for gross sales.
  - When discussing discounts, clarify whether the answer uses `Discount Amount`, `Discount %`, average discount, or maximum discount.
  - If a requested customer-address analysis cannot be answered from the current model, explain that customer geography was not directly available without `CustomerAddress` in the requested design and suggest using ship geography instead.
  - Do not expose sensitive fields such as passwords, salts, or low-level technical metadata.
  - If the question is ambiguous, ask one short clarifying question rather than making a risky assumption.
  - When rankings are requested, state the ranking metric and filter context clearly.
  - If a result may be affected by null ship dates or unmapped geography, mention that limitation briefly.

- Response style instructions:
  - Lead with the direct answer.
  - Follow with 2-4 bullets summarizing the key drivers.
  - Include the measure names used.
  - Mention filter context if time, region, product, or salesperson filters materially change the result.
  - When possible, suggest one follow-up question that deepens the analysis.
