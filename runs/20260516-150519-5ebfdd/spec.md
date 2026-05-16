# Run Spec 20260516-145843-ade58d

## Updated specs

### Iteration 1 — 2026-05-16 15:53:24Z — failed layer: bronze (run: 20260516-150519-5ebfdd)
- **Root cause (1-line summary)**: Bronze ingestion timed out after 45 minutes due to trying to land all listed source objects, including source views and binary-heavy columns, in a single long-running path without per-object limits/checkpoints.
- **What was changed**:
  - Tightened **Generic guidance** to require one source object per bounded ingestion unit with progress persisted after each successful bronze write.
  - Tightened **Bronze** to prioritize/base-table ingestion first, defer source views until after base tables succeed, and explicitly minimize hashing/processing on binary columns such as `thumb_nail_photo`.
  - Added explicit runtime guardrails for bronze reads/writes, including fail-loud per-table timeout behavior and no full-build dependency on bronze views.

### Iteration 2 — 2026-05-16 16:01:29Z — failed layer: gold (run: 20260516-150519-5ebfdd)
- **Root cause (1-line summary)**: Gold build caused Spark statement failures due to underspecified joins/grain in dimension construction, especially `dim_customer` and `dim_product`, which can create row explosion or unstable joins.
- **What was changed**:
  - Tightened **Generic guidance** to require explicit join cardinality, join keys, and post-join row-count assertions for every gold build.
  - Tightened **Gold** to define exact source tables, join keys, preferred culture rule, category self-join logic, and one-row-per-key grain rules for `dim_customer`, `dim_product`, and `dim_sales`.
  - Tightened **Semantic model** to align with the enforced gold grains, especially requiring `dim_customer` to remain one row per `customer_id` and keeping multi-address analysis in `bridge_customer_address`/`dim_address`.

## Inputs
- Workspace: `4e8e1b48-3d46-48b6-b370-80eb5b06d2fb`
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
- Target Lakehouse: **CopilotMedallion_20260516_145843**

## Generic guidance
Apply these reference skills/agents at all times:
- FabricDataEngineer agent: https://github.com/microsoft/skills-for-fabric/blob/main/agents/FabricDataEngineer.agent.md
- e2e-medallion-architecture skill: https://github.com/microsoft/skills-for-fabric/tree/main/skills/e2e-medallion-architecture
- spark-authoring-cli skill: https://github.com/microsoft/skills-for-fabric/tree/main/skills/spark-authoring-cli
- powerbi-authoring-cli skill: https://github.com/microsoft/skills-for-fabric/tree/main/skills/powerbi-authoring-cli
- powerbi-consumption-cli skill: https://github.com/microsoft/skills-for-fabric/tree/main/skills/powerbi-consumption-cli
- powerbi-semantic-model-authoring: https://github.com/RuiRomano/powerbi-agentic-plugins/tree/main/plugins/powerbi/skills/powerbi-semantic-model-authoring
- powerbi-report-authoring: https://github.com/RuiRomano/powerbi-agentic-plugins/tree/main/plugins/powerbi/skills/powerbi-report-authoring

Standard cross-cutting code rules:
- Use defensive column references in all transformations; assert required columns exist before select, join, cast, rename, groupBy, or merge logic.
- After every join, immediately project to an alias-prefixed or explicitly selected column list to avoid duplicate-column ambiguity.
- Before every `groupBy` / `agg`, assert grouping and measure columns exist.
- For REST/API handling, use defensive null checks such as `if x is None: raise ...` BEFORE any `.get()` access.
- Do not use `saveAsTable`; write with idempotent Delta patterns to Lakehouse paths or managed table write APIs supported by Fabric notebooks.
- Use parameter cells at the top of every notebook for workspace, lakehouse, schema, table, run id, and load mode.
- Use idempotent overwrite/upsert patterns so reruns of the same notebook do not create duplicates.
- Use error-loud `try/except` blocks that call `_save_error(layer, e)` and then re-raise the exception.
- Include audit metadata on writes where appropriate: ingestion timestamp, run id, source table name, record hash where useful, and processing timestamp.
- Prefer explicit schema handling over inference for stable pipelines.
- Exclude binary payloads from gold unless there is a clear analytics use case.
- For bronze orchestration, process one source object at a time and persist success state after each successful bronze write.
- Do not make bronze completion of source views a prerequisite for base-table bronze success; base tables are the required first-class ingestion targets.
- Any expensive per-row computation in bronze, especially hashing across many columns, must be scoped to a single object write and must exclude binary columns and large text blobs unless explicitly required.
- Keep bronze notebooks operationally bounded: each source object should have its own read-transform-write unit with logging of row count, start/end time, and object name.
- For every gold-table join, explicitly document and implement:
  - left/right aliases,
  - join type,
  - exact join keys,
  - expected cardinality (`1:1`, `1:many`, `many:1`),
  - and the target grain after the join.
- In gold, do not use convenience joins that multiply rows at the target dimension grain. If a supporting table is `1:many`, first reduce it to one row per target business key using an explicit ranking or aggregation rule before joining.
- After building each gold dimension/fact, assert the final row count and key uniqueness for the intended grain before writing the Delta table. Fail loudly if grain is violated.

Notebook authoring requirement for every generated notebook:
- EACH code cell must start with a short markdown comment block using Python comments.
- Required pattern:
  - `# ---`
  - `# <1-3 human-readable lines explaining what the cell does and why>`
- Never emit a code cell with no leading comment block.
- Keep comments human-readable and scannable so a Fabric user can understand the notebook by scrolling.

## Bronze
Purpose: land each selected source object into a `bronze` schema in the target lakehouse with minimal transformation, preserving source fidelity and adding ingestion metadata.

Landing pattern for all source objects:
- Source read: `SalesLT/<object>`
- Target naming: `bronze.<object_name_snake_case>`
- Write mode: full refresh overwrite per run, because no reliable source CDC column beyond `ModifiedDate` is guaranteed to represent deletes.
- File format: Delta
- Orchestration requirement:
  - Ingest each listed source object as an independent bounded step.
  - Commit/write each bronze table immediately after that object's read completes; do not hold multiple source objects in memory before writing.
  - Log per-object start time, end time, row count, and write target.
  - If a later object fails or times out, previously written bronze tables remain valid and should not be reprocessed unnecessarily on rerun.
- Ingestion order requirement:
  - Ingest base tables first in this order:
    1. `SalesLT/Address`
    2. `SalesLT/Customer`
    3. `SalesLT/CustomerAddress`
    4. `SalesLT/Product`
    5. `SalesLT/ProductCategory`
    6. `SalesLT/ProductDescription`
    7. `SalesLT/ProductModel`
    8. `SalesLT/ProductModelProductDescription`
    9. `SalesLT/SalesOrderHeader`
    10. `SalesLT/SalesOrderDetail`
  - Ingest source views only after all base tables above have been successfully written:
    11. `SalesLT/vGetAllCategories`
    12. `SalesLT/vProductAndDescription`
    13. `SalesLT/vProductModelCatalogDescription`
  - If runtime budget becomes constrained, base tables take priority and source views may be skipped/deferred without failing the bronze delivery of the core medallion path.
- Partitioning:
  - Do not partition small/master tables.
  - Partition `bronze.sales_order_header` by `year(order_date)` if volume justifies it.
  - Partition `bronze.sales_order_detail` by `sales_order_id` is not recommended; leave unpartitioned initially and rely on Delta optimization.
- Metadata columns added to every bronze table:
  - `ingestion_run_id`
  - `ingestion_ts`
  - `source_lakehouse_name`
  - `source_object_name`
  - `source_modified_date` when source has `ModifiedDate`
  - `bronze_record_hash` over all non-binary business columns
- Record-hash performance rule:
  - Compute `bronze_record_hash` only from non-binary source columns.
  - Explicitly exclude `ThumbNailPhoto` and any other binary payload columns from hashing.
  - For very wide source views, `bronze_record_hash` is optional and may be omitted if the view lands successfully without it; if omitted, keep the other ingestion metadata columns.
- Binary handling:
  - `SalesLT/Product.ThumbNailPhoto` should be retained in bronze only.
  - Do not propagate `ThumbNailPhoto` to silver/gold unless the user later requests image analytics.
  - When landing `SalesLT/Product`, do not perform any transformations on `ThumbNailPhoto` other than pass-through retention in bronze.

Table-specific bronze landing:
- `SalesLT/Address` -> `bronze.address`
  - Preserve all columns.
  - Candidate PK observed: `AddressID`
- `SalesLT/Customer` -> `bronze.customer`
  - Preserve all columns.
  - Candidate PK observed: `CustomerID`
  - Sensitive-like fields detected: `PasswordHash`, `PasswordSalt`; retain in bronze only
- `SalesLT/CustomerAddress` -> `bronze.customer_address`
  - Preserve all columns.
  - Junction table with composite natural key likely: `CustomerID`, `AddressID`, `AddressType`
- `SalesLT/Product` -> `bronze.product`
  - Preserve all columns including binary thumbnail.
  - Candidate PK observed: `ProductID`
  - Performance note: retain `ThumbNailPhoto` exactly as-is and exclude it from record-hash logic.
- `SalesLT/ProductCategory` -> `bronze.product_category`
  - Preserve all columns.
  - Candidate PK observed: `ProductCategoryID`
- `SalesLT/ProductDescription` -> `bronze.product_description`
  - Preserve all columns.
  - Candidate PK observed: `ProductDescriptionID`
- `SalesLT/ProductModel` -> `bronze.product_model`
  - Preserve all columns.
  - Candidate PK observed: `ProductModelID`
- `SalesLT/ProductModelProductDescription` -> `bronze.product_model_product_description`
  - Preserve all columns.
  - Bridge table; likely composite key: `ProductModelID`, `ProductDescriptionID`, `Culture`
- `SalesLT/SalesOrderDetail` -> `bronze.sales_order_detail`
  - Preserve all columns.
  - Candidate PK observed: `SalesOrderDetailID`; also FK-like `SalesOrderID`, `ProductID`
- `SalesLT/SalesOrderHeader` -> `bronze.sales_order_header`
  - Preserve all columns.
  - Candidate PK observed: `SalesOrderID`; FK-like `CustomerID`, `ShipToAddressID`, `BillToAddressID`
- `SalesLT/vGetAllCategories` -> `bronze.v_get_all_categories`
  - Treat as source view snapshot.
  - No `ModifiedDate`; add only ingestion metadata
  - Optional/deferred bronze object; do not block core bronze success if this view is skipped for runtime reasons.
- `SalesLT/vProductAndDescription` -> `bronze.v_product_and_description`
  - Treat as source view snapshot.
  - Optional/deferred bronze object; do not block core bronze success if this view is skipped for runtime reasons.
- `SalesLT/vProductModelCatalogDescription` -> `bronze.v_product_model_catalog_description`
  - Treat as source view snapshot.
  - Optional/deferred bronze object; do not block core bronze success if this view is skipped for runtime reasons.

## Silver
Purpose: standardize names and types, remove duplicates, separate analytics-ready entities from raw source replicas, and prepare conformed keys for gold.

Shared silver rules:
- Rename all columns to `snake_case`.
- Standardize timestamps to timestamp type; decimals preserved with source precision where practical.
- Add:
  - `silver_processed_ts`
  - `silver_run_id`
  - `is_current_record` default true for current-state snapshots
- Deduplicate using the most reliable business key available, keeping latest by `modified_date` then `ingestion_ts`.
- Drop technical columns not needed for analytics from promoted entities where appropriate.
- Mask or exclude sensitive fields from analytics-facing silver tables:
  - Exclude `password_hash`
  - Exclude `password_salt`
- OPTIMIZE all silver Delta tables after load; consider ZORDER on main keys for larger transactional tables.
- Silver builds must not require bronze source views; if `bronze.v_get_all_categories`, `bronze.v_product_and_description`, or `bronze.v_product_model_catalog_description` are absent because they were deferred in bronze, build silver from the corresponding base tables and treat the view-based silver tables as optional helpers.

Silver tables and dedup logic:
- `silver.address`
  - Source: `bronze.address`
  - Dedup key: `address_id`
  - Keep descriptive fields: `address_line1`, `address_line2`, `city`, `state_province`, `country_region`, `postal_code`, `rowguid`, `modified_date`
  - OPTIMIZE ZORDER: `address_id`, `city`, `state_province`
- `silver.customer`
  - Source: `bronze.customer`
  - Dedup key: `customer_id`
  - Build derived `full_name` from `title`, `first_name`, `middle_name`, `last_name`, `suffix`
  - Exclude `password_hash`, `password_salt`
  - OPTIMIZE ZORDER: `customer_id`, `company_name`
- `silver.customer_address`
  - Source: `bronze.customer_address`
  - Dedup key: `customer_id`, `address_id`, `address_type`
  - This is a junction table; preserve `address_type`
  - OPTIMIZE ZORDER: `customer_id`, `address_id`
- `silver.product`
  - Source: `bronze.product`
  - Dedup key: `product_id`
  - Exclude `thumbnail_photo`
  - Keep file name only if useful: `thumbnail_photo_file_name`
  - Preserve cost and price metrics, category/model IDs, sell/discontinue dates
  - OPTIMIZE ZORDER: `product_id`, `product_category_id`, `product_model_id`
- `silver.product_category`
  - Source: `bronze.product_category`
  - Dedup key: `product_category_id`
  - Keep parent-child hierarchy fields
  - OPTIMIZE ZORDER: `product_category_id`, `parent_product_category_id`
- `silver.product_description`
  - Source: `bronze.product_description`
  - Dedup key: `product_description_id`
- `silver.product_model`
  - Source: `bronze.product_model`
  - Dedup key: `product_model_id`
  - Retain `catalog_description`
- `silver.product_model_product_description`
  - Source: `bronze.product_model_product_description`
  - Dedup key: `product_model_id`, `product_description_id`, `culture`
- `silver.sales_order_header`
  - Source: `bronze.sales_order_header`
  - Dedup key: `sales_order_id`
  - Keep transactional header columns and derive:
    - `order_date_key` as `yyyyMMdd` integer
    - `due_date_key`
    - `ship_date_key`
  - OPTIMIZE ZORDER: `sales_order_id`, `customer_id`, `order_date`
- `silver.sales_order_detail`
  - Source: `bronze.sales_order_detail`
  - Dedup key: `sales_order_detail_id`
  - Preserve `sales_order_id`, `product_id`, pricing, quantity, discount, line_total`
  - OPTIMIZE ZORDER: `sales_order_id`, `product_id`
- `silver.v_get_all_categories`
  - Source: `bronze.v_get_all_categories`
  - Dedup key: `product_category_id`
  - Useful as flattened category helper
  - Optional helper table only
- `silver.v_product_and_description`
  - Source: `bronze.v_product_and_description`
  - Dedup key: `product_id`, `culture`
  - Useful for multilingual/descriptive enrichments
  - Optional helper table only
- `silver.v_product_model_catalog_description`
  - Source: `bronze.v_product_model_catalog_description`
  - Dedup key: `product_model_id`
  - Useful for product attributes not present in base tables
  - Optional helper table only

Silver modeling notes and alternatives:
- Preferred approach: treat source views as enrichment helpers, not authoritative dimensions, because they appear derived from base tables.
- Alternative: if users prefer fewer joins and simpler gold, use `v_get_all_categories`, `v_product_and_description`, and `v_product_model_catalog_description` as primary enrichment sources for dimension builds.
- Address role handling:
  - Preferred: keep one `dim_address` and create two relationships from fact via ship/bill role-playing.
  - Alternative: materialize `dim_ship_to_address` and `dim_bill_to_address` separately for simpler self-service modeling.

## Gold
Proposed gold model is a sales star schema centered on order-line analytics because `sales_order_detail` is the clearest fact-like table and `sales_order_header` provides order-level measures and customer/address context.

Most important specs:
Dim_Customer -> join customer, customer address, address and leave all relevant/meaningfull fields
Dim_Product -> join product, productcategory, producdescription, productmodel, productmodelproductdescription. Please notice that in productcategory there is a self join. So in the end I have the category (which is the parent category) and a subcategory.
Dim_Sales --> Use the salesperson field from salesorderheader, but remove the adventureworks/ in front of the name and remove the last number at the end off the name

Use the below to accomplish the most important specs
Entity classification based on actual columns:
- Fact-like:
  - `sales_order_detail` because it is transactional, numeric, and FK-heavy
  - `sales_order_header` can support an order-header fact or enrich the line fact
- Dimension-like:
  - `customer`
  - `address`
  - `product`
  - `product_category`
  - `product_model`
  - `product_description` as optional text dimension
- Junction/bridge:
  - `customer_address`
  - `product_model_product_description`
- Derived/helper views:
  - `v_get_all_categories`
  - `v_product_and_description`
  - `v_product_model_catalog_description`

Gold build order and hard constraints:
- Build gold tables in this order:
  1. `gold.dim_date`
  2. `gold.dim_address`
  3. `gold.bridge_customer_address`
  4. `gold.dim_customer`
  5. `gold.dim_product`
  6. `gold.dim_sales`
  7. `gold.fact_sales_order_header`
  8. `gold.fact_sales_order_line`
- For every gold table, select only the required columns listed in this spec before writing.
- Do not use `select("*")` after joins in gold.
- All joins in gold must use DataFrame aliases and explicit equality predicates.
- If an enrichment source can introduce more than one row per target key, reduce it first with a deterministic rule before joining.

### Proposed gold dimensions
- `gold.dim_customer`
  - Grain: exactly one row per `customer_id`
  - Base source: `silver.customer` alias `c`
  - Required supporting sources:
    - `silver.customer_address` alias `ca`
    - `silver.address` alias `a`
  - Join intent:
    - `c.customer_id = ca.customer_id` is `1:many`
    - `ca.address_id = a.address_id` is `many:1`
  - To preserve one row per `customer_id`, DO NOT directly join all customer-address rows into the final dimension.
  - Required reduction rule before final join:
    - Build an intermediate customer-address choice table with at most one row per `customer_id`.
    - Preferred ranking:
      1. `address_type = 'Main Office'`
      2. else `address_type = 'Primary'`
      3. else `address_type = 'Shipping'`
      4. else `address_type = 'Billing'`
      5. else alphabetical by `address_type`
      6. then lowest `address_id` as final tie-breaker
    - Join `c` to that reduced one-row-per-customer table only.
  - Final join keys:
    - `c.customer_id = chosen_ca.customer_id`
    - `chosen_ca.address_id = a.address_id`
  - Final columns:
    - `customer_id`
    - `full_name`
    - `name_style`
    - `title`
    - `first_name`
    - `middle_name`
    - `last_name`
    - `suffix`
    - `company_name`
    - `sales_person`
    - `email_address`
    - `phone`
    - `address_type`
    - `address_id`
    - `address_line1`
    - `address_line2`
    - `city`
    - `state_province`
    - `country_region`
    - `postal_code`
    - `modified_date`
  - Write-time assertions:
    - `customer_id` must be non-null
    - `customer_id` must be unique in `gold.dim_customer`
  - Multi-address requirement:
    - Publish all customer-to-address combinations separately in `gold.bridge_customer_address`
    - Do not duplicate customers inside `gold.dim_customer` to represent multiple addresses

- `gold.dim_address`
  - Grain: one row per `address_id`
  - Source: `silver.address`
  - Columns:
    - `address_id`
    - `address_line1`
    - `address_line2`
    - `city`
    - `state_province`
    - `country_region`
    - `postal_code`
    - `modified_date`

- `gold.dim_product`
  - Grain: exactly one row per `product_id`
  - Base source: `silver.product` alias `p`
  - Required supporting sources:
    - `silver.product_category` alias `pc_child`
    - `silver.product_category` alias `pc_parent`
    - `silver.product_model` alias `pm`
    - `silver.product_model_product_description` alias `pmpd`
    - `silver.product_description` alias `pd`
  - Join keys:
    - `p.product_category_id = pc_child.product_category_id` (`many:1`)
    - `pc_child.parent_product_category_id = pc_parent.product_category_id` (`many:1`, self-join)
    - `p.product_model_id = pm.product_model_id` (`many:1`)
    - `p.product_model_id = pmpd.product_model_id` (`1:many` before reduction)
    - `pmpd.product_description_id = pd.product_description_id` (`many:1`)
  - Category naming rule:
    - `category_name` = `pc_parent.name`
    - `subcategory_name` = `pc_child.name`
    - `parent_category_name` = `pc_parent.name`
  - Because `pmpd` can create multiple rows per `product_model_id`, reduce description rows before joining to product.
  - Required description reduction rule:
    - First join `pmpd` to `pd` on `product_description_id`
    - Then create one preferred description row per `product_model_id` using:
      1. `culture = 'en'` preferred if present
      2. else first non-null `culture` in ascending order
      3. then lowest `product_description_id` as tie-breaker
    - Join `p` to this reduced one-row-per-`product_model_id` description set only.
  - Final columns:
    - `product_id`
    - `product_name` from `p.name`
    - `product_number`
    - `color`
    - `standard_cost`
    - `list_price`
    - `size`
    - `weight`
    - `product_category_id`
    - `product_model_id`
    - `sell_start_date`
    - `sell_end_date`
    - `discontinued_date`
    - `category_name`
    - `subcategory_name`
    - `parent_category_name`
    - `product_model_name` from `pm.name`
    - `culture`
    - `product_description` from `pd.description`
  - Optional model attributes:
    - only include `manufacturer`, `material`, `product_line`, `style`, `rider_experience` if those columns were successfully parsed into silver and explicitly exist
  - Write-time assertions:
    - `product_id` must be non-null
    - `product_id` must be unique in `gold.dim_product`

- `gold.dim_date`
  - Grain: one row per date
  - Built from min/max of `order_date`, `due_date`, `ship_date`, `sell_start_date`, `sell_end_date`, `discontinued_date`
  - Standard attributes: date_key, full_date, year, quarter, month, month_name, week, day, weekday, is_month_end

- `gold.dim_sales`
  - Grain: one row per cleaned salesperson value
  - Source: distinct values from `silver.sales_order_header.sales_person`
  - Build only from non-null, non-empty `sales_person`
  - Cleaning rule:
    - remove case-insensitive prefix `adventureworks/`
    - then remove trailing digits at the end of the remaining string
    - then trim whitespace
  - Required output columns:
    - `sales_person_clean`
    - `sales_person_original`
  - Write-time assertions:
    - `sales_person_clean` must be non-null
    - `sales_person_clean` must be unique

### Proposed gold facts
- `gold.fact_sales_order_line`
  - Grain: one row per `sales_order_detail_id`
  - Source: `silver.sales_order_detail` alias `d` joined to `silver.sales_order_header` alias `h`
  - Join key:
    - `d.sales_order_id = h.sales_order_id`
  - Cardinality:
    - `d -> h` is `many:1`
  - Required selected columns:
    - `sales_order_detail_id`
    - `sales_order_id`
    - `order_date_key`
    - `due_date_key`
    - `ship_date_key`
    - `customer_id`
    - `product_id`
    - `ship_to_address_id`
    - `bill_to_address_id`
    - `sales_person`
    - `sales_order_number`
    - `purchase_order_number`
    - `account_number`
    - `ship_method`
    - `credit_card_approval_code`
    - `status`
    - `online_order_flag`
    - `order_qty`
    - `unit_price`
    - `unit_price_discount`
    - `line_total`
    - `header_sub_total` from `h.sub_total`
    - `header_tax_amt` from `h.tax_amt`
    - `header_freight` from `h.freight`
    - `header_total_due` from `h.total_due`
  - Add cleaned salesperson column using the same rule as `gold.dim_sales`:
    - `sales_person_clean`
  - Write-time assertions:
    - `sales_order_detail_id` must be unique

- `gold.fact_sales_order_header`
  - Grain: one row per `sales_order_id`
  - Source: `silver.sales_order_header`
  - Required columns:
    - `sales_order_id`
    - `order_date_key`
    - `due_date_key`
    - `ship_date_key`
    - `customer_id`
    - `ship_to_address_id`
    - `bill_to_address_id`
    - `sales_person`
    - `sales_person_clean`
    - `sub_total`
    - `tax_amt`
    - `freight`
    - `total_due`
    - `sales_order_number`
    - `purchase_order_number`
    - `account_number`
    - `ship_method`
    - `credit_card_approval_code`
    - `status`
    - `online_order_flag`
  - Add cleaned salesperson column using the same rule as `gold.dim_sales`
  - Write-time assertions:
    - `sales_order_id` must be unique

- `gold.bridge_customer_address`
  - Grain: one row per `customer_id`, `address_id`, `address_type`
  - Source: `silver.customer_address` joined to `silver.address`
  - Join key:
    - `customer_address.address_id = address.address_id`
  - Required columns:
    - `customer_id`
    - `address_id`
    - `address_type`
    - `address_line1`
    - `address_line2`
    - `city`
    - `state_province`
    - `country_region`
    - `postal_code`
  - Write-time assertions:
    - composite key `customer_id`, `address_id`, `address_type` must be unique

Gold modeling alternatives:
- Preferred star:
  - Use both `fact_sales_order_line` and `fact_sales_order_header`
  - Best balance between order-level and line-level analysis
- Simpler alternative:
  - Only publish `fact_sales_order_line` and hide repeated header amount columns
  - Use distinct count orders for order-level KPIs
- Richer product alternative:
  - Split product-related enrichment into `dim_product`, `dim_product_model`, and `dim_product_description`
  - Better normalization, but more complex semantic model

### Data quality tests
1. Row count reconciliation
- Assert bronze vs silver counts are equal after dedup for tables expected unique by PK (`address`, `customer`, `product`, `product_category`, `product_description`, `product_model`, `sales_order_header`)
- For junction/view tables, allow silver count <= bronze count and log removed duplicates
- Assert `gold.dim_customer` row count equals distinct `silver.customer.customer_id`
- Assert `gold.dim_product` row count equals distinct `silver.product.product_id`
- Assert `gold.fact_sales_order_header` row count equals distinct `silver.sales_order_header.sales_order_id`
- Assert `gold.fact_sales_order_line` row count equals distinct `silver.sales_order_detail.sales_order_detail_id`

2. No-null primary keys
- `address.address_id`
- `customer.customer_id`
- `product.product_id`
- `product_category.product_category_id`
- `product_description.product_description_id`
- `product_model.product_model_id`
- `sales_order_header.sales_order_id`
- `sales_order_detail.sales_order_detail_id`

3. Unique primary keys
- Assert uniqueness of:
  - `address_id`
  - `customer_id`
  - `product_id`
  - `product_category_id`
  - `product_description_id`
  - `product_model_id`
  - `sales_order_id`
  - `sales_order_detail_id`
- Assert composite uniqueness of:
  - `customer_id`, `address_id`, `address_type` in `customer_address`
  - `product_model_id`, `product_description_id`, `culture` in `product_model_product_description`
- Assert uniqueness of:
  - `gold.dim_customer.customer_id`
  - `gold.dim_product.product_id`
  - `gold.dim_sales.sales_person_clean`
  - `gold.fact_sales_order_header.sales_order_id`
  - `gold.fact_sales_order_line.sales_order_detail_id`

4. Referential integrity
- `sales_order_header.customer_id` -> `customer.customer_id`
- `sales_order_header.ship_to_address_id` -> `address.address_id`
- `sales_order_header.bill_to_address_id` -> `address.address_id`
- `sales_order_detail.sales_order_id` -> `sales_order_header.sales_order_id`
- `sales_order_detail.product_id` -> `product.product_id`
- `product.product_category_id` -> `product_category.product_category_id`
- `product.product_model_id` -> `product_model.product_model_id`
- `customer_address.customer_id` -> `customer.customer_id`
- `customer_address.address_id` -> `address.address_id`
- `product_model_product_description.product_model_id` -> `product_model.product_model_id`
- `product_model_product_description.product_description_id` -> `product_description.product_description_id`

5. Domain/value checks
- `order_qty > 0`
- `unit_price >= 0`
- `unit_price_discount >= 0`
- `line_total >= 0`
- `sub_total >= 0`, `tax_amt >= 0`, `freight >= 0`, `total_due >= 0`
- `ship_date >= order_date` where ship date is not null
- `due_date >= order_date` where due date is not null
- Optional consistency check: compare `sales_order_header.sub_total` to sum of related `sales_order_detail.line_total` within toleranceals

## Semantic model
Mode: Direct Lake on the gold layer.

Most important specs:
create dimensions and facts based on the actual tables in gold.

Supporting specs:
Proposed tables in semantic model:
- `dim_date`
- `dim_customer`
- `dim_address`
- `dim_product`
- `dim_sales`
- `fact_sales_order_header`
- `fact_sales_order_line`
- Optional hidden helper: `bridge_customer_address`

Table expectations aligned to gold:
- `dim_customer`
  - One row per `customer_id`
  - Must reflect the gold requirement to join `customer`, `customer_address`, and `address`
  - Must use the reduced single-address-per-customer result from gold, not the full customer-address bridge
  - Expose customer attributes plus meaningful address attributes that are safe for analysis
  - Recommended visible columns:
    - `customer_id`
    - `full_name`
    - `company_name`
    - `sales_person`
    - `email_address`
    - `phone`
    - customer-address fields where present in gold, such as:
      - `address_type`
      - `address_id`
      - `address_line1`
      - `address_line2`
      - `city`
      - `state_province`
      - `country_region`
      - `postal_code`
- `dim_product`
  - One row per `product_id`
  - Must reflect the gold requirement to join product, category hierarchy, model, description, and model-description bridge
  - Must use the reduced single-description-per-product-model rule from gold so the table stays one row per `product_id`
  - Recommended visible columns:
    - `product_id`
    - `product_name`
    - `product_number`
    - `color`
    - `size`
    - `weight`
    - `standard_cost`
    - `list_price`
    - `product_model_name`
    - `parent_category_name`
    - `category_name`
    - `subcategory_name`
    - `product_description`
    - `culture`
- `dim_sales`
  - One row per cleaned salesperson value derived from `sales_order_header.sales_person`
  - Cleaning rule already implemented in gold:
    - remove `adventureworks/` prefix
    - remove trailing numeric suffix
  - Recommended columns:
    - `sales_person_clean`
    - optional original source value if retained in gold as hidden column
- `fact_sales_order_header`
  - Keep order-level financial totals for correct aggregation
  - Recommended columns:
    - `sales_order_id`
    - `order_date_key`
    - `due_date_key`
    - `ship_date_key`
    - `customer_id`
    - `ship_to_address_id`
    - `bill_to_address_id`
    - `sales_person_clean` or the corresponding sales dimension key
    - `sub_total`
    - `tax_amt`
    - `freight`
    - `total_due`
    - `online_order_flag`
    - `status`
    - `ship_method`
- `fact_sales_order_line`
  - Keep product and unit-level analytics
  - Recommended columns:
    - `sales_order_detail_id`
    - `sales_order_id`
    - `order_date_key`
    - `due_date_key`
    - `ship_date_key`
    - `customer_id`
    - `product_id`
    - `ship_to_address_id`
    - `bill_to_address_id`
    - `sales_person_clean` or the corresponding sales dimension key
    - `order_qty`
    - `unit_price`
    - `unit_price_discount`
    - `line_total`

Relationships:
- `fact_sales_order_line[customer_id]` -> `dim_customer[customer_id]` many-to-one
- `fact_sales_order_line[product_id]` -> `dim_product[product_id]` many-to-one
- `fact_sales_order_line[order_date_key]` -> `dim_date[date_key]` many-to-one
- `fact_sales_order_line[sales_person_clean]` -> `dim_sales[sales_person_clean]` many-to-one if `dim_sales` uses the cleaned name as key
- `fact_sales_order_header[customer_id]` -> `dim_customer[customer_id]` many-to-one
- `fact_sales_order_header[order_date_key]` -> `dim_date[date_key]` many-to-one
- `fact_sales_order_header[sales_person_clean]` -> `dim_sales[sales_person_clean]` many-to-one if `dim_sales` uses the cleaned name as key
- Role-playing address relationships:
  - Active: `fact_sales_order_header[ship_to_address_id]` -> `dim_address[address_id]`
  - Inactive: `fact_sales_order_header[bill_to_address_id]` -> `dim_address[address_id]`
  - Same pattern for `fact_sales_order_line`
- Optional inactive relationships:
  - `due_date_key` -> `dim_date[date_key]`
  - `ship_date_key` -> `dim_date[date_key]`

Modeling notes:
- Keep `dim_sales` as the authoritative salesperson dimension for reporting by salesperson after the cleanup rule.
- `dim_customer` must remain one row per `customer_id`; do not model full customer-address multiplicity there.
- Preferred semantic-model handling for customer-address enrichment:
  - Keep `dim_customer` at one row per `customer_id`
  - Use `dim_address` plus fact ship/bill relationships for geography
  - Expose `bridge_customer_address` as hidden helper only if needed for advanced customer-address analysis
- Hide technical audit columns and source lineage columns.
- Hide repeated header amount columns on `fact_sales_order_line` if present.
- Prefer `fact_sales_order_header` for financial totals and order KPIs.
- If multiple product descriptions remain across cultures, either filter gold to one preferred culture or keep `culture` visible and require user selection.

Explicit measures:
- `Total Sales = SUM(fact_sales_order_header[total_due])`
- `Subtotal Sales = SUM(fact_sales_order_header[sub_total])`
- `Total Tax = SUM(fact_sales_order_header[tax_amt])`
- `Total Freight = SUM(fact_sales_order_header[freight])`
- `Order Count = DISTINCTCOUNT(fact_sales_order_header[sales_order_id])`
- `Units Sold = SUM(fact_sales_order_line[order_qty])`
- `Line Revenue = SUM(fact_sales_order_line[line_total])`
- `Average Order Value = DIVIDE([Total Sales], [Order Count])`
- `Average Selling Price = DIVIDE([Line Revenue], [Units Sold])`
- `Distinct Customers = DISTINCTCOUNT(fact_sales_order_header[customer_id])`
- `Sales per Customer = DIVIDE([Total Sales], [Distinct Customers])`
- `Discount Amount = SUMX(fact_sales_order_line, fact_sales_order_line[unit_price] * fact_sales_order_line[order_qty] * fact_sales_order_line[unit_price_discount])`

Additional measures aligned to the edited gold:
- `Sales Orders by Salesperson = DISTINCTCOUNT(fact_sales_order_header[sales_order_id])`
- `Total Sales by Salesperson = [Total Sales]`
- `Products Sold = DISTINCTCOUNT(fact_sales_order_line[product_id])`

Recommended hierarchies:
- Date hierarchy: Year > Quarter > Month > Date
- Product hierarchy: Parent Category > Category > Product Name
- Geography hierarchy: Country Region > State Province > City > Postal Code
- Sales hierarchy: Sales Person

## Report
Create a Power BI report on the Direct Lake semantic model with 3 pages.

### Page 1: Sales Overview
Purpose: executive summary of order, revenue, and salesperson performance.
Visuals:
- KPI cards:
  - `Total Sales`
  - `Order Count`
  - `Average Order Value`
  - `Distinct Customers`
- Line chart:
  - `Total Sales` by `dim_date[month_name]` and `dim_date[full_date]`
- Clustered column chart:
  - `Line Revenue` by `dim_product[parent_category_name]` then `dim_product[category_name]`
- Bar chart:
  - `Total Sales` by `dim_sales[sales_person_clean]`
- Donut chart:
  - `Total Sales` by `fact_sales_order_header[online_order_flag]`
- Matrix:
  - Rows: `dim_sales[sales_person_clean]`
  - Values: `Total Sales`, `Order Count`, `Average Order Value`
- Slicers:
  - Date
  - Parent Category
  - Category
  - Salesperson

### Page 2: Product and Customer Analysis
Purpose: analyze product hierarchy, customer detail, and customer-address context.
Visuals:
- Bar chart:
  - Top 10 products by `Line Revenue`
  - Axis: `dim_product[product_name]`
- Treemap:
  - Group: `dim_product[parent_category_name]`
  - Details: `dim_product[category_name]`, `dim_product[product_name]`
  - Values: `Line Revenue`
- Scatter chart:
  - X = `Average Selling Price`
  - Y = `Units Sold`
  - Details = `dim_product[product_name]`
  - Legend = `dim_product[category_name]`
- Table:
  - `dim_product[product_name]`
  - `dim_product[product_number]`
  - `dim_product[parent_category_name]`
  - `dim_product[category_name]`
  - `dim_product[product_model_name]`
  - `dim_product[color]`
  - `dim_product[list_price]`
  - `dim_product[standard_cost]`
  - `Units Sold`
  - `Line Revenue`
- Customer table or matrix:
  - `dim_customer[customer_id]`
  - `dim_customer[full_name]`
  - `dim_customer[company_name]`
  - `dim_customer[address_type]` if available
  - `dim_customer[city]`
  - `dim_customer[state_province]`
  - `dim_customer[country_region]`
  - `Total Sales`
  - `Order Count`
- Map or filled map:
  - Location from either:
    - `dim_customer[city]`, `dim_customer[state_province]`, `dim_customer[country_region]` if customer-address fields are exposed cleanly, or
    - `dim_address[city]`, `dim_address[state_province]`, `dim_address[country_region]` via shipping address relationship
  - Size/color by `Total Sales`
- Slicers:
  - Parent Category
  - Category
  - Product Model
  - Country Region
  - State Province

### Page 3: Salesperson and Data Quality
Purpose: highlight cleaned salesperson analytics and pipeline reliability.
Visuals:
- Clustered column chart:
  - `Total Sales` by `dim_sales[sales_person_clean]`
- Bar chart:
  - `Order Count` by `dim_sales[sales_person_clean]`
- Matrix:
  - Rows: `dim_sales[sales_person_clean]`
  - Columns: `dim_product[parent_category_name]`
  - Values: `Total Sales`, `Units Sold`
- Cards:
  - count of DQ test failures
  - latest successful run timestamp
  - bronze/silver/gold row counts for key tables
- Table:
  - test name, layer, table, status, failure_count, run_id, test_timestamp
- Column chart:
  - duplicates removed by table in silver
- Matrix:
  - referential integrity failures by relationship
- Trend line:
  - row counts by run for `sales_order_header` and `sales_order_detail`

Report notes:
- Prioritize the cleaned salesperson from `dim_sales[sales_person_clean]` everywhere instead of the raw source salesperson text.
- Prioritize the product hierarchy `parent_category_name` -> `category_name` -> `product_name`.
- Because `dim_customer` is constrained to one row per `customer_id`, it is safe for standard customer visuals; use `bridge_customer_address` only for advanced multi-address analysis.
- For geography visuals, prefer shipping geography through `dim_address` if customer-address joins introduce ambiguity.
- If multilingual product descriptions are retained, add a culture slicer only when needed.
- If only one address relationship is exposed initially, prioritize shipping address.

## Data Agent
AISkill grounded on the semantic model.

Role:
- You are a sales analytics copilot for a transactional order dataset.
- Help users understand sales performance, customer purchasing, product mix, product hierarchy, salesperson performance, shipping patterns, and data quality status using the published Direct Lake semantic model.
- Prefer order-level totals from `fact_sales_order_header` for financial totals and line-level metrics from `fact_sales_order_line` for units/product analysis.
- Use the cleaned salesperson dimension from `dim_sales`, not the raw `sales_person` string from the source.

Domain hints:
- Customers are identified by `customer_id`.
- `dim_customer` is enriched from customer, customer-address, and address data, but is constrained to one visible row per customer; use address attributes there as the selected representative address only.
- Orders come from `fact_sales_order_header`; order lines come from `fact_sales_order_line`.
- Products are analyzed through `dim_product`, which combines product, product category hierarchy, product model, and description data.
- Product hierarchy should use:
  - `parent_category_name` as the higher level
  - `category_name` as the higher-level category label used for reporting
  - `subcategory_name` as the lower category/subcategory level
  - `product_name` at the product level
- Salespeople are analyzed through `dim_sales`, where values were cleaned from `sales_order_header.sales_person` by removing the `adventureworks/` prefix and the trailing number.
- Address analysis should distinguish shipping vs billing when the model exposes both through `dim_address`.
- Sensitive source fields such as password-related columns are excluded from analytics and must never be referenced.

Starter questions:
- What are total sales, order count, and average order value by month?
- Which parent categories and categories generate the most line revenue?
- What are the top 10 products by units sold and by revenue?
- Which cleaned salespeople have the highest total sales and order counts?
- Which customers have the highest total sales?
- What is the sales distribution by customer city, state, and country?
- How do online orders compare with non-online orders?
- Which ship methods are associated with the highest sales?
- Are there orders with unusually high discounts?
- What data quality issues were detected in the latest pipeline run?

Guardrails:
- Do not answer using columns or tables outside the semantic model.
- Do not expose or infer hidden sensitive fields such as password hash or salt.
- Use `fact_sales_order_header` measures for order financial totals to avoid double counting.
- Use the cleaned salesperson from `dim_sales`; do not present raw values with the `adventureworks/` prefix or trailing numbers.
- When users ask for category performance, use the hierarchy `parent_category_name`, `category_name`, and `subcategory_name` correctly.
- If a question is ambiguous about date or geography role, ask whether to use order date vs ship date, or shipping vs billing address.
- If customer geography can come from either enriched customer-address fields or shipping/billing address relationships, clarify which one the user wants when needed.
- If multilingual product descriptions exist, clarify which culture to analyze when relevant.
- If a requested metric is not modeled, say so explicitly and suggest the nearest available measure.
- Surface data quality caveats when latest DQ tests show failures.
- Avoid unsupported causal claims; describe observed patterns only.