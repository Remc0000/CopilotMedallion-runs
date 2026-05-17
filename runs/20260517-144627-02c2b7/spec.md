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
- After every join, immediately project to an explicit alias-prefixed/select-renamed column list to prevent ambiguous names from propagating.
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

### Silver table specifications
- `silver.address`
  - Source: `bronze.address`
  - Key: `address_id`
  - Dedup key: `address_id`
  - Retain: `address_line1`, `address_line2`, `city`, `state_province`, `country_region`, `postal_code`, `rowguid`, `modified_date`
- `silver.customer`
  - Source: `bronze.customer`
  - Key: `customer_id`
  - Dedup key: `customer_id`
  - Retain: `name_style`, `title`, `first_name`, `middle_name`, `last_name`, `suffix`, `company_name`, `sales_person`, `email_address`, `phone`, `rowguid`, `modified_date`
  - Add derived columns:
    - `full_name` from name parts
    - `customer_name_preferred` choosing `company_name` when present, otherwise `full_name`
- `silver.customer_address`
  - Source: `bronze.customer_address`
  - Dedup key: `customer_id`, `address_id`, `address_type`
  - Retain junction structure for later billing/shipping role-play logic
- `silver.product`
  - Source: `bronze.product`
  - Key: `product_id`
  - Dedup key: `product_id`
  - Retain: `name`, `product_number`, `color`, `standard_cost`, `list_price`, `size`, `weight`, `product_category_id`, `product_model_id`, `sell_start_date`, `sell_end_date`, `discontinued_date`, `thumbnail_photo_file_name`, `rowguid`, `modified_date`
  - Add derived flags:
    - `is_discontinued`
    - `is_sellable_currently`
- `silver.product_category`
  - Source: `bronze.product_category`
  - Key: `product_category_id`
  - Dedup key: `product_category_id`
- `silver.product_description`
  - Source: `bronze.product_description`
  - Key: `product_description_id`
  - Dedup key: `product_description_id`
- `silver.product_model`
  - Source: `bronze.product_model`
  - Key: `product_model_id`
  - Dedup key: `product_model_id`
- `silver.product_model_product_description`
  - Source: `bronze.product_model_product_description`
  - Dedup key: `product_model_id`, `product_description_id`, `culture`
- `silver.sales_order_header`
  - Source: `bronze.sales_order_header`
  - Key: `sales_order_id`
  - Dedup key: `sales_order_id`
  - Retain transactional columns and add:
    - `order_date_key` as `yyyyMMdd` integer from `order_date`
    - `due_date_key` as `yyyyMMdd` integer from `due_date`
    - `ship_date_key` as `yyyyMMdd` integer from `ship_date` when not null
    - `order_year`, `order_month`, `order_status_desc` from `status`
- `silver.sales_order_detail`
  - Source: `bronze.sales_order_detail`
  - Key: `sales_order_id`, `sales_order_detail_id`
  - Dedup key: `sales_order_id`, `sales_order_detail_id`
  - Add derived columns:
    - `gross_line_amount = order_qty * unit_price`
    - `discount_amount = order_qty * unit_price * unit_price_discount`
    - `net_line_amount = line_total`
- `silver.v_get_all_categories`
  - Source: `bronze.v_get_all_categories`
  - Dedup key: `product_category_id`
  - Use as optional hierarchy helper in Gold
- `silver.v_product_and_description`
  - Source: `bronze.v_product_and_description`
  - Dedup key: `product_id`, `culture`
  - Optional multilingual product description helper
- `silver.v_product_model_catalog_description`
  - Source: `bronze.v_product_model_catalog_description`
  - Dedup key: `product_model_id`
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

### `gold.dim_customer`
- Grain: exactly one row per `customer_id`.
- Join path:
  - start from `silver.customer c`
  - left join `silver.customer_address ca` on `c.customer_id = ca.customer_id`
  - left join `silver.address a` on `ca.address_id = a.address_id`
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
- Category hierarchy rule:
  - `subcategory` = `pc_child.name`
  - `category` = `pc_parent.name`
  - If `pc_child.parent_product_category_id` is null, then set:
    - `subcategory = pc_child.name`
    - `category = null`
- English description rule:
  - keep only one english bridge row per `product_model_id`
  - if multiple english rows exist, choose the lowest `product_description_id`
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
  - always project `pc_child.name as subcategory` and `pc_parent.name as category`

### `gold.dim_sales`
- Grain: exactly one row per cleaned salesperson value present in `silver.sales_order_header`.
- Source: `silver.sales_order_header h`
- Resolve the source salesperson column from the header contract above into `sales_person_source`, then derive `sales_person_clean` from that resolved field.
- Keep only non-null distinct cleaned salesperson values.
- Output columns must be exactly:
  - `sales_person_clean`

### `gold.fact_sales`
- Grain: exactly one row per `sales_order_id`, `sales_order_detail_id`.
- Join path:
  - start from `silver.sales_order_detail d`
  - inner join `silver.sales_order_header h` on `d.sales_order_id = h.sales_order_id`
- After joining, immediately project to the final output list only; do not keep duplicate header/detail columns in an intermediate persisted DataFrame.
- `sales_person_clean` in fact must be derived from the shared salesperson derivation contract using the resolved header field `sales_person_source`.
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
- Compare distinct `silver.product.product_id` count to `gold.dim_product`

2. No-null primary key tests
- `customer.customer_id`
- `address.address_id`
- `product.product_id`
- `product_category.product_category_id`
- `product_model.product_model_id`
- `sales_order_header.sales_order_id`
- `sales_order_detail.sales_order_id`, `sales_order_detail.sales_order_detail_id`
- `gold.dim_customer.customer_id`
- `gold.dim_product.product_id`
- `gold.fact_sales.sales_order_id`, `gold.fact_sales.sales_order_detail_id`

3. Unique primary/composite key tests
- `customer_id`
- `address_id`
- `product_id`
- `product_category_id`
- `product_model_id`
- `sales_order_id`
- `sales_order_id + sales_order_detail_id`
- `customer_id + address_id + address_type`
- `product_model_id + product_description_id + culture`
- `gold.dim_customer.customer_id`
- `gold.dim_product.product_id`
- `gold.dim_sales.sales_person_clean`
- `gold.fact_sales.sales_order_id + gold.fact_sales.sales_order_detail_id`

4. Referential integrity tests
- `sales_order_header.customer_id -> customer.customer_id`
- `sales_order_header.ship_to_address_id -> address.address_id`
- `sales_order_header.bill_to_address_id -> address.address_id`
- `sales_order_detail.sales_order_id -> sales_order_header.sales_order_id`
- `sales_order_detail.product_id -> product.product_id`
- `product.product_category_id -> product_category.product_category_id`
- `product.product_model_id -> product_model.product_model_id`
- `customer_address.customer_id -> customer.customer_id`
- `customer_address.address_id -> address.address_id`
- `gold.fact_sales.customer_id -> gold.dim_customer.customer_id`
- `gold.fact_sales.product_id -> gold.dim_product.product_id`
- `gold.fact_sales.sales_person_clean -> gold.dim_sales.sales_person_clean`

5. Domain/value checks
- `order_qty > 0`
- `unit_price >= 0`
- `line_total >= 0`
- `sub_total >= 0`, `tax_amt >= 0`, `freight >= 0`, `total_due >= 0`
- `status` within known observed codes from source
- `unit_price_discount` between 0 and 1 if represented as a rate
- `ship_date >= order_date` when ship date is not null
- `due_date >= order_date` when due date is not null

## Semantic model
Create a Direct Lake semantic model on top of the four Gold tables explicitly requested by the user: `dim_customer`, `dim_product`, `dim_sales`, and `fact_sales`.

### Semantic model tables
- Dimensions:
  - `dim_customer`
  - `dim_product`
  - `dim_sales`
- Fact:
  - `fact_sales`

### Expected dimension/fact content
- `dim_customer`
  - One row per `customer_id`
  - Includes customer descriptive columns from `silver.customer`
  - Includes flattened address context from `silver.customer_address` and `silver.address`
  - Flattening must use the deterministic Gold rule defined above so the table remains exactly one row per `customer_id`
  - Expose:
    - `customer_id`
    - `customer_name_preferred`
    - `full_name`
    - `company_name`
    - `email_address`
    - `phone`
    - `sales_person`
    - `customer_address_type`
    - `customer_address_id`
    - `customer_address_line1`
    - `customer_address_line2`
    - `customer_city`
    - `customer_state_province`
    - `customer_country_region`
    - `customer_postal_code`
    - `modified_date`

- `dim_product`
  - One row per `product_id`
  - Includes joined attributes from product, product category self-join, product model, english-only product model/product description bridge, and product description
  - Expose:
    - `product_id`
    - `product_name`
    - `product_number`
    - `color`
    - `size`
    - `weight`
    - `standard_cost`
    - `list_price`
    - `product_model_id`
    - `product_model_name`
    - `product_description`
    - `category`
    - `subcategory`
    - `is_discontinued`
    - `is_sellable_currently`

- `dim_sales`
  - One row per cleaned salesperson value derived from `sales_order_header`
  - Must use the cleaned salesperson text from Gold, where:
    - `adventureworks/` prefix is removed
    - trailing numeric suffix is removed
  - Expose:
    - `sales_person_clean`

- `fact_sales`
  - Grain: one row per `sales_order_id`, `sales_order_detail_id`
  - Joined from `sales_order_detail` and `sales_order_header`
  - Expose:
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

### Relationships
Use a simple star schema with single-direction filtering from dimensions to fact.

- `fact_sales[customer_id]` -> `dim_customer[customer_id]`
- `fact_sales[product_id]` -> `dim_product[product_id]`
- `fact_sales[sales_person_clean]` -> `dim_sales[sales_person_clean]`

### Date handling
Because the requested Gold design does not call for a separate date dimension, keep date columns directly on `fact_sales`.

- Keep these reportable date columns on `fact_sales`:
  - `order_date`
  - `due_date`
  - `ship_date`
- Set default date behavior to use `order_date` for most visuals
- If needed, allow report authors to create separate visuals using `ship_date` or `due_date`
- Do not introduce `dim_date` unless the user later edits Gold to include it

### Recommended hidden fields
- Hide audit columns, hashes, and ETL control fields
- Hide raw technical lineage columns
- Hide rowguid-like technical identifiers if surfaced
- Hide address identifier columns in `fact_sales` unless drillthrough requires them

### Explicit measures
Use measures primarily from `fact_sales` because the requested Gold model centers on a single sales fact.

- `Sales Amount = SUM(fact_sales[net_line_amount])`
- `Gross Sales Amount = SUM(fact_sales[gross_line_amount])`
- `Discount Amount = SUM(fact_sales[discount_amount])`
- `Order Quantity = SUM(fact_sales[order_qty])`
- `Order Count = DISTINCTCOUNT(fact_sales[sales_order_id])`
- `Customer Count = DISTINCTCOUNT(fact_sales[customer_id])`
- `Product Count = DISTINCTCOUNT(fact_sales[product_id])`
- `Average Selling Price = DIVIDE([Sales Amount], [Order Quantity])`
- `Average Order Value = DIVIDE(SUM(fact_sales[total_due]), [Order Count])`
- `Total Due = SUM(fact_sales[total_due])`
- `Sub Total = SUM(fact_sales[sub_total])`
- `Tax Amount = SUM(fact_sales[tax_amt])`
- `Freight Amount = SUM(fact_sales[freight])`

### Semantic model notes
- Product hierarchy analysis should use `dim_product[category]` and `dim_product[subcategory]`.
- Salesperson analysis must use the cleaned salesperson attribute from `dim_sales`.
- Because `fact_sales` contains both line-level amounts and repeated header-level monetary columns, report authors must be careful with `sub_total`, `tax_amt`, `freight`, and `total_due` to avoid double counting if Gold does not allocate them to line level.
- Prefer `Sales Amount`, `Gross Sales Amount`, `Discount Amount`, and `Order Quantity` for product and customer analysis.

## Report
Build a Power BI report tailored to the requested Gold model: customer, product hierarchy, salesperson, and sales fact analysis.

### Page 1: Sales Overview
Visuals:
- KPI cards:
  - `Sales Amount`
  - `Order Count`
  - `Customer Count`
  - `Average Order Value`
- Line chart:
  - `Sales Amount` by `fact_sales[order_date]`
- Clustered column chart:
  - `Sales Amount` by `dim_sales[sales_person_clean]`
- Stacked column or bar chart:
  - `Sales Amount` by `dim_product[category]` and `dim_product[subcategory]`
- Donut chart:
  - `Sales Amount` by `fact_sales[online_order_flag]`

Filters/slicers:
- `fact_sales[order_date]`
- `dim_sales[sales_person_clean]`
- `dim_product[category]`
- `dim_product[subcategory]`
- `dim_customer[customer_name_preferred]`

### Page 2: Product Analysis
Visuals:
- Matrix:
  - Rows: `dim_product[category]` > `dim_product[subcategory]` > `dim_product[product_name]`
  - Values: `Sales Amount`, `Order Quantity`, `Discount Amount`
- Bar chart:
  - Top products by `Sales Amount`
- Scatter chart:
  - `dim_product[list_price]` vs `Sales Amount` by `dim_product[product_name]`
- Table:
  - `product_name`, `product_number`, `product_model_name`, `product_description`, `color`, `list_price`, `standard_cost`, `Sales Amount`, `Order Quantity`
- Optional slicers:
  - `is_discontinued`
  - `is_sellable_currently`

### Page 3: Customer Analysis
Visuals:
- Bar chart:
  - Top customers by `Sales Amount`
- Table:
  - `customer_name_preferred`, `full_name`, `company_name`, `email_address`, `phone`, address-related columns from `dim_customer`, `Order Count`, `Sales Amount`
- Map or filled map if geography columns are present in `dim_customer` after flattening:
  - Use customer address fields retained in Gold, such as city/state/country
  - Value: `Sales Amount`
- Column chart:
  - `Sales Amount` by customer city or country if available in `dim_customer`

Filters/slicers:
- customer/company
- city/state/country if present
- salesperson

### Page 4: Salesperson Analysis
Visuals:
- Bar chart:
  - `Sales Amount` by `dim_sales[sales_person_clean]`
- Column chart:
  - `Order Count` by `dim_sales[sales_person_clean]`
- Table:
  - `sales_person_clean`, `Sales Amount`, `Order Count`, `Customer Count`
- Ribbon or stacked chart:
  - `Sales Amount` by salesperson across `dim_product[category]`
- Detail table:
  - salesperson, customer, product, order date, order quantity, sales amount

### Page 5: Order Detail
Visuals:
- Detail table:
  - `sales_order_number`
  - `sales_order_id`
  - `sales_order_detail_id`
  - `order_date`
  - `due_date`
  - `ship_date`
  - customer
  - product
  - salesperson
  - `order_qty`
  - `unit_price`
  - `unit_price_discount`
  - `gross_line_amount`
  - `discount_amount`
  - `net_line_amount`
  - `status`
  - `ship_method`
- Decomposition tree:
  - Analyze `Sales Amount` by category > subcategory > product > customer > salesperson
- Drillthrough target:
  - From customer, product, or salesperson to order-line detail

### Report notes
- Default all trend visuals to `fact_sales[order_date]`.
- Use `dim_product[category]` and `dim_product[subcategory]` consistently rather than raw category IDs.
- Use only cleaned salesperson values from `dim_sales`.
- If header-level columns such as `total_due` remain repeated on each line in `fact_sales`, do not use them in visuals that aggregate across many order lines unless Gold explicitly prevents double counting.

## Data Agent
Create an AISkill grounded on the semantic model built from `dim_customer`, `dim_product`, `dim_sales`, and `fact_sales`.

### Role
Sales analytics assistant for customer, product hierarchy, salesperson, and sales-order analysis based on the Gold model derived from `SalesLT`.

### Domain hints
- Primary business process is sales at order-line grain in `fact_sales`.
- Customer analysis comes from a flattened customer dimension that combines customer, customer-address, and address information.
- Product analysis comes from a flattened product dimension that combines product, product model, english-only product description mapping, and a self-joined product category hierarchy.
- Product hierarchy should be interpreted as:
  - `category` = parent product category
  - `subcategory` = child product category tied directly to the product
- Salesperson analysis must use the cleaned salesperson value from `dim_sales`, where the `adventureworks/` prefix and trailing number have been removed.
- Date analysis should default to `order_date` unless the user explicitly asks for due-date or ship-date context.

### Starter questions
- What are total sales and order count over time?
- Which categories and subcategories generate the most sales?
- Which products have the highest sales amount and order quantity?
- Which customers or companies contribute the most sales?
- Which salesperson generates the highest sales amount?
- How do sales differ by cleaned salesperson over time?
- Which customer locations have the highest sales?
- What discounts were given by category, subcategory, and product?
- Which orders have the highest net line amount?
- How do online and non-online orders compare?

### Guardrails
- Answer only from the grounded semantic model and exposed measures/fields.
- Use cleaned salesperson values only; do not invent a broader salesperson hierarchy.
- Do not infer unsupported business concepts such as margin, profit, inventory, returns, targets, quotas, or commissions unless explicitly modeled.
- Do not use `password_hash`, `password_salt`, binary image content, or hidden technical columns in responses.
- When answering geography questions, rely only on customer/address fields actually retained in `dim_customer`.
- When a question asks for order totals using `total_due`, `sub_total`, `tax_amt`, or `freight`, be careful about line-grain duplication and clarify if the model exposes those as repeated line attributes.
- Refuse unsupported write-back or operational actions; this agent is analytic and read-only.