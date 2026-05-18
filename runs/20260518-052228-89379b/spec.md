# Run Spec 20260518-051803-a256b5

## Updated specs

### Iteration 1 — 2026-05-18 05:24:57Z — failed layer: bronze (run: 20260518-052228-89379b)
- **Root cause (1-line summary)**: Bronze completed without producing discoverable Delta tables under the `bronze` schema path/name convention, so Silver could not find any upstream inputs.
- **Cross-table audit**: `SalesLT/Address`: yes — Bronze discovery failure would block it like all other sources because this is a table-registration/path-convention issue, not a source-specific shape issue; `SalesLT/Customer`: yes — same reason; `SalesLT/CustomerAddress`: yes — same reason; `SalesLT/Product`: yes — same reason; `SalesLT/ProductCategory`: yes — same reason; `SalesLT/ProductDescription`: yes — same reason; `SalesLT/ProductModel`: yes — same reason; `SalesLT/ProductModelProductDescription`: yes — same reason; `SalesLT/SalesOrderDetail`: yes — same reason; `SalesLT/SalesOrderHeader`: yes — same reason; `SalesLT/vGetAllCategories`: yes — same reason; `SalesLT/vProductAndDescription`: yes — same reason; `SalesLT/vProductModelCatalogDescription`: yes — same reason.
- **Fix approach**: GENERALIZE — the root cause is systemic across every Bronze source object because the next layer depends on a uniform bronze schema/table discovery pattern.
- **What was changed**:
  - Tightened **Generic guidance** with mandatory Bronze discoverability checks and required catalog/path parity for every written table.
  - Tightened **Bronze** to require creation of the `bronze` schema, exact table names, exact physical layout under `Tables/bronze/<table_name>`, and post-write verification that all expected Bronze tables are discoverable before the layer is considered successful.
  - Added an explicit Bronze failure rule: if any expected Bronze table is missing after write/registration, raise and fail Bronze rather than allowing Silver to start.

## Inputs
- Workspace: `2127a578-6ec2-447e-b8a9-94868408b064`
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
- Target Lakehouse: **e2egp54**

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
- Use defensive column references everywhere; before selecting, renaming, deriving, joining, filtering, aggregating, or windowing, assert required columns exist on the DataFrame in that exact step.
- After every join, immediately project columns with explicit aliases/prefixes so downstream logic never depends on ambiguous duplicate names.
- Before every `groupBy` / `agg`, assert the grouping and aggregated columns exist; fail fast with a clear error naming the missing column(s) and DataFrame variable.
- For REST/API responses, use defensive handling: `if x is None: raise ...` BEFORE any `.get()` access.
- Do not use `saveAsTable`; write using path/table patterns supported for Lakehouse notebooks and keep writes idempotent.
- Every notebook must start with parameter cells for workspace, source lakehouse, target lakehouse, schema, table names, and run id so the same notebook can be rerun safely.
- Use idempotent overwrite/upsert patterns by layer; Bronze can overwrite full landing per table, Silver should overwrite curated tables deterministically, Gold should overwrite dimensional/fact outputs deterministically.
- Use error-loud `try/except` blocks around major read/transform/write units; in `except`, call `_save_error(layer, e)` and then `raise`.
- Never silently skip missing required inputs; if an expected table or column is absent, raise with a specific message.
- Avoid hard-coding schema assumptions beyond the actual source columns discovered at runtime.
- Treat the three `v*` objects as source tables/views and validate whether they add business value versus duplicating derivations already achievable from base tables.
- Bronze/Silver discoverability rule: a layer is only considered successful if its outputs are both physically written and discoverable by Spark catalog/listing in the target Lakehouse using the expected schema-qualified names. For Bronze specifically, Silver must be able to discover the outputs as `bronze.<table_name>` objects; if discovery fails for any expected table, raise in Bronze and stop the run.
- Catalog/path parity rule: when writing managed Lakehouse tables, ensure the registered table name and the physical folder layout align with the schema/table convention expected by downstream layers. Do not write orphaned Delta paths without a discoverable table object, and do not register a table under a different schema/name than the medallion spec.
- Post-write verification rule: after each table write, immediately verify the table exists in the target schema and is queryable; after the full Bronze loop, verify the complete expected Bronze set is present before allowing downstream generation.

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

ALSO REQUIRE for every generated notebook: EACH code cell must start with a short markdown comment block (Python `# ---` divider + 1-3 lines of `# ` comments) describing what the cell is doing and why — never emit a cell with no leading comment. Example:
```
# ---
# Read raw bronze table and apply schema enforcement.
# Drops rows where any required key is null.
df = spark.read.table(...)
```
Keep the comments human-readable, not the code repeated in prose. The goal: someone scrolling through the notebook in Fabric can understand each block at a glance.

## Bronze
Land each selected source object into the `bronze` schema as a raw-but-queryable Delta table with minimal transformation plus technical metadata.

### Bronze landing pattern
- One bronze table per source object, preserving source granularity and original business meaning.
- Recommended bronze names:
  - `bronze.address`
  - `bronze.customer`
  - `bronze.customer_address`
  - `bronze.product`
  - `bronze.product_category`
  - `bronze.product_description`
  - `bronze.product_model`
  - `bronze.product_model_product_description`
  - `bronze.sales_order_detail`
  - `bronze.sales_order_header`
  - `bronze.v_get_all_categories`
  - `bronze.v_product_and_description`
  - `bronze.v_product_model_catalog_description`
- The Bronze notebook MUST create/ensure the `bronze` schema exists in the target Lakehouse before writing any table.
- For each source object above, the write outcome MUST be a discoverable schema-qualified table with the exact name listed above; do not write only to an unmanaged/orphan path and assume downstream discovery will infer it.
- Physical organization MUST align with the Lakehouse table convention expected by downstream discovery: each Bronze output must land under the target Lakehouse managed tables area for schema `bronze` and table `<table_name>` so that listing/querying `bronze.<table_name>` succeeds immediately after the write.
- After writing each Bronze table, immediately verify all of the following before moving on:
  - the table is discoverable as `bronze.<table_name>`
  - it is queryable by Spark
  - it contains at least the source columns plus the required Bronze technical columns
- After the full Bronze load, perform a completeness assertion that the full expected Bronze set exists and is discoverable:
  - `bronze.address`
  - `bronze.customer`
  - `bronze.customer_address`
  - `bronze.product`
  - `bronze.product_category`
  - `bronze.product_description`
  - `bronze.product_model`
  - `bronze.product_model_product_description`
  - `bronze.sales_order_detail`
  - `bronze.sales_order_header`
  - `bronze.v_get_all_categories`
  - `bronze.v_product_and_description`
  - `bronze.v_product_model_catalog_description`
- If any expected Bronze table is missing from discovery after the write loop, fail Bronze with an explicit missing-table error and do NOT allow the run to proceed to Silver.
- Add standard technical columns to every bronze table:
  - `ingested_at_utc`
  - `run_id`
  - `source_workspace_id`
  - `source_lakehouse_id`
  - `source_object_name`
  - `source_row_hash` for change detection / dedup support
- Keep source column names intact in Bronze; do not snake_case until Silver.
- Write mode: deterministic full overwrite for this run, because these are source snapshots and no CDC columns are provided.
- Partitioning:
  - Do not over-partition the small master tables.
  - For `sales_order_header` and `sales_order_detail`, prefer no partition initially unless runtime row counts justify date partitioning; if partitioning is needed, derive a date partition in Silver from `OrderDate` via header join rather than forcing it in Bronze.
- Binary handling:
  - Keep `ThumbNailPhoto` in `bronze.product`, but plan to drop or isolate it in Silver/Gold if not analytically useful.
- View-like source objects:
  - Land all three `v*` objects in Bronze for traceability, but treat them as optional accelerators in later layers because their content overlaps with base tables.

### Bronze table-specific notes
- `Address`: natural key appears to be `AddressID`; preserve all address text fields as-is.
- `Customer`: likely primary key `CustomerID`; retain sensitive columns in Bronze (`PasswordHash`, `PasswordSalt`) for lineage only, but mark for removal in Silver.
- `CustomerAddress`: likely junction table with composite key (`CustomerID`, `AddressID`, `AddressType`).
- `Product`: likely primary key `ProductID`; retain pricing and lifecycle dates.
- `ProductCategory`: likely self-referencing hierarchy via `ParentProductCategoryID`.
- `ProductDescription`: likely primary key `ProductDescriptionID`.
- `ProductModel`: likely primary key `ProductModelID`.
- `ProductModelProductDescription`: likely composite key (`ProductModelID`, `ProductDescriptionID`, `Culture`).
- `SalesOrderHeader`: header grain appears to be one row per `SalesOrderID`.
- `SalesOrderDetail`: line grain appears to be one row per (`SalesOrderID`, `SalesOrderDetailID`).
- `vGetAllCategories`: flattened category helper keyed by `ProductCategoryID`.
- `vProductAndDescription`: product-description helper, likely one row per (`ProductID`, `Culture`) if multilingual values exist.
- `vProductModelCatalogDescription`: product model enrichment keyed by `ProductModelID`.

## Silver
Curate to conformed, cleaned, snake_case tables in the `silver` schema. Remove sensitive fields, standardize types/names, deduplicate deterministically, and add audit columns.

### Silver standards
- Rename columns to `snake_case`.
- Trim strings and normalize blank strings to null where appropriate on descriptive fields.
- Add:
  - `silver_loaded_at_utc`
  - `record_source`
  - `is_current_row` where dedup logic keeps latest record
- Deduplicate by business key using latest available timestamp, typically `modified_date`, with `rowguid` as a tiebreaker where available.
- Run `OPTIMIZE` on large Silver outputs after writes, especially sales header/detail.
- Remove or quarantine analytically sensitive/non-reporting fields:
  - Drop `password_hash`, `password_salt` from customer in Silver.
  - Drop `thumbnail_photo` from product Silver unless explicitly required for reporting.

### Suggested silver tables and dedup keys
- `silver.address`
  - Dedup key: `address_id`
  - Keep latest by `modified_date`, then `rowguid`
  - Standardize city/state/country casing only if done deterministically
- `silver.customer`
  - Dedup key: `customer_id`
  - Keep latest by `modified_date`, then `rowguid`
  - Add derived `full_name`
  - Retain `company_name`, `sales_person`, `email_address`, `phone`
- `silver.customer_address`
  - Dedup key: (`customer_id`, `address_id`, `address_type`)
  - Keep latest by `modified_date`, then `rowguid`
  - Standardize `address_type`
- `silver.product`
  - Dedup key: `product_id`
  - Keep latest by `modified_date`, then `rowguid`
  - Add lifecycle flags in dependency order:
    - `is_discontinued`
    - `is_sellable_currently`
    - optional `is_priced` based on `list_price`
- `silver.product_category`
  - Dedup key: `product_category_id`
  - Keep latest by `modified_date`, then `rowguid`
  - Preserve parent-child hierarchy columns
- `silver.product_description`
  - Dedup key: `product_description_id`
  - Keep latest by `modified_date`, then `rowguid`
- `silver.product_model`
  - Dedup key: `product_model_id`
  - Keep latest by `modified_date`, then `rowguid`
- `silver.product_model_product_description`
  - Dedup key: (`product_model_id`, `product_description_id`, `culture`)
  - Keep latest by `modified_date`, then `rowguid`
- `silver.sales_order_header`
  - Dedup key: `sales_order_id`
  - Keep latest by `modified_date`, then `sales_order_number`, then `rowguid`
  - Add derived dates:
    - `order_date_key`
    - `due_date_key`
    - `ship_date_key`
  - Add flags:
    - `is_online_order`
    - `is_shipped`
- `silver.sales_order_detail`
  - Dedup key: (`sales_order_id`, `sales_order_detail_id`)
  - Keep latest by `modified_date`, then `rowguid`
  - Add `net_unit_price` = `unit_price - unit_price_discount` only if type-safe and business-approved
- `silver.category_flat`
  - Prefer sourcing from `vGetAllCategories`
  - Dedup key: `product_category_id`
  - Use as optional flattened helper rather than core source of truth
- `silver.product_description_flat`
  - Two plausible routes:
    - Route A: prefer `vProductAndDescription` as prebuilt helper keyed by (`product_id`, `culture`)
    - Route B: derive by joining `product` -> `product_model` -> `product_model_product_description` -> `product_description`
  - Recommend Route A for simpler semantic reporting, but retain Route B as fallback if row counts mismatch
- `silver.product_model_catalog_description`
  - Dedup key: `product_model_id`
  - Keep latest by `modified_date`, then `rowguid`
  - Treat as optional enrichment for narrative/product attributes

### Silver join intent
- Join `customer_address` to `address` only in downstream curated entities; keep base Silver tables normalized.
- Join `sales_order_detail` to `sales_order_header` in Gold, not in core Silver, to preserve line grain.
- Use `product_category_id` from product to category.
- Use `product_model_id` from product to model.
- Use `product_model_id` and `product_description_id` bridge to attach descriptions.
- Prefer base-table derivations over view-based helpers when reconciling discrepancies; if view and derived output differ, log row-count comparison and choose one path explicitly.

Gold should look like this:
Dim_Customer -> join customer, customeraddress (only use the AddressType='Main Office'), address and leave all relevant/meaningfull fields
Dim_Product -> join product, productcategory, producdescription, productmodel, productmodelproductdescription (where culture='en'). Please notice that in productcategory there is a self join. So in the end I have the category (which is the parent category) and a subcategory.
Dim_Sales --> Use the salesperson field from salesorderheader, but remove the adventureworks/ in front of the name and remove the last number at the end off the name
Fact_Sales --> join salesorderdetail and salesorderheader and keep all relevant fields

## Gold
Build business-facing star-schema outputs in the `gold` schema from the conformed Silver layer.

### Gold tables
- `gold.dim_customer`
  - Grain: one row per `customer_id`
  - Join `silver.customer` to `silver.customer_address` on `customer_id`
  - Filter `silver.customer_address.address_type = 'Main Office'`
  - Join to `silver.address` on `address_id`
  - Keep relevant customer and Main Office address attributes
- `gold.dim_product`
  - Grain: one row per `product_id`
  - Join `silver.product` to `silver.product_category` using `product.product_category_id = product_category.product_category_id`
  - Self-join `silver.product_category` to resolve parent category vs subcategory
  - Join `silver.product` to `silver.product_model` using `product_model_id`
  - Join through `silver.product_model_product_description` filtered to `culture = 'en'`
  - Join to `silver.product_description` on `product_description_id`
  - Output both parent category and child subcategory attributes
- `gold.dim_sales`
  - Grain: one row per cleaned salesperson
  - Source from distinct `silver.sales_order_header.sales_person`
  - Clean rule:
    - remove `adventure-works\` or `adventureworks\` prefix if present
    - remove trailing numeric suffix
- `gold.fact_sales`
  - Grain: one row per (`sales_order_id`, `sales_order_detail_id`)
  - Join `silver.sales_order_detail` to `silver.sales_order_header` on `sales_order_id`
  - Keep relevant line and header fields needed for reporting and semantic modeling

## Semantic model
Build a Direct Lake semantic model over the Gold outputs produced from the revised star schema.

### Tables
- `dim_customer`
  - Grain: one row per `customer_id`
  - Sourced from Gold `dim_customer`
  - Expected columns to expose:
    - `customer_id`
    - `full_name`
    - `company_name`
    - `title`
    - `email_address`
    - `phone`
    - `customer_name_style` if present
    - `address_id` for the selected Main Office address
    - `address_type` (should resolve to `Main Office` where populated)
    - `address_line1`
    - `address_line2`
    - `city`
    - `state_province`
    - `country_region`
    - `postal_code`
- `dim_product`
  - Grain: one row per `product_id`
  - Sourced from Gold `dim_product`
  - Expected columns to expose:
    - `product_id`
    - `product_name`
    - `product_number`
    - `color`
    - `size`
    - `weight`
    - `standard_cost`
    - `list_price`
    - `sell_start_date`
    - `sell_end_date`
    - `discontinued_date`
    - `is_discontinued`
    - `is_sellable_currently`
    - `product_model_id`
    - `product_model_name`
    - `product_description`
    - `culture`
    - `category_id` or `product_category_id`
    - `category_name` for the parent category
    - `subcategory_id`
    - `subcategory_name`
- `dim_sales`
  - Grain: one row per cleaned salesperson value
  - Sourced from Gold `dim_sales`
  - Expected columns to expose:
    - `sales_person`
    - optional helper columns if implemented, such as cleaned/raw variants, but hide raw variants from report authors
  - Business rule:
    - built from `sales_order_header.sales_person`
    - remove `adventure-works\` or `adventureworks\` prefix if present
    - remove trailing numeric suffix from the salesperson name/code
- `fact_sales`
  - Grain: one row per sales order detail line after joining detail and header
  - Sourced from Gold `fact_sales`
  - Expected columns to expose:
    - `sales_order_id`
    - `sales_order_detail_id`
    - `sales_order_number`
    - `customer_id`
    - `product_id`
    - `sales_person`
    - `order_date`
    - `due_date`
    - `ship_date`
    - `order_date_key` if materialized in Gold
    - `due_date_key` if materialized in Gold
    - `ship_date_key` if materialized in Gold
    - `ship_to_address_id`
    - `bill_to_address_id`
    - `status`
    - `online_order_flag` or `is_online_order`
    - `order_qty`
    - `unit_price`
    - `unit_price_discount`
    - `line_total`
    - `sub_total`
    - `tax_amt`
    - `freight`
    - `total_due`

### Relationships
- `fact_sales[customer_id]` -> `dim_customer[customer_id]` many-to-one
- `fact_sales[product_id]` -> `dim_product[product_id]` many-to-one
- `fact_sales[sales_person]` -> `dim_sales[sales_person]` many-to-one
- Do not create separate semantic relationships to address tables because customer geography is already denormalized into `dim_customer`.
- If `fact_sales` also carries customer geography attributes directly from header/address joins in Gold, keep those hidden unless required; prefer the conformed dimensions above.

### Recommended explicit measures
- `Sales Amount = SUM(fact_sales[line_total])`
- `Order Count = DISTINCTCOUNT(fact_sales[sales_order_id])`
- `Units Sold = SUM(fact_sales[order_qty])`
- `Average Order Value = DIVIDE(SUM(fact_sales[total_due]), [Order Count])`
- `Average Selling Price = DIVIDE([Sales Amount], [Units Sold])`
- `Discount Amount = SUMX(fact_sales, fact_sales[unit_price_discount] * fact_sales[order_qty])`
- `Freight Amount = SUM(fact_sales[freight])`
- `Tax Amount = SUM(fact_sales[tax_amt])`
- `Online Order Count = CALCULATE([Order Count], fact_sales[online_order_flag] = TRUE())`
  - If Gold uses `is_online_order` instead of `online_order_flag`, bind the measure to that final column name instead.
- `Shipped Order Count = CALCULATE([Order Count], NOT ISBLANK(fact_sales[ship_date]))`

### Semantic modeling notes
- This model is intentionally centered on a single line-grain fact table plus three dimensions: customer, product, and salesperson.
- Because `fact_sales` contains header values repeated at line grain, use:
  - `line_total` and `order_qty` for line/product analysis
  - `total_due`, `freight`, and `tax_amt` only with caution for order-level analysis
- To avoid double counting on order-level financials, prefer either:
  - report visuals filtered to distinct orders, or
  - additional DAX measures that aggregate by distinct `sales_order_id` when needed
- Hide technical fields and duplicate raw variants, especially:
  - raw salesperson text if both raw and cleaned forms exist
  - surrogate/helper keys not intended for authoring
  - culture if it is fixed to `en`
- Expose a product hierarchy:
  - `category_name` > `subcategory_name` > `product_name`
- Expose a customer geography hierarchy:
  - `country_region` > `state_province` > `city` > `postal_code`
- If `dim_product` contains only English descriptions by design, no culture slicer is needed.

## Report
Create a Power BI report tailored to the revised Gold model with customer, product, salesperson, and fact-level sales analysis. Always include a Data quality page.

### Page 1: Sales Overview
- KPI cards:
  - Sales Amount
  - Order Count
  - Units Sold
  - Average Order Value
- Line chart:
  - Sales Amount by `fact_sales[order_date]`
- Clustered column chart:
  - Sales Amount by `dim_product[category_name]`
- Bar chart:
  - Sales Amount by `dim_sales[sales_person]`
- Matrix:
  - `dim_product[category_name]` -> `dim_product[subcategory_name]` -> `dim_product[product_name]`
  - Values:
    - Sales Amount
    - Units Sold
    - Average Selling Price

### Page 2: Customer and Geography
- Bar chart:
  - Top customers by Sales Amount using `dim_customer[full_name]` or `dim_customer[company_name]`
- Map or filled map if geography quality is sufficient:
  - Sales Amount by `dim_customer[country_region]`, `dim_customer[state_province]`, and `dim_customer[city]`
- Table:
  - `dim_customer[full_name]`
  - `dim_customer[company_name]`
  - `dim_customer[email_address]`
  - `dim_customer[phone]`
  - `dim_customer[address_line1]`
  - `dim_customer[city]`
  - `dim_customer[state_province]`
  - Order Count
  - Sales Amount
- Slicer set:
  - `fact_sales[order_date]`
  - `dim_customer[country_region]`
  - `dim_customer[state_province]`
  - `dim_sales[sales_person]`

### Page 3: Product Performance
- Bar chart:
  - Top products by Sales Amount or Units Sold using `dim_product[product_name]`
- Scatter chart:
  - `dim_product[list_price]` vs Sales Amount by `dim_product[product_name]`
- Table:
  - `dim_product[product_name]`
  - `dim_product[product_number]`
  - `dim_product[color]`
  - `dim_product[size]`
  - `dim_product[standard_cost]`
  - `dim_product[list_price]`
  - `dim_product[category_name]`
  - `dim_product[subcategory_name]`
  - Sales Amount
  - Units Sold
- Optional decomposition tree:
  - Sales Amount by `category_name` > `subcategory_name` > `product_name`
- Optional detail card or table:
  - `dim_product[product_model_name]`
  - `dim_product[product_description]`
  - `dim_product[is_sellable_currently]`
  - `dim_product[is_discontinued]`

### Page 4: Salesperson Analysis
- Bar chart:
  - Sales Amount by `dim_sales[sales_person]`
- Table:
  - `dim_sales[sales_person]`
  - Order Count
  - Units Sold
  - Sales Amount
  - Average Order Value
- Trend visual:
  - Sales Amount by `fact_sales[order_date]` split by `dim_sales[sales_person]`
- Slicers:
  - `dim_sales[sales_person]`
  - `dim_product[category_name]`
  - `fact_sales[order_date]`

### Page 5: Data quality
- Cards:
  - Gold `fact_sales` row count
  - Gold `dim_customer` distinct customer count
  - Gold `dim_product` distinct product count
  - Gold `dim_sales` distinct salesperson count
- Validation cards or KPI indicators:
  - customers missing Main Office address
  - products missing English description
  - products missing parent category or subcategory
  - fact rows with missing customer, product, or salesperson key
- Table:
  - failing test name
  - table
  - severity
  - failed row count
  - run id
- Bar chart:
  - failures by Gold table
- Detail grid:
  - sample missing `customer_id`, `product_id`, or cleaned `sales_person`
  - sample products with null `category_name` / `subcategory_name`

### Report design notes
- Prefer `fact_sales` for all core metrics because Gold now centers on a single joined fact.
- Use `dim_sales[sales_person]` everywhere instead of raw salesperson text from the fact.
- Use `dim_customer` for geography because Main Office address fields are already denormalized there.
- Use the product hierarchy from `dim_product`:
  - parent category as `category_name`
  - child category as `subcategory_name`
- Do not expose raw or uncleaned salesperson strings in visuals.
- Because product descriptions are constrained to English in Gold, no culture slicer is required.
- Be cautious with order-level measures derived from repeated header fields on the line fact; validate card logic so it does not unintentionally double count.

## Data Agent
Create an AISkill grounded on the semantic model for sales, customer, product, and salesperson analytics.

### Role
You are a sales analytics assistant for this dataset. Help users understand sales trends, customer performance, product performance, category/subcategory mix, pricing and discount behavior, salesperson performance, and customer geography using the curated semantic model only.

### Domain hints
- The model contains:
  - `dim_customer` with customer attributes and Main Office address details
  - `dim_product` with product, model, English description, parent category, and subcategory
  - `dim_sales` with cleaned salesperson names derived from `sales_order_header.sales_person`
  - `fact_sales` at sales order line grain
- Customer geography comes from the Main Office address chosen in Gold from `customer_address` where `address_type = 'Main Office'`.
- Product enrichment comes from joining product, product category, product model, product model product description, and product description with `culture = 'en'`.
- Product category is hierarchical:
  - parent category = category
  - child category = subcategory
- Salesperson values are cleaned from the source field by removing the AdventureWorks prefix and trailing number suffix.
- `fact_sales` contains both line-level and repeated order-header attributes because it is built by joining `sales_order_detail` and `sales_order_header`.

### Starter questions
- What are total sales, total orders, and units sold by month?
- Which parent categories and subcategories generate the highest sales?
- Which customers have the highest total sales?
- Which salespeople generate the most sales amount and order volume?
- What are the top products by units sold and by sales amount?
- Which countries, states, or cities have the highest sales based on customer Main Office address?
- What is the average order value over time?
- How much discount amount was given by category, subcategory, or product?
- Which products are currently sellable versus discontinued?
- Which products are missing category or English description attributes?

### Guardrails
- Answer only from the semantic model; do not invent data or infer unavailable fields.
- Do not expose or discuss removed sensitive source fields such as password hashes or salts.
- Distinguish clearly between:
  - line-level measures such as `order_qty`, `unit_price`, `unit_price_discount`, and `line_total`
  - order-level amounts such as `total_due`, `tax_amt`, and `freight`
- Warn about possible double counting when a question aggregates repeated header amounts from `fact_sales` without distinct-order logic.
- Use cleaned salesperson values from `dim_sales`; do not present raw prefixed source names unless explicitly modeled and requested.
- When discussing geography, state that it is based on the customer Main Office address, not ship-to or bill-to order address.
- When discussing product descriptions, state that the implemented Gold model uses English (`culture = 'en'`) descriptions.
- If a question depends on unavailable attributes, say what is missing and suggest the closest supported analysis.
- Prefer curated measures over implicit aggregation.
- Keep responses analytical, concise, and business-facing.