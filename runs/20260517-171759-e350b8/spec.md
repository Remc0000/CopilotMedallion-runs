# Run Spec 20260517-170731-a2063c

## Updated specs

### Iteration 1 — 2026-05-17 17:26:31Z — failed layer: gold (run: 20260517-171759-e350b8)
- **Root cause (1-line summary)**: Gold build likely failed due to unresolved/incorrect join columns and ambiguous self-join logic in `dim_product` and `dim_customer`, especially around category self-join, English description filtering, and `Main Office` address selection.
- **What was changed**:
  - Tightened **Generic guidance** to require schema assertions and alias-qualified joins for all Gold builds, especially self-joins and bridge-table joins.
  - Added an explicit **Gold** section defining required source tables, exact join paths, join keys, culture/address filters, self-join aliases, and output columns for `dim_customer`, `dim_product`, `dim_sales`, and `fact_sales`.
  - Clarified salesperson cleaning and customer address fallback behavior to avoid duplicate rows and missing-column assumptions.

### Iteration 2 — 2026-05-17 17:35:14Z — failed layer: gold (run: 20260517-171759-e350b8)
- **Root cause (1-line summary)**: Gold build still failed because the spec did not constrain the implementation to the actual SalesLT header column names and likely referenced a non-existent `sales_person` column in `silver.sales_order_header`.
- **What was changed**:
  - Tightened **Generic guidance** to require exact-source-name validation before Silver renames and to forbid inventing columns not present in upstream schemas.
  - Updated **Silver** to explicitly derive `sales_person` from source column `SalesPerson` if present, otherwise from `Salesperson`, and to fail fast if neither exists.
  - Updated **Gold** to require `dim_sales` and `fact_sales` to use only the validated `silver.sales_order_header.sales_person` column and to prohibit fallback to `customer.sales_person`.

### Iteration 3 — 2026-05-17 17:42:39Z — failed layer: silver (run: 20260517-171759-e350b8)
- **Root cause (1-line summary)**: Silver build likely failed from assuming `modified_date`/`rowguid` exist on every Bronze table, especially bridge/view objects such as `customer_address` and `product_model_product_description`.
- **What was changed**:
  - Tightened **Generic guidance** to require per-table Bronze schema inspection before Silver dedup/order/output logic and to forbid referencing optional source columns unless their presence is explicitly validated.
  - Updated **Silver common rules** to use source-specific tie-breakers when `modified_date` is absent and to make `rowguid`/`modified_date` optional passthrough columns rather than mandatory on every Silver table.
  - Updated the affected **Silver** table specs to name the exact expected columns for bridge/view tables and to require fail-fast schema assertions for the true business keys only.

## Inputs
- Workspace: `581484c6-28a0-48c9-8931-644c455e9376`
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
- Target Lakehouse: **CopilotMedallion_20260517_170731**

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
- Use defensive column references in every transformation: assert required columns exist before select, rename, join, filter, aggregate, or write.
- After every join, immediately project an explicit column list using aliases prefixed by table role to avoid ambiguous names.
- Before every `groupBy` / `agg`, assert all grouping and aggregate input columns exist.
- For REST/API responses, use defensive handling and fail fast with `if x is None: raise ...` before any `.get()` access.
- Do not use `saveAsTable`; write with explicit Lakehouse paths or managed table patterns supported by the target runtime.
- Use parameter cells at the top of every notebook for workspace, source lakehouse, target lakehouse, schema, load type, and run id.
- Use idempotent overwrite patterns for full loads and deterministic merge/upsert patterns where incremental logic is later added.
- Wrap notebook logic in error-loud `try/except`; on exception call `_save_error(layer, e)` and re-raise.
- Add standard audit columns consistently: `ingest_run_id`, `ingest_ts`, `source_system`, `source_table`, and where useful `record_hash`.
- Enforce snake_case naming in Silver and Gold.
- Never silently drop unexpected duplicates; quarantine or log them where business keys should be unique.
- Treat binary/image columns carefully; keep in Bronze, and only propagate to Silver/Gold if there is a clear reporting use case.
- In Gold, do not infer join keys from similar names; use only explicitly validated columns present in Silver.
- For every Gold join, assert the exact required columns before execution and use fully aliased DataFrames (`c`, `ca`, `a`, `p`, `pc_child`, `pc_parent`, `pm`, `pmpd`, `pd`, `soh`, `sod`).
- For self-joins, bridge tables, and filtered helper joins, never use `select("*")`; project only named output columns from the correct alias after the join.
- If a requested business filter yields no rows for some entities, preserve the base grain with a left join unless the spec explicitly requires row exclusion.
- Before any Silver rename step, inspect the actual Bronze schema and map source columns by exact observed names; do not assume camel-case, PascalCase, or alternate spellings without validation.
- Do not invent derived canonical columns in Silver unless the exact source-to-target mapping is stated in the spec. If a required source column is absent, fail fast with a clear schema assertion instead of guessing.
- In Silver, treat source-system columns such as `ModifiedDate` and `rowguid` as optional unless the spec for that exact source object requires them and the Bronze schema confirms they exist.
- For every Silver table, first inspect and log the Bronze schema, then build a per-table required-column set. Deduplication and ordering must only use columns actually present in that source object.
- If a Bronze source object lacks `ModifiedDate`, use `bronze_ingest_ts` as the deterministic latest-row tie-breaker. Do not reference `modified_date` in window specs, selects, or output projections unless it exists in the source and has been renamed into Silver.
- If a Bronze source object lacks `rowguid`, omit `rowguid` from the Silver output schema for that table rather than fabricating or referencing it.

Notebook cell comment requirement:
- EACH code cell must start with a short markdown-style comment block using Python comments.
- Required pattern:
  - `# ---`
  - `# <1-3 human-readable lines describing what the cell is doing and why>`
- Never emit a code cell with no leading comment block.
- Keep comments concise and scroll-friendly so a Fabric user can understand each cell at a glance.

## Bronze
Land each source object into a `bronze` schema with minimal transformation, preserving source fidelity.

Common Bronze pattern for all tables/views:
- Read from source Lakehouse table/view as-is.
- Add metadata columns:
  - `bronze_ingest_run_id`
  - `bronze_ingest_ts`
  - `bronze_source_system` = `SalesLT`
  - `bronze_source_object` = source object name
- Write mode: overwrite for this initial build spec.
- Partitioning:
  - Partition large transactional tables by date derived from source timestamp where available.
  - Do not partition small dimensions/junctions/views unless row volume later justifies it.
- Preserve original column casing and types in Bronze.
- Keep all source columns, including `rowguid`, `ModifiedDate`, and binary columns.

### Bronze table landing plan
- `bronze.address`
  - Source: `SalesLT/Address`
  - Natural key candidate: `AddressID`
  - Partitioning: none
  - Notes: retain address lines, geographic fields, `ModifiedDate`

- `bronze.customer`
  - Source: `SalesLT/Customer`
  - Natural key candidate: `CustomerID`
  - Partitioning: none
  - Notes: retain PII-like fields and security-sensitive fields (`PasswordHash`, `PasswordSalt`) in Bronze only; flag for removal in Silver

- `bronze.customer_address`
  - Source: `SalesLT/CustomerAddress`
  - Natural key candidate: composite (`CustomerID`, `AddressID`, `AddressType`)
  - Partitioning: none
  - Notes: this appears to be a junction/bridge between customers and addresses

- `bronze.product`
  - Source: `SalesLT/Product`
  - Natural key candidate: `ProductID`
  - Partitioning: none
  - Notes: retain binary `ThumbNailPhoto` in Bronze; retain price/cost dates and category/model keys

- `bronze.product_category`
  - Source: `SalesLT/ProductCategory`
  - Natural key candidate: `ProductCategoryID`
  - Partitioning: none
  - Notes: preserve parent-child hierarchy via `ParentProductCategoryID`

- `bronze.product_description`
  - Source: `SalesLT/ProductDescription`
  - Natural key candidate: `ProductDescriptionID`
  - Partitioning: none

- `bronze.product_model`
  - Source: `SalesLT/ProductModel`
  - Natural key candidate: `ProductModelID`
  - Partitioning: none

- `bronze.product_model_product_description`
  - Source: `SalesLT/ProductModelProductDescription`
  - Natural key candidate: composite (`ProductModelID`, `ProductDescriptionID`, `Culture`)
  - Partitioning: none
  - Notes: bridge between product model and multilingual descriptions

- `bronze.sales_order_detail`
  - Source: `SalesLT/SalesOrderDetail`
  - Natural key candidate: composite (`SalesOrderID`, `SalesOrderDetailID`)
  - Partitioning: by `ModifiedDate` month or derived `modified_year_month`
  - Notes: transactional line-level table, likely fact grain input

- `bronze.sales_order_header`
  - Source: `SalesLT/SalesOrderHeader`
  - Natural key candidate: `SalesOrderID`
  - Partitioning: by `OrderDate` month
  - Notes: transactional order header, likely fact/order grain input

- `bronze.v_get_all_categories`
  - Source: `SalesLT/vGetAllCategories`
  - Natural key candidate: `ProductCategoryID`
  - Partitioning: none
  - Notes: derived/category flattening view; ingest for convenience but prefer base tables for authoritative lineage

- `bronze.v_product_and_description`
  - Source: `SalesLT/vProductAndDescription`
  - Natural key candidate: composite (`ProductID`, `Culture`)
  - Partitioning: none
  - Notes: convenience denormalized view combining product/model/description fields

- `bronze.v_product_model_catalog_description`
  - Source: `SalesLT/vProductModelCatalogDescription`
  - Natural key candidate: `ProductModelID`
  - Partitioning: none
  - Notes: extended product-model attributes; likely useful for product enrichment

## Silver
Standardize, clean, deduplicate, and conform data into `silver` schema with snake_case names and reporting-safe attributes.

Common Silver rules:
- Rename all columns to snake_case.
- Add:
  - `silver_loaded_ts`
  - `silver_run_id`
  - `is_current` = true for current-state full load outputs
  - `record_hash` over business content columns for change detection readiness
- Preserve source business keys as integer/string columns.
- Deduplicate using the latest available business timestamp for that specific source object; prefer `modified_date` only when the Bronze schema actually contains `ModifiedDate`. If no business timestamp exists, use highest `bronze_ingest_ts`.
- Quarantine duplicates that violate expected PK uniqueness.
- Remove or mask sensitive fields not needed analytically.
- `rowguid` and `modified_date` are optional passthrough columns in Silver. Include them only for tables where the Bronze schema contains `rowguid` and `ModifiedDate` respectively after snake_case renaming.
- For bridge tables and helper views, do not assume system columns exist; required Silver outputs are limited to the explicitly named business columns plus audit columns, unless schema validation confirms extra passthrough columns.

### silver.address
- Source: `bronze.address`
- Required Bronze columns before rename: `AddressID`, `AddressLine1`, `AddressLine2`, `City`, `StateProvince`, `CountryRegion`, `PostalCode`
- Optional Bronze passthrough columns: `rowguid`, `ModifiedDate`
- Dedup key: `address_id`
- Latest-row rule: highest `modified_date` when present, else highest `bronze_ingest_ts`
- Cleansing:
  - trim text fields
  - standardize empty strings to null for `address_line2`
  - retain `city`, `state_province`, `country_region`, `postal_code`
- Audit/output columns:
  - required: `address_id`, `address_line1`, `address_line2`, `city`, `state_province`, `country_region`, `postal_code`, audit columns
  - optional when present in source: `rowguid`, `modified_date`

### silver.customer
- Source: `bronze.customer`
- Required Bronze columns before rename: `CustomerID`, `NameStyle`, `Title`, `FirstName`, `MiddleName`, `LastName`, `Suffix`, `CompanyName`, `SalesPerson`, `EmailAddress`, `Phone`
- Optional Bronze passthrough columns: `rowguid`, `ModifiedDate`
- Dedup key: `customer_id`
- Latest-row rule: highest `modified_date` when present, else highest `bronze_ingest_ts`
- Cleansing:
  - trim name/contact fields
  - create `full_name` from available title/first/middle/last/suffix for convenience
  - retain `company_name`, `sales_person`, `email_address`, `phone`
  - drop `password_hash` and `password_salt` from Silver for security
- Audit/output columns:
  - required: `customer_id`, `name_style`, `title`, `first_name`, `middle_name`, `last_name`, `suffix`, `full_name`, `company_name`, `sales_person`, `email_address`, `phone`, audit columns
  - optional when present in source: `rowguid`, `modified_date`

### silver.customer_address
- Source: `bronze.customer_address`
- Required Bronze columns before rename: `CustomerID`, `AddressID`, `AddressType`
- Optional Bronze passthrough columns: `rowguid`, `ModifiedDate`
- Dedup key: (`customer_id`, `address_id`, `address_type`)
- Latest-row rule: highest `modified_date` when present, else highest `bronze_ingest_ts`
- Cleansing:
  - trim `address_type`
  - keep as bridge table
- Audit/output columns:
  - required: `customer_id`, `address_id`, `address_type`, audit columns
  - optional when present in source: `rowguid`, `modified_date`

### silver.product
- Source: `bronze.product`
- Required Bronze columns before rename: `ProductID`, `Name`, `ProductNumber`, `Color`, `StandardCost`, `ListPrice`, `Size`, `Weight`, `ProductCategoryID`, `ProductModelID`, `SellStartDate`, `SellEndDate`, `DiscontinuedDate`, `ThumbNailPhotoFileName`
- Optional Bronze passthrough columns: `rowguid`, `ModifiedDate`
- Dedup key: `product_id`
- Latest-row rule: highest `modified_date` when present, else highest `bronze_ingest_ts`
- Cleansing:
  - trim descriptive fields
  - retain measures: `standard_cost`, `list_price`, `weight`
  - retain lifecycle dates: `sell_start_date`, `sell_end_date`, `discontinued_date`
  - drop `thumbnail_photo` from Silver unless image reporting is explicitly required
  - retain `thumbnail_photo_file_name`
- Audit/output columns:
  - required: `product_id`, `name`, `product_number`, `color`, `standard_cost`, `list_price`, `size`, `weight`, `product_category_id`, `product_model_id`, `sell_start_date`, `sell_end_date`, `discontinued_date`, `thumbnail_photo_file_name`, audit columns
  - optional when present in source: `rowguid`, `modified_date`

### silver.product_category
- Source: `bronze.product_category`
- Required Bronze columns before rename: `ProductCategoryID`, `ParentProductCategoryID`, `Name`
- Optional Bronze passthrough columns: `rowguid`, `ModifiedDate`
- Dedup key: `product_category_id`
- Latest-row rule: highest `modified_date` when present, else highest `bronze_ingest_ts`
- Cleansing:
  - trim `name`
  - preserve hierarchy via `parent_product_category_id`
- Audit/output columns:
  - required: `product_category_id`, `parent_product_category_id`, `name`, audit columns
  - optional when present in source: `rowguid`, `modified_date`

### silver.product_description
- Source: `bronze.product_description`
- Required Bronze columns before rename: `ProductDescriptionID`, `Description`
- Optional Bronze passthrough columns: `rowguid`, `ModifiedDate`
- Dedup key: `product_description_id`
- Latest-row rule: highest `modified_date` when present, else highest `bronze_ingest_ts`
- Cleansing:
  - trim `description`
- Audit/output columns:
  - required: `product_description_id`, `description`, audit columns
  - optional when present in source: `rowguid`, `modified_date`

### silver.product_model
- Source: `bronze.product_model`
- Required Bronze columns before rename: `ProductModelID`, `Name`, `CatalogDescription`
- Optional Bronze passthrough columns: `rowguid`, `ModifiedDate`
- Dedup key: `product_model_id`
- Latest-row rule: highest `modified_date` when present, else highest `bronze_ingest_ts`
- Cleansing:
  - trim `name`
  - keep `catalog_description` as long text
- Audit/output columns:
  - required: `product_model_id`, `name`, `catalog_description`, audit columns
  - optional when present in source: `rowguid`, `modified_date`

### silver.product_model_product_description
- Source: `bronze.product_model_product_description`
- Required Bronze columns before rename: `ProductModelID`, `ProductDescriptionID`, `Culture`
- Optional Bronze passthrough columns: `rowguid`, `ModifiedDate`
- Dedup key: (`product_model_id`, `product_description_id`, `culture`)
- Latest-row rule: highest `modified_date` when present, else highest `bronze_ingest_ts`
- Cleansing:
  - uppercase/normalize `culture`
- Audit/output columns:
  - required: `product_model_id`, `product_description_id`, `culture`, audit columns
  - optional when present in source: `rowguid`, `modified_date`

### silver.sales_order_header
- Source: `bronze.sales_order_header`
- Required Bronze columns before rename:
  - `SalesOrderID`, `RevisionNumber`, `OrderDate`, `DueDate`, `ShipDate`, `Status`, `OnlineOrderFlag`, `SalesOrderNumber`, `PurchaseOrderNumber`, `AccountNumber`, `CustomerID`, `ShipToAddressID`, `BillToAddressID`, `ShipMethod`, `CreditCardApprovalCode`, `SubTotal`, `TaxAmt`, `Freight`, `TotalDue`, `Comment`
- Optional Bronze passthrough columns: `rowguid`, `ModifiedDate`
- Dedup key: `sales_order_id`
- Latest-row rule: highest `modified_date` when present, else highest `bronze_ingest_ts`
- Exact source-column rule:
  - inspect `bronze.sales_order_header` and map the salesperson source column by exact observed name
  - if source column `SalesPerson` exists, rename it to `sales_person`
  - else if source column `Salesperson` exists, rename it to `sales_person`
  - else fail fast with a schema assertion; do not fabricate `sales_person`
- Cleansing:
  - trim string fields
  - retain order milestone timestamps and shipping/billing/customer keys
  - derive `order_date_key`, `due_date_key`, `ship_date_key` as `yyyyMMdd` integers for Gold/date joins
- Audit/output columns:
  - required: `sales_order_id`, `revision_number`, `order_date`, `due_date`, `ship_date`, `status`, `online_order_flag`, `sales_order_number`, `purchase_order_number`, `account_number`, `customer_id`, `ship_to_address_id`, `bill_to_address_id`, `ship_method`, `credit_card_approval_code`, `sales_person`, `sub_total`, `tax_amt`, `freight`, `total_due`, `comment`, `order_date_key`, `due_date_key`, `ship_date_key`, audit columns
  - optional when present in source: `rowguid`, `modified_date`

### silver.sales_order_detail
- Source: `bronze.sales_order_detail`
- Required Bronze columns before rename: `SalesOrderID`, `SalesOrderDetailID`, `OrderQty`, `ProductID`, `UnitPrice`, `UnitPriceDiscount`, `LineTotal`
- Optional Bronze passthrough columns: `rowguid`, `ModifiedDate`
- Dedup key: (`sales_order_id`, `sales_order_detail_id`)
- Latest-row rule: highest `modified_date` when present, else highest `bronze_ingest_ts`
- Cleansing:
  - retain quantity and price metrics
  - derive `gross_line_amount` = `order_qty * unit_price`
  - derive `discount_amount` = `order_qty * unit_price * unit_price_discount`
- Audit/output columns:
  - required: `sales_order_id`, `sales_order_detail_id`, `order_qty`, `product_id`, `unit_price`, `unit_price_discount`, `line_total`, `gross_line_amount`, `discount_amount`, audit columns
  - optional when present in source: `rowguid`, `modified_date`

### silver.v_get_all_categories
- Source: `bronze.v_get_all_categories`
- Required Bronze columns before rename: `ProductCategoryID`
- Optional Bronze passthrough columns: none assumed; only include additional columns if schema inspection confirms them
- Dedup key: `product_category_id`
- Latest-row rule: highest `bronze_ingest_ts`
- Use case:
  - optional helper table for flattened category reporting
  - not authoritative over `product_category`; use base table lineage for core dimensions
- Output columns:
  - required when present after exact schema inspection: `product_category_id`
  - include `parent_product_category_name` and `product_category_name` only if those exact source columns exist in the view
  - always include audit columns

### silver.v_product_and_description
- Source: `bronze.v_product_and_description`
- Required Bronze columns before rename: `ProductID`, `Culture`
- Optional Bronze passthrough columns: none assumed; only include additional columns if schema inspection confirms them
- Dedup key: (`product_id`, `culture`)
- Latest-row rule: highest `bronze_ingest_ts`
- Use case:
  - optional helper enrichment for product descriptions by culture
- Output columns:
  - required: `product_id`, `culture`, audit columns
  - include `name`, `product_model`, `description` only if those exact source columns exist in the view

### silver.v_product_model_catalog_description
- Source: `bronze.v_product_model_catalog_description`
- Required Bronze columns before rename: `ProductModelID`
- Optional Bronze passthrough columns: none assumed; only include additional columns if schema inspection confirms them
- Dedup key: `product_model_id`
- Latest-row rule: highest `bronze_ingest_ts`
- Use case:
  - optional helper enrichment for product model attributes
- Output columns:
  - required: `product_model_id`, audit columns
  - include `name`, `summary`, `manufacturer`, `copyright`, `product_url`, `warranty_period`, `warranty_description`, `no_of_years`, `maintenance_description`, `wheel`, `saddle`, `pedal`, `bike_frame`, `crankset`, `picture_angle`, `picture_size`, `product_photo_id`, `material`, `color`, `product_line`, `style`, `rider_experience` only if those exact source columns exist in the view
  - include `rowguid`, `modified_date` only if confirmed present in the Bronze schema

### Silver optimization
- Run `OPTIMIZE` on:
  - `silver.sales_order_header`
  - `silver.sales_order_detail`
  - `silver.product`
- Z-order / clustering preference if supported:
  - `sales_order_header`: `sales_order_id`, `customer_id`, `order_date_key`
  - `sales_order_detail`: `sales_order_id`, `product_id`
  - `product`: `product_id`, `product_category_id`, `product_model_id`

I want to end up with the following tables:
Dim_Customer -> join customer, customeraddress, address (notice there are multiple address types for a customer. I only need the address type Main Office),. Leave all relevant/meaningfull fields
Dim_Product -> join product, productcategory, productdescription, productmodel, productmodelproductdescription. Please notice that in productcategory there is a self join. So in the end I have the category (which is the parent category) and a subcategory. Also notice that in productmodelproductdescription there is a culture field, filter this on en
Dim_Sales --> Use the salesperson field from salesorderheader, but remove the adventureworks/ in front of the name and remove the last number at the end off the name
Fact_Sales --> join salesorderdetail and salesorderheader and keep all relevant fields

## Gold
Build business-facing Gold tables in `gold` schema using only explicitly validated Silver columns.

### Common Gold rules
- Gold naming must be snake_case table and column names.
- All Gold joins must use aliased DataFrames and explicit join predicates; no natural joins and no unqualified column references.
- Before building each Gold table, assert the required input columns exist exactly as named in Silver.
- After each join, select only the final target columns listed below; do not carry duplicate source columns forward.
- Preserve the requested base grain for each dimension/fact and prevent accidental row multiplication from bridge tables.
- Write Gold tables in overwrite mode for this initial full build.
- For Gold tables that depend on salesperson, use only `silver.sales_order_header.sales_person` after Silver validation. Do not read salesperson from `silver.customer` for `dim_sales` or `fact_sales`.
- If `silver.sales_order_header.sales_person` is absent, fail the Gold build immediately with a schema assertion naming that exact missing column.

### gold.dim_customer
- Base grain: one row per `customer_id`.
- Required inputs:
  - `silver.customer`: `customer_id`, `full_name`, `company_name`, `sales_person`, `email_address`, `phone`
  - `silver.customer_address`: `customer_id`, `address_id`, `address_type`
  - `silver.address`: `address_id`, `address_line1`, `address_line2`, `city`, `state_province`, `country_region`, `postal_code`
- Join path:
  - start from `silver.customer` as `c`
  - left join filtered `silver.customer_address` as `ca` on `c.customer_id = ca.customer_id`
  - left join `silver.address` as `a` on `ca.address_id = a.address_id`
- Address filter rule:
  - filter `silver.customer_address` to `address_type = 'Main Office'` after trimming
  - comparison should be exact on the cleaned value `Main Office`
- Duplicate prevention rule:
  - if more than one `Main Office` row exists for the same `customer_id`, keep exactly one deterministic row using:
    1. lowest `address_id`
    2. then latest `modified_date` if needed
  - quarantine any additional `Main Office` duplicates
- Missing address rule:
  - keep all customers even when no `Main Office` address exists; address fields may be null
- Output columns:
  - `customer_id`
  - `full_name`
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

### gold.dim_product
- Base grain: one row per `product_id`.
- Required inputs:
  - `silver.product`: `product_id`, `name`, `product_number`, `color`, `size`, `weight`, `standard_cost`, `list_price`, `product_category_id`, `product_model_id`, `sell_start_date`, `sell_end_date`, `discontinued_date`
  - `silver.product_category`: `product_category_id`, `parent_product_category_id`, `name`
  - `silver.product_model`: `product_model_id`, `name`
  - `silver.product_model_product_description`: `product_model_id`, `product_description_id`, `culture`
  - `silver.product_description`: `product_description_id`, `description`
- Authoritative lineage rule:
  - build `dim_product` from base tables only; do not substitute `silver.v_get_all_categories` or `silver.v_product_and_description` for the core join path
- Join path:
  - start from `silver.product` as `p`
  - left join `silver.product_category` as `pc_child` on `p.product_category_id = pc_child.product_category_id`
  - left join `silver.product_category` as `pc_parent` on `pc_child.parent_product_category_id = pc_parent.product_category_id`
  - left join `silver.product_model` as `pm` on `p.product_model_id = pm.product_model_id`
  - left join filtered `silver.product_model_product_description` as `pmpd` on `p.product_model_id = pmpd.product_model_id`
  - left join `silver.product_description` as `pd` on `pmpd.product_description_id = pd.product_description_id`
- Self-join rule for category:
  - `pc_child.name` is the product subcategory
  - `pc_parent.name` is the parent category
  - do not reference unqualified `name` after either category join
- Culture filter rule:
  - filter `pmpd` to English only using normalized `culture = 'EN'`
  - expose output `culture` as lowercase `'en'`
- Duplicate prevention rule:
  - if multiple English description rows exist for the same `product_model_id`, keep exactly one deterministic row using lowest `product_description_id`
  - preserve the product row with null description if no English mapping exists
- Output columns:
  - `product_id`
  - `name`
  - `product_number`
  - `color`
  - `size`
  - `weight`
  - `standard_cost`
  - `list_price`
  - `product_model_id`
  - `product_model_name` from `pm.name`
  - `product_category_id`
  - `category` from `pc_parent.name`
  - `subcategory` from `pc_child.name`
  - `culture`
  - `description`
  - `sell_start_date`
  - `sell_end_date`
  - `discontinued_date`

### gold.dim_sales
- Base grain: one row per cleaned `sales_person`.
- Required inputs:
  - `silver.sales_order_header`: `sales_order_id`, `customer_id`, `sales_order_number`, `order_date`, `sales_person`
- Source rule:
  - use salesperson from `silver.sales_order_header`, not from `silver.customer`
  - the only allowed upstream column name in Gold is `sales_person`
- Cleaning rule for salesperson:
  - start from `sales_person`
  - remove leading prefix `adventure-works\` if present
  - also remove leading prefix `adventureworks/` if present
  - remove trailing digits at the end of the string
  - trim the final value
- Null/blank handling:
  - exclude rows where cleaned `sales_person` is null or blank from `dim_sales`
- Output columns:
  - `sales_person`

### gold.fact_sales
- Base grain: one row per sales order line, keyed by (`sales_order_id`, `sales_order_detail_id`).
- Required inputs:
  - `silver.sales_order_detail`: `sales_order_id`, `sales_order_detail_id`, `order_qty`, `product_id`, `unit_price`, `unit_price_discount`, `line_total`, `gross_line_amount`, `discount_amount`
  - `silver.sales_order_header`: `sales_order_id`, `revision_number`, `order_date`, `due_date`, `ship_date`, `status`, `online_order_flag`, `sales_order_number`, `purchase_order_number`, `account_number`, `customer_id`, `ship_to_address_id`, `bill_to_address_id`, `ship_method`, `sub_total`, `tax_amt`, `freight`, `total_due`, `comment`, `order_date_key`, `due_date_key`, `ship_date_key`, `sales_person`
- Join path:
  - start from `silver.sales_order_detail` as `sod`
  - inner join `silver.sales_order_header` as `soh` on `sod.sales_order_id = soh.sales_order_id`
- Salesperson rule:
  - create `sales_person` in `fact_sales` from cleaned `soh.sales_person` using the exact same logic as `gold.dim_sales`
  - do not reference any alternative column spelling such as `salesperson` in Gold
- Output columns:
  - `sales_order_id`
  - `sales_order_detail_id`
  - `sales_order_number`
  - `revision_number`
  - `order_date`
  - `due_date`
  - `ship_date`
  - `order_date_key`
  - `due_date_key`
  - `ship_date_key`
  - `status`
  - `online_order_flag`
  - `customer_id`
  - `product_id`
  - `sales_person`
  - `purchase_order_number`
  - `account_number`
  - `ship_method`
  - `ship_to_address_id`
  - `bill_to_address_id`
  - `comment`
  - `order_qty`
  - `unit_price`
  - `unit_price_discount`
  - `gross_line_amount`
  - `discount_amount`
  - `line_total`
  - `sub_total`
  - `tax_amt`
  - `freight`
  - `total_due`

## Semantic model
Create a Direct Lake semantic model over the Gold tables produced from the requested design:
- `dim_customer`
- `dim_product`
- `dim_sales`
- `fact_sales`

Use a star schema centered on `fact_sales`, with supporting dimensions for customer, product, and salesperson, plus a date dimension derived from fact date keys for time intelligence.

### Semantic model tables
- `dim_date`
- `dim_customer`
- `dim_product`
- `dim_sales`
- `fact_sales`

### Relationships
- `fact_sales[customer_id] -> dim_customer[customer_id]`
- `fact_sales[product_id] -> dim_product[product_id]`
- `fact_sales[sales_person] -> dim_sales[sales_person]`
- `fact_sales[order_date_key] -> dim_date[date_key]` active
- `fact_sales[due_date_key] -> dim_date[date_key]` inactive
- `fact_sales[ship_date_key] -> dim_date[date_key]` inactive

### Modeling notes
- `dim_customer` should expose the customer grain only, already filtered in Gold to the `Main Office` address type where present.
- `dim_customer` should include meaningful descriptive attributes from customer and address, such as:
  - `customer_id`
  - `full_name`
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
- `dim_product` should expose the product grain with enriched descriptive attributes from product, product category self-join, product model, and English product description, such as:
  - `product_id`
  - `name`
  - `product_number`
  - `color`
  - `size`
  - `weight`
  - `standard_cost`
  - `list_price`
  - `product_model_id`
  - `product_model_name`
  - `product_category_id`
  - `category`
  - `subcategory`
  - `culture`
  - `description`
  - `sell_start_date`
  - `sell_end_date`
  - `discontinued_date`
- `dim_sales` should contain the cleaned salesperson domain from Gold, using the transformed salesperson value with `adventureworks/` removed and trailing numeric suffix removed.
- `fact_sales` should be the primary report-facing fact and include relevant joined header/detail fields, including:
  - order identifiers: `sales_order_id`, `sales_order_detail_id`, `sales_order_number`
  - date keys/dates: `order_date`, `due_date`, `ship_date`, `order_date_key`, `due_date_key`, `ship_date_key`
  - customer/product/salesperson keys: `customer_id`, `product_id`, `sales_person`
  - shipping/billing/order descriptors: `status`, `online_order_flag`, `purchase_order_number`, `account_number`, `ship_method`, `ship_to_address_id`, `bill_to_address_id`, `comment`
  - numeric measures: `order_qty`, `unit_price`, `unit_price_discount`, `gross_line_amount`, `discount_amount`, `line_total`, `sub_total`, `tax_amt`, `freight`, `total_due`
- Hide audit columns, hashes, lineage columns, and technical helper columns from report view.
- If `dim_sales` contains one row per cleaned salesperson, ensure the relationship is one-to-many from `dim_sales` to `fact_sales`.

### Explicit measures
Propose these initial DAX measures:
- `Total Sales = SUM(fact_sales[line_total])`
- `Total Orders = DISTINCTCOUNT(fact_sales[sales_order_id])`
- `Total Order Lines = DISTINCTCOUNT(fact_sales[sales_order_detail_id])`
- `Total Quantity = SUM(fact_sales[order_qty])`
- `Gross Sales = SUM(fact_sales[gross_line_amount])`
- `Total Discount Amount = SUM(fact_sales[discount_amount])`
- `Average Unit Price = AVERAGE(fact_sales[unit_price])`
- `Average Order Value = DIVIDE(SUM(fact_sales[total_due]), DISTINCTCOUNT(fact_sales[sales_order_id]))`
- `Total Freight = SUM(fact_sales[freight])`
- `Total Tax = SUM(fact_sales[tax_amt])`
- `Products Sold = DISTINCTCOUNT(fact_sales[product_id])`
- `Customers Buying = DISTINCTCOUNT(fact_sales[customer_id])`

Optional time-intelligence measures after `dim_date` is marked:
- `Sales YTD`
- `Sales YoY`
- `Orders MTD`

### Semantic model UX recommendations
- Format `Total Sales`, `Gross Sales`, `Total Discount Amount`, `Average Order Value`, `Total Freight`, and `Total Tax` as currency.
- Mark `dim_date` as the date table using `date_key` / full date column.
- Sort month name by month number in `dim_date`.
- Use friendly display names:
  - `dim_customer` → `Dim Customer`
  - `dim_product` → `Dim Product`
  - `dim_sales` → `Dim Sales`
  - `fact_sales` → `Fact Sales`
- Hide foreign key columns from dimensions where duplicate descriptive fields are already exposed, but keep business keys available if users need drillthrough.
- Set default summarization to `Do not summarize` for IDs and descriptive text columns.

## Report
Build a Power BI report tailored to the final Gold model with `fact_sales` as the main analytical table and dimensions `dim_customer`, `dim_product`, and `dim_sales`.

### Page 1: Sales Overview
Visuals:
- KPI cards:
  - `Total Sales`
  - `Total Orders`
  - `Average Order Value`
  - `Total Quantity`
- Line chart:
  - `Total Sales` by `dim_date[month]` and year
- Clustered bar chart:
  - `Total Sales` by `dim_product[category]`
- Clustered bar chart:
  - `Total Sales` by `dim_sales[sales_person]`
- Donut chart:
  - `Total Sales` by `fact_sales[online_order_flag]`
- Matrix:
  - `dim_customer[company_name]` or `dim_customer[full_name]` by `Total Sales`, `Total Orders`, `Total Quantity`

Slicers:
- order date
- category
- subcategory
- salesperson
- customer
- ship method
- online order flag

### Page 2: Product Analysis
Visuals:
- Bar chart:
  - top products by `Total Sales` using `dim_product[name]`
- Bar chart:
  - `Total Sales` by `dim_product[subcategory]`
- Treemap:
  - `category` > `subcategory` sized by `Total Sales`
- Scatter plot:
  - `Gross Sales` vs `Total Discount Amount` by `dim_product[name]`
- Table:
  - `dim_product[name]`, `product_number`, `category`, `subcategory`, `product_model_name`, `color`, `size`, `list_price`, with `Total Sales` and `Total Quantity`
- Optional detail visual:
  - English product `description` for selected product

Slicers:
- category
- subcategory
- product model
- color

### Page 3: Customer & Geography
Visuals:
- Bar chart:
  - top customers by `Total Sales`
- Map or filled map if data quality is sufficient:
  - `Total Sales` by `dim_customer[country_region]`, `state_province`, `city`
- Matrix:
  - geography hierarchy `country_region` > `state_province` > `city` with `Total Sales`, `Total Orders`, `Customers Buying`
- Table:
  - `full_name`, `company_name`, `email_address`, `phone`, `address_line1`, `city`, `state_province`, `country_region`, `postal_code`, `Total Sales`
- Card or slicer note:
  - indicate that customer address data reflects only `Main Office` address type from Gold

Slicers:
- country_region
- state_province
- city
- customer

### Page 4: Salesperson & Order Detail
Visuals:
- Bar chart:
  - `Total Sales` by `dim_sales[sales_person]`
- Column chart:
  - `Total Orders` by `fact_sales[status]`
- Column chart:
  - `Total Sales` by `fact_sales[ship_method]`
- Trend chart:
  - `Total Orders` by order date
- Table:
  - order-level/line-level detail using `sales_order_number`, `sales_order_id`, `sales_order_detail_id`, `order_date`, customer, salesperson, product, `order_qty`, `unit_price`, `line_total`, `tax_amt`, `freight`, `total_due`
- Optional decomposition tree:
  - `Total Sales` by salesperson > category > subcategory > customer

Slicers:
- salesperson
- status
- ship method
- order date

### Page 5: Data Quality
Visuals:
- Card:
  - Gold row count snapshots for `dim_customer`, `dim_product`, `dim_sales`, `fact_sales`
- Table:
  - duplicate-key exceptions by Gold table
- Table:
  - referential integrity failures for:
    - `fact_sales.customer_id` not in `dim_customer`
    - `fact_sales.product_id` not in `dim_product`
    - `fact_sales.sales_person` not in `dim_sales`
- Card set:
  - null key violations
  - orphan fact rows
  - missing `Main Office` customer address coverage
  - missing English (`en`) product description coverage
- Trend or snapshot visual:
  - test outcomes by run id / load timestamp

## Data Agent
Create an AISkill grounded on the semantic model.

### Role
- Sales analytics assistant for the `SalesLT` domain using the final Gold model centered on `fact_sales`.
- Helps business users answer questions about sales trends, product performance, customer purchasing, salesperson performance, shipping patterns, and core data quality status using only the published semantic model.

### Domain hints
- The central fact table is `fact_sales`, built from joined sales order detail and sales order header data.
- Customer analysis should use `dim_customer`, which already includes customer and address information and is limited to the `Main Office` address type in Gold.
- Product analysis should use `dim_product`, which already includes:
  - product attributes
  - parent `category`
  - child `subcategory`
  - English (`en`) product description/model enrichment
- Salesperson analysis should use `dim_sales`, where salesperson names have been cleaned by removing the `adventureworks/` prefix and trailing numeric suffix.
- Sales amounts for most analysis should use line-level measures such as `Total Sales`, `Gross Sales`, and `Total Discount Amount`.
- Order-level charges such as freight and tax come from the joined fields on `fact_sales`.
- Time analysis should use `dim_date` with the active order date relationship unless the user explicitly asks for due date or ship date logic.

### Starter questions
- What are total sales and total orders by month?
- Which categories and subcategories generate the most sales?
- Who are the top customers by sales amount?
- Which salespeople generate the highest sales?
- How much discount are we giving by product or subcategory?
- What is the average order value over time?
- How do online orders compare with offline orders?
- Which ship methods are associated with the highest sales?
- What are sales by country, state, and city based on Main Office addresses?
- Which English-described products have the highest quantity sold?
- Are there any orphan fact rows or missing customer/product/salesperson keys?

### Guardrails
- Use only the semantic model tables and explicit measures; do not invent fields not present in the model.
- Prefer `fact_sales` measures for sales analysis.
- Use `dim_customer` for customer and geography questions, and remember it reflects only `Main Office` address records where available.
- Use `dim_product[category]` and `dim_product[subcategory]` for product hierarchy questions rather than inventing other hierarchies.
- Use cleaned salesperson values from `dim_sales`; do not reintroduce the raw `adventureworks/` prefix or numeric suffix.
- Do not expose hidden technical columns, hashes, or audit fields unless the user explicitly asks for data quality metadata.
- Do not reveal Bronze-only sensitive fields such as password-related columns.
- If a user asks for unsupported concepts such as returns, inventory, margin, profit, or payments beyond available columns, state that the model does not currently include those data points.
- If a user asks for non-English product descriptions, state that the modeled product description is filtered to English (`en`) in Gold.
- When comparing totals, clarify whether the user wants line-level sales (`line_total` / `Total Sales`) or header-derived order totals (`total_due` / `Average Order Value`).
- If the user asks for customer addresses beyond `Main Office`, state that the current Gold dimension was intentionally limited to that address type.