# Run Spec 20260517-144043-a594f6

## Updated specs

### Iteration 1 — 2026-05-17 14:52:28Z — failed layer: gold (run: 20260517-144627-02c2b7)
- **Root cause (1-line summary)**: Gold build likely failed due to under-specified joins/column names in the requested custom Gold tables (`dim_customer`, `dim_product`, `dim_sales`, `fact_sales`), especially around self-join/category hierarchy and flattened customer-address logic.
- **What was changed**:
  - Tightened `## Gold` to define the exact required Gold tables, their grains, deterministic flattening rules, join paths, aliases, and output column names.
  - Added explicit rules for the product category self-join and english-only product description selection, plus cleaned salesperson derivation logic.
  - Tightened `## Semantic model` to align strictly to the four requested Gold tables and their exact join keys/field names.

### Iteration 2 — 2026-05-17 15:03:15Z — failed layer: gold (run: 20260517-144627-02c2b7)
- **Root cause (1-line summary)**: Gold layer remains vulnerable to unresolved/incorrect column references because several required Gold fields were specified optimistically rather than constrained to validated SalesLT Silver columns.
- **What was changed**:
  - Tightened `## Generic guidance` with a new rule to validate actual source/Silver schemas before coding Gold logic and to avoid assuming optional SalesLT columns exist.
  - Tightened `## Gold` with explicit fallback/derivation rules for salesperson fields and stricter column contracts for `sales_order_header`, `customer`, and `product_model`.
  - Added mandatory implementation guidance to build `dim_sales` and `fact_sales` from a shared cleaned-salesperson expression and to use only explicitly listed output columns.

### Iteration 3 — 2026-05-17 15:13:41Z — failed layer: gold (run: 20260517-144627-02c2b7)
- **Root cause (1-line summary)**: UNRESOLVED_COLUMN on `c.customer_id` in `gold.dim_customer` windowing because the row-number ranking was applied after alias-qualified columns had already been projected/renamed away.
- **What was changed**:
  - Tightened `## Generic guidance` to forbid referencing source aliases after a projection that removes those aliases, especially in window specs, filters, and later transforms.
  - Tightened `## Gold` / `gold.dim_customer` with a mandatory two-step build pattern: rank on alias-qualified join output first, then project final renamed columns after filtering to rank 1.
  - Added an explicit window-column contract for `dim_customer` using only `c.customer_id`, `ca.address_type`, and `ca.address_id` before the final select.

### Iteration 1 — 2026-05-17 15:16:19Z — failed layer: gold (run: 20260517-144627-02c2b7)
- **Root cause (1-line summary)**: Gold run still failed at session level because Gold transformations remain under-constrained for execution safety; the build must stage and validate each Gold table independently before continuing.
- **What was changed**:
  - Tightened `## Generic guidance` to require fail-fast, table-by-table Gold validation and to stop immediately on the first Gold table error instead of continuing until session cancellation.
  - Tightened `## Gold` with a mandatory build order, pre-write validation checks, and explicit row-level uniqueness/null-key assertions for each of the four required Gold tables.
  - Added an explicit implementation rule that each Gold table must be materialized from its own final DataFrame and validated before starting the next Gold table.

### Iteration 2 — 2026-05-17 15:22:53Z — failed layer: gold (run: 20260517-144627-02c2b7)
- **Root cause (1-line summary)**: Gold execution still reached session cancellation because the spec did not force per-table schema assertions and immediate fail-fast checkpoints before and after each Gold transformation step.
- **What was changed**:
  - Tightened `## Generic guidance` to require explicit schema/row-count checkpoints after each intermediate Gold step and immediate notebook termination on the first failed assertion.
  - Tightened `## Gold` with mandatory per-table preflight checks, intermediate step names, and write verification for `dim_customer`, `dim_product`, `dim_sales`, and `fact_sales`.
  - Added a stricter rule that no later Gold table may start until the prior table has passed source-column validation, final-column validation, key validation, and post-write readability verification.

### Iteration 3 — 2026-05-17 15:30:12Z — failed layer: silver (run: 20260517-144627-02c2b7)
- **Root cause (1-line summary)**: Silver session was cancelled after statement failures because Silver transformations were still under-specified on exact retained columns, dedup ordering, and per-table fail-fast validation.
- **What was changed**:
  - Tightened `## Generic guidance` with the same blocking, per-table validation discipline already required in Gold, now explicitly applied to Silver.
  - Tightened `## Silver` with an exact required build order, per-table source/target column contracts, deterministic dedup ordering rules, and post-write readback validation.
  - Added an explicit rule to build Silver final DataFrames with exactly the declared output columns only, and to stop immediately on the first Silver table failure.

## Inputs
- Workspace: `d97c0ec7-74ac-4f12-9e70-124cb0d5a4ad`
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
- Target Lakehouse: **CopilotMedallion_20260517_144043**

## Generic guidance
Apply these reference skills/agents at all times:
- FabricDataEngineer agent: https://github.com/microsoft/skills-for-fabric/blob/main/agents/FabricDataEngineer.agent.md
- e2e-medallion-architecture skill: https://github.com/microsoft/skills-for-fabric/tree/main/skills/e2e-medallion-architecture
- spark-authoring-cli skill: https://github.com/microsoft/skills-for-fabric/tree/main/skills/spark-authoring-cli
- powerbi-authoring-cli skill: https://github.com/microsoft/skills-for-fabric/tree/main/skills/powerbi-authoring-cli
- powerbi-consumption-cli skill: https://github.com/microsoft/skills-for-fabric/tree/main/skills/powerbi-consumption-cli
- powerbi-semantic-model-authoring: https://github.com/RuiRomano/powerbi-agentic-plugins/tree/main/plugins/powerbi/skills/powerbi-semantic-model-authoring
- powerbi-report-authoring: https://github.com/RuiRomano/powerbi-agentic-plugins/tree/main/plugins/powerbi-report-authoring

Cross-cutting code rules:
- Use defensive column references everywhere; assert required columns exist before select, join, dedup, rename, filter, aggregate, and write.
- After every join, immediately project to an explicit alias-prefixed/select-renamed column list to prevent ambiguous names from propagating, unless a downstream window/filter in the same logical build step still needs alias-qualified columns; in that case, keep the needed alias-qualified columns through the ranking/filter step and only then perform the final renamed projection.
- Before every `groupBy`/`agg`, assert all grouping and aggregation input columns exist.
- For REST/API responses, use defensive handling and raise early: `if x is None: raise ...` before any `.get()` access.
- Never use `saveAsTable`; write using idempotent Delta overwrite/merge patterns against paths or managed table APIs supported by Fabric notebooks.
- Put notebook parameters in dedicated parameter cells at the top of each notebook.
- Use idempotent overwrite patterns for full loads and deterministic merge/upsert patterns where incremental logic is introduced.
- All `try/except` blocks must fail loudly: call `_save_error(layer, e)` and then `raise`.
- Add audit/control columns consistently across layers, including load timestamp, run id, source system/table, and record hash where useful.
- Optimize physical layout after writes using partition-aware maintenance only where justified by the actual data shape; avoid unnecessary partitioning on small dimensions.
- Treat source views (`vGetAllCategories`, `vProductAndDescription`, `vProductModelCatalogDescription`) as read-only source entities; do not assume they are canonical over base tables unless explicitly chosen in Gold.
- In Gold, do not infer column names from business descriptions. Use only validated Silver column names after `snake_case` conversion and fail early if an expected column is absent.
- In Gold self-joins and bridge-based enrichments, every input DataFrame must be aliased and every selected output column must be explicitly renamed to its final Gold name in the same projection step.
- Before implementing any Gold table, inspect the actual Silver schema and reconcile it to this spec. If a field in the business request is not a validated Silver column, derive it only from the explicit fallback rules in this spec; otherwise fail early with a clear missing-column error instead of guessing an alternative column name.
- Never reference an alias-qualified column name after a projection/select step that has removed or renamed that alias-qualified column. This applies especially to `Window.partitionBy`, `Window.orderBy`, `filter`, `withColumn`, and `dropDuplicates`. If a later step needs `c.customer_id`, `ca.address_type`, or similar alias-qualified fields, perform that later step before the final renaming/projection.
- In Gold, build and validate one table at a time in this exact order: `gold.dim_customer`, then `gold.dim_product`, then `gold.dim_sales`, then `gold.fact_sales`. Do not start the next Gold table until the current one has passed schema, null-key, and uniqueness checks and has been written successfully.
- For each Gold table, create a dedicated final DataFrame variable containing exactly the persisted output columns for that table. Validate that final DataFrame before write, write it, then clear/release intermediate DataFrames before building the next Gold table.
- If any Gold table fails validation or write, stop the notebook immediately after `_save_error('gold', e)` and `raise`; do not continue attempting later Gold tables, because that can lead to session-wide statement failure/cancellation.
- In Gold, add explicit fail-fast checkpoints after each intermediate build step: source read, join result, ranking/dedup result, final projection, pre-write validation, and post-write readback. Each checkpoint must assert expected columns exist; for ranking/dedup/final steps it must also assert the DataFrame is instantiable and countable without error.
- Gold notebook control flow must be strictly linear and blocking: build one table, validate it, write it, read it back once to verify schema/readability, then and only then proceed to the next table. Never queue or lazily defer multiple Gold actions before validating the current table.
- Apply the same fail-fast discipline to Silver: build and validate one Silver table at a time, write it, read it back, and only then continue to the next Silver table.
- If any Silver table fails validation or write, stop the notebook immediately after `_save_error('silver', e)` and `raise`; do not continue attempting later Silver tables.
- For every Silver table, validate source Bronze existence, required source columns, intermediate DataFrame schema, final DataFrame exact column list, non-null key columns, uniqueness at the declared grain, and post-write readability.
- In Silver, do not keep pass-through columns by implication. Each Silver table must be written from a dedicated final DataFrame containing exactly the columns explicitly listed in this spec for that table, plus the required Silver audit columns.

Required notebook cell commenting standard:
- EACH code cell must start with a short markdown-style Python comment block.
- Use a `# ---` divider followed by 1-3 human-readable `# ` comment lines describing what the cell is doing and why.
- Never emit a code cell without this leading comment block.
- Example:
  - `# ---`
  - `# Read raw bronze table and apply schema enforcement.`
  - `# Drops rows where any required key is null.`

## Bronze
Land each selected source object into the `bronze` schema as 1:1 raw Delta tables with minimal transformation and consistent metadata.

### Bronze write pattern
- Load type: full snapshot ingestion for all listed tables/views unless the user later edits to introduce incremental logic based on `ModifiedDate`.
- Write mode: idempotent overwrite per table for each run.
- Table naming: `bronze.address`, `bronze.customer`, `bronze.customer_address`, `bronze.product`, `bronze.product_category`, `bronze.product_description`, `bronze.product_model`, `bronze.product_model_product_description`, `bronze.sales_order_detail`, `bronze.sales_order_header`, `bronze.v_get_all_categories`, `bronze.v_product_and_description`, `bronze.v_product_model_catalog_description`.
- Metadata columns to append to every bronze table:
  - `ingested_at_utc`
  - `run_id`
  - `source_workspace_id`
  - `source_lakehouse_id`
  - `source_lakehouse_name`
  - `source_object_name`
  - `source_record_hash` from all business columns
- Partitioning:
  - `bronze.sales_order_header`: partition by `order_year` derived from `OrderDate`
  - `bronze.sales_order_detail`: no partition initially unless volume proves large; detail lacks a natural date without joining header
  - All other tables/views: no partitioning due to likely small/master/reference size
- Schema drift handling:
  - Enforce expected schema from provided column lists
  - Log and fail on missing required source columns
  - Optionally allow additive columns only if user edits the spec to support schema evolution

### Table-specific landing notes
- `SalesLT/Address`
  - Raw primary key candidate: `AddressID`
  - Preserve address text exactly; no trimming/standardization in Bronze
- `SalesLT/Customer`
  - Raw primary key candidate: `CustomerID`
  - Keep sensitive columns `PasswordHash` and `PasswordSalt` in Bronze only
- `SalesLT/CustomerAddress`
  - Junction table between customer and address
  - Composite key candidate: `CustomerID`, `AddressID`, `AddressType`
- `SalesLT/Product`
  - Raw primary key candidate: `ProductID`
  - Preserve binary `ThumbNailPhoto` in Bronze; likely exclude from Silver/Gold analytics
- `SalesLT/ProductCategory`
  - Raw primary key candidate: `ProductCategoryID`
  - Contains parent-child hierarchy via `ParentProductCategoryID`
- `SalesLT/ProductDescription`
  - Raw primary key candidate: `ProductDescriptionID`
- `SalesLT/ProductModel`
  - Raw primary key candidate: `ProductModelID`
- `SalesLT/ProductModelProductDescription`
  - Bridge table among product model, description, and culture
  - Composite key candidate: `ProductModelID`, `ProductDescriptionID`, `Culture`
- `SalesLT/SalesOrderDetail`
  - Transaction line fact candidate
  - Composite key candidate: `SalesOrderID`, `SalesOrderDetailID`
- `SalesLT/SalesOrderHeader`
  - Order header fact candidate
  - Raw primary key candidate: `SalesOrderID`
- `SalesLT/vGetAllCategories`
  - Reference view already flattening category hierarchy
  - Key candidate: `ProductCategoryID`
- `SalesLT/vProductAndDescription`
  - Enriched product text view by `ProductID`, `Culture`
- `SalesLT/vProductModelCatalogDescription`
  - Wide descriptive view by `ProductModelID`
  - Treat as analytic enrichment source, not transactional source

## Silver
Standardize names, types, deduplicate, remove clearly non-analytic sensitive/binary fields where appropriate, and produce conformed clean tables in the `silver` schema.

### Silver common rules
- Rename all columns to `snake_case`.
- Add audit columns:
  - `silver_loaded_at_utc`
  - `run_id`
  - `record_hash`
- Standardize string blanks to null where appropriate for descriptive columns.
- Preserve source keys as-is; do not generate surrogate keys in Silver.
- Deduplicate using the stated business keys, keeping the latest by `modified_date` where available.
- Apply `OPTIMIZE` after writes for large tables:
  - `silver.sales_order_header`
  - `silver.sales_order_detail`
- Exclude highly sensitive/non-analytic fields from downstream conformed analytics:
  - Drop `password_hash`, `password_salt` from `silver.customer`
  - Drop `thumbnail_photo` from `silver.product` unless the user wants image reporting scenarios
- Build and validate Silver tables one at a time in this exact order:
  1. `silver.address`
  2. `silver.customer`
  3. `silver.customer_address`
  4. `silver.product`
  5. `silver.product_category`
  6. `silver.product_description`
  7. `silver.product_model`
  8. `silver.product_model_product_description`
  9. `silver.sales_order_header`
  10. `silver.sales_order_detail`
  11. `silver.v_get_all_categories`
  12. `silver.v_product_and_description`
  13. `silver.v_product_model_catalog_description`
- For each Silver table:
  - validate source Bronze table existence before reading
  - validate required Bronze source columns after `snake_case` normalization and before any transformation
  - create a dedicated `<table>_final_df` containing exactly the specified retained/derived columns plus `silver_loaded_at_utc`, `run_id`, `record_hash`
  - validate the final DataFrame exact column list before write
  - validate key non-nullness and uniqueness at the declared grain before write
  - write the table
  - read the written table back and verify schema/readability before proceeding
- If a Silver source table lacks `modified_date`, deduplicate deterministically by the declared dedup key using a stable fallback order:
  1. non-null dedup key columns first
  2. then ascending dedup key columns
  3. then ascending `rowguid` if present
  4. else no additional business-column guessing; keep the first row from the stable ordered window
- If a Silver source table has `modified_date`, the dedup window must order by:
  1. `modified_date` descending nulls last
  2. `rowguid` descending nulls last when present
  3. dedup key columns ascending
- Do not apply `dropDuplicates()` blindly on whole rows for Silver conformance tables. Use the declared dedup key and deterministic windowed row selection.

### Silver table specifications
- `silver.address`
  - Source: `bronze.address`
  - Key: `address_id`
  - Dedup key: `address_id`
  - Required source columns after snake_case: `address_id`, `address_line1`, `address_line2`, `city`, `state_province`, `country_region`, `postal_code`, `rowguid`, `modified_date`
  - Final output columns must be exactly:
    - `address_id`
    - `address_line1`
    - `address_line2`
    - `city`
    - `state_province`
    - `country_region`
    - `postal_code`
    - `rowguid`
    - `modified_date`
    - `silver_loaded_at_utc`
    - `run_id`
    - `record_hash`
- `silver.customer`
  - Source: `bronze.customer`
  - Key: `customer_id`
  - Dedup key: `customer_id`
  - Required source columns after snake_case: `customer_id`, `name_style`, `title`, `first_name`, `middle_name`, `last_name`, `suffix`, `company_name`, `sales_person`, `email_address`, `phone`, `rowguid`, `modified_date`
  - Final output columns must be exactly:
    - `customer_id`
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
    - `rowguid`
    - `modified_date`
    - `full_name`
    - `customer_name_preferred`
    - `silver_loaded_at_utc`
    - `run_id`
    - `record_hash`
  - Add derived columns:
    - `full_name` from name parts
    - `customer_name_preferred` choosing `company_name` when present, otherwise `full_name`
  - `full_name` derivation contract:
    - concatenate `title`, `first_name`, `middle_name`, `last_name`, `suffix` in that logical order using single spaces between non-null non-blank parts
    - trim final result
    - if final result is empty, set to null
- `silver.customer_address`
  - Source: `bronze.customer_address`
  - Dedup key: `customer_id`, `address_id`, `address_type`
  - Required source columns after snake_case: `customer_id`, `address_id`, `address_type`, `rowguid`, `modified_date`
  - Final output columns must be exactly:
    - `customer_id`
    - `address_id`
    - `address_type`
    - `rowguid`
    - `modified_date`
    - `silver_loaded_at_utc`
    - `run_id`
    - `record_hash`
  - Retain junction structure for later billing/shipping role-play logic
- `silver.product`
  - Source: `bronze.product`
  - Key: `product_id`
  - Dedup key: `product_id`
  - Required source columns after snake_case: `product_id`, `name`, `product_number`, `color`, `standard_cost`, `list_price`, `size`, `weight`, `product_category_id`, `product_model_id`, `sell_start_date`, `sell_end_date`, `discontinued_date`, `thumbnail_photo_file_name`, `rowguid`, `modified_date`
  - Final output columns must be exactly:
    - `product_id`
    - `name`
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
    - `thumbnail_photo_file_name`
    - `rowguid`
    - `modified_date`
    - `is_discontinued`
    - `is_sellable_currently`
    - `silver_loaded_at_utc`
    - `run_id`
    - `record_hash`
  - Add derived flags:
    - `is_discontinued`
    - `is_sellable_currently`
- `silver.product_category`
  - Source: `bronze.product_category`
  - Key: `product_category_id`
  - Dedup key: `product_category_id`
  - Required source columns after snake_case: `product_category_id`, `parent_product_category_id`, `name`, `rowguid`, `modified_date`
  - Final output columns must be exactly:
    - `product_category_id`
    - `parent_product_category_id`
    - `name`
    - `rowguid`
    - `modified_date`
    - `silver_loaded_at_utc`
    - `run_id`
    - `record_hash`
- `silver.product_description`
  - Source: `bronze.product_description`
  - Key: `product_description_id`
  - Dedup key: `product_description_id`
  - Required source columns after snake_case: `product_description_id`, `description`, `rowguid`, `modified_date`
  - Final output columns must be exactly:
    - `product_description_id`
    - `description`
    - `rowguid`
    - `modified_date`
    - `silver_loaded_at_utc`
    - `run_id`
    - `record_hash`
- `silver.product_model`
  - Source: `bronze.product_model`
  - Key: `product_model_id`
  - Dedup key: `product_model_id`
  - Required source columns after snake_case: `product_model_id`, `name`, `rowguid`, `modified_date`
  - Final output columns must be exactly:
    - `product_model_id`
    - `name`
    - `rowguid`
    - `modified_date`
    - `silver_loaded_at_utc`
    - `run_id`
    - `record_hash`
- `silver.product_model_product_description`
  - Source: `bronze.product_model_product_description`
  - Dedup key: `product_model_id`, `product_description_id`, `culture`
  - Required source columns after snake_case: `product_model_id`, `product_description_id`, `culture`, `rowguid`, `modified_date`
  - Final output columns must be exactly:
    - `product_model_id`
    - `product_description_id`
    - `culture`
    - `rowguid`
    - `modified_date`
    - `silver_loaded_at_utc`
    - `run_id`
    - `record_hash`
- `silver.sales_order_header`
  - Source: `bronze.sales_order_header`
  - Key: `sales_order_id`
  - Dedup key: `sales_order_id`
  - Required source columns after snake_case: `sales_order_id`, `sales_order_number`, `purchase_order_number`, `account_number`, `customer_id`, `sales_person`, `bill_to_address_id`, `ship_to_address_id`, `order_date`, `due_date`, `ship_date`, `status`, `online_order_flag`, `ship_method`, `sub_total`, `tax_amt`, `freight`, `total_due`, `rowguid`, `modified_date`
  - Final output columns must be exactly:
    - `sales_order_id`
    - `sales_order_number`
    - `purchase_order_number`
    - `account_number`
    - `customer_id`
    - `sales_person`
    - `bill_to_address_id`
    - `ship_to_address_id`
    - `order_date`
    - `due_date`
    - `ship_date`
    - `status`
    - `online_order_flag`
    - `ship_method`
    - `sub_total`
    - `tax_amt`
    - `freight`
    - `total_due`
    - `rowguid`
    - `modified_date`
    - `order_date_key`
    - `due_date_key`
    - `ship_date_key`
    - `order_year`
    - `order_month`
    - `order_status_desc`
    - `silver_loaded_at_utc`
    - `run_id`
    - `record_hash`
  - Retain transactional columns and add:
    - `order_date_key` as `yyyyMMdd` integer from `order_date`
    - `due_date_key` as `yyyyMMdd` integer from `due_date`
    - `ship_date_key` as `yyyyMMdd` integer from `ship_date` when not null
    - `order_year`, `order_month`, `order_status_desc` from `status`
- `silver.sales_order_detail`
  - Source: `bronze.sales_order_detail`
  - Key: `sales_order_id`, `sales_order_detail_id`
  - Dedup key: `sales_order_id`, `sales_order_detail_id`
  - Required source columns after snake_case: `sales_order_id`, `sales_order_detail_id`, `order_qty`, `product_id`, `unit_price`, `unit_price_discount`, `line_total`, `rowguid`, `modified_date`
  - Final output columns must be exactly:
    - `sales_order_id`
    - `sales_order_detail_id`
    - `order_qty`
    - `product_id`
    - `unit_price`
    - `unit_price_discount`
    - `line_total`
    - `rowguid`
    - `modified_date`
    - `gross_line_amount`
    - `discount_amount`
    - `net_line_amount`
    - `silver_loaded_at_utc`
    - `run_id`
    - `record_hash`
  - Add derived columns:
    - `gross_line_amount = order_qty * unit_price`
    - `discount_amount = order_qty * unit_price * unit_price_discount`
    - `net_line_amount = line_total`
- `silver.v_get_all_categories`
  - Source: `bronze.v_get_all_categories`
  - Dedup key: `product_category_id`
  - Required source columns after snake_case: validate actual Bronze schema first, then require at minimum `product_category_id`
  - Final output columns:
    - retain only validated source columns from the view after snake_case normalization
    - plus `silver_loaded_at_utc`, `run_id`, `record_hash`
  - Use as optional hierarchy helper in Gold
- `silver.v_product_and_description`
  - Source: `bronze.v_product_and_description`
  - Dedup key: `product_id`, `culture`
  - Required source columns after snake_case: validate actual Bronze schema first, then require at minimum `product_id`, `culture`
  - Final output columns:
    - retain only validated source columns from the view after snake_case normalization
    - plus `silver_loaded_at_utc`, `run_id`, `record_hash`
  - Optional multilingual product description helper
- `silver.v_product_model_catalog_description`
  - Source: `bronze.v_product_model_catalog_description`
  - Dedup key: `product_model_id`
  - Required source columns after snake_case: validate actual Bronze schema first, then require at minimum `product_model_id`
  - Final output columns:
    - retain only validated source columns from the view after snake_case normalization
    - plus `silver_loaded_at_utc`, `run_id`, `record_hash`
  - Optional wide model attribute helper

### Silver modeling note
Two valid enrichment routes exist for product/category text:
- Route A: build Gold mainly from base tables plus bridges, using source views only as helper enrichments.
- Route B: use `vGetAllCategories` and `vProductAndDescription` as preferred descriptive sources because they already flatten logic.
- Proposed default: Route A for lineage clarity; user may edit to Route B for simpler Gold logic.

## Gold
Model a sales analytics star schema in `gold`, using actual transaction, master, and bridge tables from the provided schema.
I want to end up with the following tables:
Dim_Customer -> join customer, customer address, address and leave all relevant/meaningfull fields
Dim_Product -> join product, productcategory, producdescription, productmodel, productmodelproductdescription. Please notice that in productcategory there is a self join. So in the end I have the category (which is the parent category) and a subcategory.
Dim_Sales --> Use the salesperson field from salesorderheader, but remove the adventureworks/ in front of the name and remove the last number at the end off the name
Fact_Sales --> join salesorderdetail and salesorderheader and keep all relevant fields
Use below for more insights, but what I just described is your end goal in gold

### Gold implementation rules for this run
Build only these four Gold tables for the final curated layer and semantic model:
- `gold.dim_customer`
- `gold.dim_product`
- `gold.dim_sales`
- `gold.fact_sales`

Do not materialize alternative Gold tables such as `gold.dim_address`, `gold.dim_date`, `gold.dim_ship_method`, `gold.bridge_customer_address`, `gold.fact_sales_order_line`, or `gold.fact_sales_order_header` in place of the four requested outputs. They may be used as temporary DataFrames during notebook execution, but the persisted Gold outputs for this run must be exactly the four requested tables above.

### Gold source table and column contract
Use these Silver tables and only these validated key columns for Gold joins:
- `silver.customer` as `c`
  - required columns: `customer_id`, `customer_name_preferred`, `full_name`, `company_name`, `email_address`, `phone`, `name_style`, `modified_date`
  - optional column: `sales_person`
  - if `c.sales_person` is absent, `gold.dim_customer` must still build successfully and emit `sales_person` as null
- `silver.customer_address` as `ca`
  - required columns: `customer_id`, `address_id`, `address_type`
- `silver.address` as `a`
  - required columns: `address_id`, `address_line1`, `address_line2`, `city`, `state_province`, `country_region`, `postal_code`
- `silver.product` as `p`
  - required columns: `product_id`, `name`, `product_number`, `color`, `size`, `weight`, `standard_cost`, `list_price`, `product_category_id`, `product_model_id`, `is_discontinued`, `is_sellable_currently`
- `silver.product_category` as `pc_child`
  - required columns: `product_category_id`, `parent_product_category_id`, `name`
- `silver.product_category` as `pc_parent`
  - required columns: `product_category_id`, `name`
- `silver.product_model` as `pm`
  - required columns: `product_model_id`
  - optional column: `name`
  - if `pm.name` is absent, `gold.dim_product` must still build successfully and emit `product_model_name` as null
- `silver.product_model_product_description` as `pmpd`
  - required columns: `product_model_id`, `product_description_id`, `culture`
- `silver.product_description` as `pd`
  - required columns: `product_description_id`, `description`
- `silver.sales_order_header` as `h`
  - required columns: `sales_order_id`, `sales_order_number`, `purchase_order_number`, `account_number`, `customer_id`, `bill_to_address_id`, `ship_to_address_id`, `order_date`, `due_date`, `ship_date`, `status`, `order_status_desc`, `online_order_flag`, `ship_method`, `sub_total`, `tax_amt`, `freight`, `total_due`
  - salesperson source contract:
    - preferred required column for salesperson derivation: `sales_person`
    - if `h.sales_person` does not exist after schema validation, use `salesperson` only if that exact snake_case/renamed column is present in Silver
    - do not guess any other header column name for salesperson
    - once resolved, immediately normalize it to a single temporary column named `sales_person_source` before any further select/join logic
- `silver.sales_order_detail` as `d`
  - required columns: `sales_order_id`, `sales_order_detail_id`, `product_id`, `order_qty`, `unit_price`, `unit_price_discount`, `gross_line_amount`, `discount_amount`, `net_line_amount`

If any required column above is missing, fail immediately with a clear error naming the table alias and column. For optional columns, emit the required Gold output column as null rather than failing.

### Shared salesperson derivation contract
- Build one reusable expression/temporary projection for cleaned salesperson values and reuse it identically in both `gold.dim_sales` and `gold.fact_sales`.
- Input to the cleaning expression must be the resolved header column `sales_person_source`.
- Cleaning logic for `sales_person_clean`:
  - cast source to string
  - remove leading `adventureworks/` case-insensitively
  - then remove trailing digits at the end of the string
  - then trim whitespace
  - if the cleaned result is empty string, set to null
- Do not reference raw `h.sales_person` directly in multiple places after resolution; resolve once to `sales_person_source`, then derive `sales_person_clean`.

### Gold build sequencing and validation contract
- Build the four Gold tables in this exact order:
  1. `gold.dim_customer`
  2. `gold.dim_product`
  3. `gold.dim_sales`
  4. `gold.fact_sales`
- For each table, before write:
  - validate the final DataFrame contains exactly the specified output columns and no extras
  - validate the required grain key columns are present and non-null
  - validate uniqueness at the declared grain
- For each table, after write:
  - immediately read back the persisted Gold table
  - verify the readback schema matches the specified output column list exactly
  - verify the readback is queryable with at least a lightweight action
- If any validation fails, stop immediately and do not continue to subsequent Gold tables.
- Do not chain all four Gold writes inside one broad `try` that masks which table failed; each table must have its own explicit validation-and-write block.
- Mandatory per-table execution checkpoints:
  1. validate source Silver table existence and required columns
  2. build each intermediate DataFrame step named in this spec
  3. after each intermediate step, assert required columns for that step exist before proceeding
  4. materialize/check the final DataFrame before write
  5. write the table
  6. read the table back and verify schema/readability
- No later Gold table may start until all six checkpoints above have succeeded for the current table.

### Global Spark column-reference rules (apply to all Gold tables)
These rules exist to prevent recurring `UNRESOLVED_COLUMN` analyzer errors.
- Never pass dotted strings like `"c.customer_id"`, `"ca.address_type"`, `"h.sales_person"`, or `"pc_child.name"` to `F.col(...)`, `withColumn(...)`, `Window.partitionBy(...)`, `Window.orderBy(...)`, or `select(...)`. Spark treats `"c.customer_id"` as a single column literally named `c.customer_id`, which does not resolve once any projection or rename has been applied.
- Alias scope (`.alias("c")`, `.alias("ca")`, ...) is only valid inside the SAME `select` / `join` expression that introduces it. Once you produce a new DataFrame via `select(...)` or `withColumn(...)`, the dotted alias form is gone and you must reference plain column names.
- For any column that will later be referenced by a `Window`, a `withColumn`, or a downstream join after a projection, first materialize it as a flat, unambiguous helper column (e.g. `rank_customer_id`, `rank_address_type`, `sales_person_source`) in the same select that introduces the join aliases.
- Before applying a window, `withColumn`, or final projection, do not drop columns that the next step still needs. Helper / source columns may only be dropped in the FINAL select that produces the Gold output schema.

### `gold.dim_customer`
- Grain: exactly one row per `customer_id`.
- Join path:
  - start from `silver.customer c`
  - left join `silver.customer_address ca` on `c.customer_id = ca.customer_id`
  - left join `silver.address a` on `ca.address_id = a.address_id`
- Mandatory implementation pattern for this table:
  - Step 1: build `dim_customer_joined_df` from the three-table join. In the SAME select that produces this DataFrame, create unambiguous, non-dotted ranking helper columns so the window never depends on alias-qualified names:
    - `rank_customer_id`   = `c.customer_id`
    - `rank_address_type`  = `ca.address_type`
    - `rank_address_id`    = `ca.address_id`
    Also keep every source attribute needed for the final projection (customer fields from `c`, address fields from `a`, and `ca.address_type` / `ca.address_id` for the final renamed output columns).
  - Step 2: assert `dim_customer_joined_df` contains the columns `rank_customer_id`, `rank_address_type`, `rank_address_id` AND every final-output source column. Fail with a clear message if any are missing.
  - Step 3: build `dim_customer_ranked_df` by adding `_rn = row_number().over(w)` where `w` is defined ONLY against the flat helper columns from Step 1 (see window contract below).
  - Step 4: assert `_rn`, `rank_customer_id`, `rank_address_type`, `rank_address_id` all exist on `dim_customer_ranked_df`.
  - Step 5: build `dim_customer_filtered_df = dim_customer_ranked_df.filter(F.col('_rn') == 1)`.
  - Step 6: build `dim_customer_final_df` by selecting/renaming to the final Gold column list. This is the FIRST point at which `rank_*` and `_rn` helper columns may be dropped.
- Deterministic flattening rule for customers with multiple addresses:
  - rank addresses per customer with this precedence:
    1. `address_type = 'Main Office'`
    2. `address_type = 'Primary'`
    3. `address_type = 'Billing'`
    4. `address_type = 'Home'`
    5. `address_type = 'Shipping'`
    6. any other non-null `address_type` alphabetically
    7. lowest `address_id`
  - keep only rank 1 per `customer_id`
- Window specification contract for `dim_customer`:
  - The window MUST be defined as:
    ```python
    w = (
        Window
        .partitionBy(F.col("rank_customer_id"))
        .orderBy(
            F.when(F.col("rank_address_type") == "Main Office", 1)
             .when(F.col("rank_address_type") == "Primary",     2)
             .when(F.col("rank_address_type") == "Billing",     3)
             .when(F.col("rank_address_type") == "Home",        4)
             .when(F.col("rank_address_type") == "Shipping",    5)
             .when(F.col("rank_address_type").isNotNull(),      6)
             .otherwise(7),
            F.col("rank_address_type").asc_nulls_last(),
            F.col("rank_address_id").asc_nulls_last(),
        )
    )
    ```
  - Hard prohibitions (any of these is an immediate fail, do not generate code that contains them):
    - Do NOT pass dotted strings like `"c.customer_id"`, `"ca.address_type"`, or `"ca.address_id"` to `Window.partitionBy`, `Window.orderBy`, `F.col`, or `withColumn`. Spark interprets `"c.customer_id"` as a single column literally named `c.customer_id` and it will not resolve after a projection.
    - Do NOT reference `c.*`, `ca.*`, or `a.*` alias-qualified columns AFTER the Step 1 select that produced `dim_customer_joined_df`. The alias scope ends at that select.
    - Do NOT define the window AFTER renaming columns to their final Gold names (e.g. against `customer_address_type` / `customer_address_id`). The window must be defined against the `rank_*` helper columns from Step 1.
    - Do NOT use `.alias("c")` / `.alias("ca")` and then expect `F.col("c.customer_id")` to work outside a `selectExpr` / SQL string context.
- Self-check before writing `gold.dim_customer`:
  - Print/log the schema of `dim_customer_joined_df` and `dim_customer_ranked_df`. The ranked DataFrame must contain `_rn`, `rank_customer_id`, `rank_address_type`, `rank_address_id`.
- Final validation checks before writing `gold.dim_customer`:
  - required output columns must match this list exactly
  - `customer_id` must be non-null
  - `customer_id` must be unique
- Output columns must be exactly:
  - `customer_id`
  - `customer_name_preferred`
  - `full_name`
  - `company_name`
  - `sales_person`
  - `email_address`
  - `phone`
  - `name_style`
  - `customer_address_type`
  - `customer_address_id`
  - `customer_address_line1`
  - `customer_address_line2`
  - `customer_city`
  - `customer_state_province`
  - `customer_country_region`
  - `customer_postal_code`
  - `modified_date`
- `sales_person` in `gold.dim_customer` comes only from `silver.customer.sales_person` when that optional column exists; otherwise output null as `sales_person`.
- Use explicit renames from joined aliases; do not leave duplicate raw `address_id`, `city`, `state_province`, or `postal_code` columns unqualified.

### `gold.dim_product`
- Grain: exactly one row per `product_id`.
- Mandatory enrichment route for this run: use base tables and bridge tables, not source views, to avoid column-name uncertainty.
- Join path:
  - start from `silver.product p`
  - left join `silver.product_category pc_child` on `p.product_category_id = pc_child.product_category_id`
  - left join `silver.product_category pc_parent` on `pc_child.parent_product_category_id = pc_parent.product_category_id`
  - left join english-only bridge `silver.product_model_product_description pmpd` on `p.product_model_id = pmpd.product_model_id AND upper(pmpd.culture) = 'EN'`
  - left join `silver.product_description pd` on `pmpd.product_description_id = pd.product_description_id`
  - left join `silver.product_model pm` on `p.product_model_id = pm.product_model_id`
- Mandatory implementation pattern for this table:
  - Step 1: create `pmpd_en_df` containing only english rows and only columns `product_model_id`, `product_description_id`.
  - Step 2: deduplicate english bridge rows to one row per `product_model_id` by lowest `product_description_id`, resulting in `pmpd_en_dedup_df`.
  - Step 3: validate `pmpd_en_dedup_df` uniqueness on `product_model_id`.
  - Step 4: create `dim_product_joined_df` from the specified joins. In the SAME select, project `pc_child.name as subcategory_src` and `pc_parent.name as category_src` so neither is referenced by an unaliased `name` later.
  - Step 5: validate `dim_product_joined_df` contains all source columns needed for the final projection, including `subcategory_src` and `category_src`.
  - Step 6: create `dim_product_final_df` by projecting exactly the final columns below.
- Category hierarchy rule:
  - `subcategory` = `subcategory_src` (i.e. `pc_child.name`)
  - `category` = `category_src` (i.e. `pc_parent.name`)
  - If `pc_child.parent_product_category_id` is null, then set:
    - `subcategory = subcategory_src`
    - `category = null`
- English description rule:
  - keep only one english bridge row per `product_model_id`
  - if multiple english rows exist, choose the lowest `product_description_id`
  - perform this bridge dedup in a dedicated intermediate DataFrame before joining to `p`
- Final validation checks before writing `gold.dim_product`:
  - required output columns must match this list exactly
  - `product_id` must be non-null
  - `product_id` must be unique
- Output columns must be exactly:
  - `product_id`
  - `product_name` from `p.name`
  - `product_number`
  - `color`
  - `size`
  - `weight`
  - `standard_cost`
  - `list_price`
  - `product_category_id`
  - `product_model_id`
  - `product_model_name` from `pm.name` when present, else null
  - `product_description` from `pd.description`
  - `category`
  - `subcategory`
  - `is_discontinued`
  - `is_sellable_currently`
- Self-join safety:
  - never reference unaliased `name` or unaliased `product_category_id`
  - always project `pc_child.name as subcategory_src` and `pc_parent.name as category_src` in the Step 4 join select, and reference only those helper names downstream

### `gold.dim_sales`
- Grain: exactly one row per cleaned salesperson value present in `silver.sales_order_header`.
- Source: `silver.sales_order_header h`
- Resolve the source salesperson column from the header contract above into `sales_person_source`, then derive `sales_person_clean` from that resolved field.
- Mandatory implementation pattern for this table:
  - Step 1: create `dim_sales_source_df` from header with only the resolved `sales_person_source`.
  - Step 2: validate `dim_sales_source_df` contains `sales_person_source`.
  - Step 3: create `dim_sales_clean_df` with `sales_person_clean`.
  - Step 4: create `dim_sales_final_df` by filtering to non-null `sales_person_clean` and selecting distinct `sales_person_clean`.
- Keep only non-null distinct cleaned salesperson values.
- Final validation checks before writing `gold.dim_sales`:
  - required output columns must match this list exactly
  - `sales_person_clean` must be non-null
  - `sales_person_clean` must be unique
- Output columns must be exactly:
  - `sales_person_clean`

### `gold.fact_sales`
- Grain: exactly one row per `sales_order_id`, `sales_order_detail_id`.
- Join path:
  - start from `silver.sales_order_detail d`
  - inner join `silver.sales_order_header h` on `d.sales_order_id = h.sales_order_id`
- Mandatory implementation pattern for this table:
  - Step 1: create `fact_sales_joined_df` from the specified inner join and immediately resolve `sales_person_source` from header in that same joined step.
  - Step 2: validate `fact_sales_joined_df` contains all required header/detail source columns and `sales_person_source`.
  - Step 3: create `fact_sales_final_df` by deriving `sales_person_clean` from the shared salesperson derivation contract and projecting the final output list only.
- After joining, immediately project to the final output list only; do not keep duplicate header/detail columns in an intermediate persisted DataFrame.
- `sales_person_clean` in fact must be derived from the shared salesperson derivation contract using the resolved header field `sales_person_source`.
- Final validation checks before writing `gold.fact_sales`:
  - required output columns must match this list exactly
  - `sales_order_id` and `sales_order_detail_id` must both be non-null
  - the pair (`sales_order_id`, `sales_order_detail_id`) must be unique
- Output columns must be exactly:
  - `sales_order_id`
  - `sales_order_detail_id`
  - `customer_id`
  - `product_id`
  - `sales_person_clean`
  - `sales_order_number`
  - `purchase_order_number`
  - `account_number`
  - `order_date`
  - `due_date`
  - `ship_date`
  - `status`
  - `order_status_desc`
  - `online_order_flag`
  - `ship_method`
  - `bill_to_address_id`
  - `ship_to_address_id`
  - `order_qty`
  - `unit_price`
  - `unit_price_discount`
  - `gross_line_amount`
  - `discount_amount`
  - `net_line_amount`
  - `sub_total`
  - `tax_amt`
  - `freight`
  - `total_due`

### Gold table classification based on actual columns
- Fact-like transactional tables:
  - `sales_order_header` because it has order dates, customer/address references, and monetary totals
  - `sales_order_detail` because it has line quantities, product references, prices, discounts, and line totals
- Dimension/master tables:
  - `customer`
  - `address`
  - `product`
  - `product_category`
  - `product_model`
  - `product_description`
- Junction/bridge tables:
  - `customer_address`
  - `product_model_product_description` --> Use only english, I don't need multilanguage
- Derived/reference views:
  - `v_get_all_categories`
  - `v_product_and_description`
  - `v_product_model_catalog_description`

### Proposed Gold star schema
Primary recommendation: line-level sales star centered on order detail, with order header as a header-related fact or degenerate dimension source.

#### Dimensions
- `gold.dim_customer`
  - Grain: one row per `customer_id`
  - Source: `silver.customer` flattened with `silver.customer_address` and `silver.address` using the deterministic address ranking rule above
  - Attributes:
    - `customer_id`
    - `customer_name_preferred`
    - `full_name`
    - `company_name`
    - `sales_person`
    - `email_address`
    - `phone`
    - `name_style`
    - `customer_address_type`
    - `customer_address_id`
    - `customer_address_line1`
    - `customer_address_line2`
    - `customer_city`
    - `customer_state_province`
    - `customer_country_region`
    - `customer_postal_code`
    - `modified_date`

- `gold.dim_product`
  - Grain: one row per `product_id`
  - Source: `silver.product` enriched from `silver.product_category`, `silver.product_model`, `silver.product_model_product_description`, and `silver.product_description`
  - Attributes:
    - `product_id`
    - `product_name`
    - `product_number`
    - `color`
    - `size`
    - `weight`
    - `standard_cost`
    - `list_price`
    - `product_category_id`
    - `product_model_id`
    - `product_model_name`
    - `product_description`
    - `category`
    - `subcategory`
    - `is_discontinued`
    - `is_sellable_currently`

- `gold.dim_sales`
  - Grain: one row per cleaned salesperson value
  - Source: `silver.sales_order_header`
  - Attributes:
    - `sales_person_clean`

#### Facts
- `gold.fact_sales`
  - Grain: one row per `sales_order_id`, `sales_order_detail_id`
  - Source: `silver.sales_order_detail` joined to `silver.sales_order_header`
  - Foreign keys:
    - `customer_id`
    - `product_id`
    - `sales_person_clean`
  - Degenerate dimensions:
    - `sales_order_number`
    - `purchase_order_number`
    - `account_number`
    - `status`
    - `online_order_flag`
    - `ship_method`
  - Measures:
    - `order_qty`
    - `unit_price`
    - `unit_price_discount`
    - `gross_line_amount`
    - `discount_amount`
    - `net_line_amount`
    - `sub_total`
    - `tax_amt`
    - `freight`
    - `total_due`

### Alternative Gold modeling choices the user may edit
- Option 1: Keep only the four requested final Gold outputs: `dim_customer`, `dim_product`, `dim_sales`, and `fact_sales`.
- Option 2: Later add helper dimensions such as date or ship method if the user explicitly requests them.
- Option 3: Later switch product enrichment to source views if the user explicitly wants view-driven semantics.
- Proposed default: Option 1 with the exact four requested persisted Gold tables.

### Data quality tests
Apply at minimum these 5 standard tests across Bronze/Silver/Gold, plus targeted RI checks.

1. Row count reconciliation
- Compare source object count to Bronze
- Compare Bronze to Silver after dedup
- Compare `silver.sales_order_detail` count to `gold.fact_sales`
- Compare distinct `silver.customer.customer_id` count to `gold.dim_customer`
- Compare distinct `silver.product.product