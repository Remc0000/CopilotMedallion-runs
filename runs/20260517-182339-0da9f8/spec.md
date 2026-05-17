# Run Spec 20260517-181731-482a3f

## Updated specs

### Iteration 1 — 2026-05-17 18:31:59Z — failed layer: gold (run: 20260517-182339-0da9f8)
- **Root cause (1-line summary)**: Gold build was underspecified for the multi-join dimension/fact shaping, likely causing statement failures from unresolved or ambiguous columns across customer/address, product/category self-join, and header/detail joins.
- **What was changed**:
  - Tightened `## Generic guidance` with explicit Gold join/alias rules for self-joins and multi-table joins.
  - Rewrote `## Gold` to define exact source tables, required join keys, required filters, canonical output column names, and dedup/grain rules for `dim_customer`, `dim_product`, `dim_sales`, and `fact_sales`.
  - Clarified salesperson cleaning logic and category self-join output naming to prevent ambiguous references.

### Iteration 2 — 2026-05-17 18:41:39Z — failed layer: gold (run: 20260517-182339-0da9f8)
- **Root cause (1-line summary)**: Gold build still allowed non-deterministic/multi-step statements that could fail session-wide; the product and customer dimension joins need explicit staged projections and mandatory pre-checks on exact Silver columns before any Gold write.
- **What was changed**:
  - Tightened `## Generic guidance` with a mandatory staged Gold build pattern: pre-validate columns, create one intermediate projection per join path, and write each Gold table independently with no chained SQL statements.
  - Sharpened `## Gold` with exact required Silver column names, explicit intermediate join paths, deterministic row-reduction rules, and a prohibition on using any non-specified columns in `dim_customer`, `dim_product`, `dim_sales`, and `fact_sales`.
  - Added explicit fallback behavior for missing optional descriptive columns so the build omits them rather than guessing alternate names or failing the session.

### Iteration 3 — 2026-05-17 18:47:02Z — failed layer: silver (run: 20260517-182339-0da9f8)
- **Root cause (1-line summary)**: Silver build likely submitted multiple dependent statements in one Spark session, so one table failure cancelled the entire session without enough table-specific shape validation.
- **What was changed**:
  - Tightened `## Generic guidance` to require isolated Silver builds with one table per validation/transform/write sequence and no chained SQL scripts or shared temp-view dependency chains across Silver tables.
  - Sharpened `## Silver` with mandatory exact snake_case column mappings for all base tables used downstream and explicit instructions to fail fast per table when required columns are missing.
  - Added Silver output-shape constraints for `customer`, `customer_address`, `product`, `product_category`, `product_description`, `product_model`, `product_model_product_description`, `sales_order_header`, and `sales_order_detail` so downstream Gold-required columns are guaranteed.

## Inputs
- Workspace: `842742ca-c3dd-46d6-b034-f4d7cddc8d5c`
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
- Target Lakehouse: **CopilotMedallion_20260517_181731**

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
- Use defensive column references everywhere: verify expected columns exist before select, rename, join, filter, aggregate, or merge logic.
- After every join, immediately project/select alias-prefixed columns to avoid ambiguous names and accidental column shadowing.
- Before every `groupBy` / `agg`, assert that all grouping and aggregation columns exist.
- For REST/API handling, use defensive null checks before `.get()` access: `if x is None: raise ...`.
- Do not use `saveAsTable`; write with Lakehouse-supported idempotent patterns such as overwrite to managed Delta paths/tables through supported APIs.
- Put parameters in dedicated parameter cells at the top of every notebook.
- Use idempotent overwrite patterns for full loads; if incremental logic is later added, make merge keys explicit and deterministic.
- Wrap notebook body steps in error-loud `try/except`; on exception call `_save_error(layer, e)` and re-raise.
- Add audit and lineage columns consistently across layers.
- Exclude binary payloads from Gold and semantic model unless there is a specific reporting need.
- For Gold multi-table joins, never use `select("*")` after joins. Always explicitly select the final output columns by alias-qualified source column names.
- For any self-join, especially `silver.product_category`, use distinct aliases and materialize renamed columns immediately after the join. Do not reference duplicated column names without aliases.
- For Gold builds, validate the presence of these exact Silver columns before transformation: `customer_id`, `address_id`, `address_type`, `product_id`, `product_category_id`, `parent_product_category_id`, `product_model_id`, `product_description_id`, `culture`, `sales_order_id`, `sales_order_detail_id`, and `sales_person`.
- If an expected descriptive column is absent in a helper/view path, fall back to the normalized base-table path defined in the spec rather than guessing alternate column names.
- Gold notebooks must build each Gold table in an isolated step: one validation step, one transformation step, one write step, then move to the next table. Do not submit one chained SQL script or a long sequence of dependent temp views that can cancel the whole Spark session on first failure.
- For each Gold table, create a narrow intermediate dataframe immediately after each join path and keep only the explicitly required columns for the next step. Do not carry unused columns forward.
- If an optional output column is not available from the exact specified Silver source column, omit that output column from the Gold table rather than inferring a substitute column name.
- Before writing any Gold table, assert both conditions: expected output columns exist and the dataframe grain matches the spec's key uniqueness rule.
- Silver notebooks must build each Silver table in an isolated step: one source-read validation step, one rename/type/cleaning step, one dedup step, one write step. Do not execute a single multi-statement SQL script or a long dependency chain across Silver tables in one session.
- For Silver, fail fast per table with a clear error if any required source column for that table is missing after Bronze read; do not continue to subsequent Silver tables after a required-column failure in the current table.
- For Silver base tables that feed Gold, do not guess column names from views or alternates. Use the exact Bronze source columns defined in the Silver table specification below and emit the exact snake_case output column names defined there.

Notebook authoring rule for every generated notebook:
- EACH code cell must start with a short markdown comment block using Python comments.
- Never emit a code cell with no leading comment.
- Pattern to follow:
  - `# ---`
  - `# <what this cell is doing>`
  - `# <why this step exists>`

## Bronze
Land each source object into a `bronze` schema as a raw, minimally transformed Delta table. Use full overwrite for this initial build because no reliable watermark column exists across all objects, though `ModifiedDate` can support future incremental patterns for many base tables.

Common Bronze design:
- Schema: `bronze`
- Load pattern: full snapshot overwrite per run
- Metadata columns added to every table:
  - `_ingest_run_id`
  - `_ingest_ts`
  - `_source_workspace_id`
  - `_source_lakehouse_id`
  - `_source_table`
  - `_record_hash` from non-metadata columns for change detection
- Partitioning:
  - Do not partition small reference tables.
  - Partition larger transactional tables by year/month derived from relevant business dates where available.
- Raw column preservation:
  - Preserve source column names and types in Bronze.
  - Keep `rowguid` as string.
  - Keep binary `ThumbNailPhoto` only in Bronze/Silver.

Bronze table plan:
- `bronze.address`
  - Source: `SalesLT/Address`
  - Natural key: `AddressID`
  - Partitioning: none
  - Notes: good master-data table for customer/order addresses.
- `bronze.customer`
  - Source: `SalesLT/Customer`
  - Natural key: `CustomerID`
  - Partitioning: none
  - Notes: contains PII (`EmailAddress`, `Phone`) and credential-like fields (`PasswordHash`, `PasswordSalt`); retain only because present in source, but these should be restricted and excluded from Gold.
- `bronze.customer_address`
  - Source: `SalesLT/CustomerAddress`
  - Natural composite key: (`CustomerID`, `AddressID`, `AddressType`) or (`CustomerID`, `AddressID`) if type is unstable
  - Partitioning: none
  - Notes: bridge/junction between customers and addresses.
- `bronze.product`
  - Source: `SalesLT/Product`
  - Natural key: `ProductID`
  - Partitioning: none
  - Notes: includes commercial attributes and lifecycle dates.
- `bronze.product_category`
  - Source: `SalesLT/ProductCategory`
  - Natural key: `ProductCategoryID`
  - Partitioning: none
  - Notes: self-referencing hierarchy through `ParentProductCategoryID`.
- `bronze.product_description`
  - Source: `SalesLT/ProductDescription`
  - Natural key: `ProductDescriptionID`
  - Partitioning: none
- `bronze.product_model`
  - Source: `SalesLT/ProductModel`
  - Natural key: `ProductModelID`
  - Partitioning: none
- `bronze.product_model_product_description`
  - Source: `SalesLT/ProductModelProductDescription`
  - Natural composite key: (`ProductModelID`, `ProductDescriptionID`, `Culture`)
  - Partitioning: none
  - Notes: multilingual bridge for model descriptions.
- `bronze.sales_order_header`
  - Source: `SalesLT/SalesOrderHeader`
  - Natural key: `SalesOrderID`
  - Partitioning: `year(OrderDate)`, `month(OrderDate)`
  - Notes: header-level transactional fact candidate.
- `bronze.sales_order_detail`
  - Source: `SalesLT/SalesOrderDetail`
  - Natural composite key: (`SalesOrderID`, `SalesOrderDetailID`)
  - Partitioning: derive `order_year`, `order_month` by joining header in Silver or keep unpartitioned in Bronze
  - Notes: line-level transaction fact candidate.
- `bronze.v_get_all_categories`
  - Source: `SalesLT/vGetAllCategories`
  - Natural key: `ProductCategoryID`
  - Partitioning: none
  - Notes: view appears to flatten category hierarchy; useful for Gold enrichment, but still land raw.
- `bronze.v_product_and_description`
  - Source: `SalesLT/vProductAndDescription`
  - Natural composite key: (`ProductID`, `Culture`) if multiple cultures exist; else `ProductID`
  - Partitioning: none
  - Notes: convenience view for product description by culture.
- `bronze.v_product_model_catalog_description`
  - Source: `SalesLT/vProductModelCatalogDescription`
  - Natural key: `ProductModelID`
  - Partitioning: none
  - Notes: denormalized model/catalog attributes, likely text-heavy.

## Silver
Standard Silver objectives:
- Rename columns to `snake_case`.
- Enforce datatypes explicitly.
- Remove exact duplicates.
- Deduplicate by business/natural key using latest `modified_date` where present, else deterministic hash/order fallback.
- Add audit columns:
  - `created_at_utc`
  - `updated_at_utc`
  - `source_table`
  - `ingest_run_id`
- Standardize null handling, trim strings, normalize booleans, and derive lightweight helper columns.
- Run `OPTIMIZE` after load for larger tables; focus on `silver.sales_order_header`, `silver.sales_order_detail`, and `silver.product`.
- Build and write each Silver table independently; no shared temp-view chain across multiple Silver outputs.
- For each Silver table below, first validate the exact Bronze source columns named in the mapping/spec. If any required source column is missing, fail that table immediately with a clear error that names the missing column(s) and table.

Silver table specifications:

### `silver.address`
Required Bronze source: `bronze.address`

Required source-to-target mappings:
- `AddressID` -> `address_id`
- `AddressLine1` -> `address_line1`
- `AddressLine2` -> `address_line2`
- `City` -> `city`
- `StateProvince` -> `state_province`
- `CountryRegion` -> `country_region`
- `PostalCode` -> `postal_code`
- `ModifiedDate` -> `modified_date`

Optional passthrough mappings if present:
- `rowguid` -> `rowguid`

Required output columns:
- `address_id`, `address_line1`, `address_line2`, `city`, `state_province`, `country_region`, `postal_code`, `modified_date`
- plus standard Silver audit columns

Dedup key: `address_id`
Dedup rule: keep latest `modified_date`, tie-break on `_ingest_ts`
Cleaning:
- trim all string columns
- standardize empty strings to null for `address_line2`
- uppercase or title-case location fields only if business approves; default is preserve original values

### `silver.customer`
Required Bronze source: `bronze.customer`

Required source-to-target mappings:
- `CustomerID` -> `customer_id`
- `NameStyle` -> `name_style`
- `Title` -> `title`
- `FirstName` -> `first_name`
- `MiddleName` -> `middle_name`
- `LastName` -> `last_name`
- `Suffix` -> `suffix`
- `CompanyName` -> `company_name`
- `EmailAddress` -> `email_address`
- `Phone` -> `phone`
- `PasswordHash` -> `password_hash`
- `PasswordSalt` -> `password_salt`
- `ModifiedDate` -> `modified_date`

Optional passthrough mappings if present:
- `SalesPerson` -> `sales_person`
- `rowguid` -> `rowguid`

Required output columns:
- `customer_id`, `name_style`, `title`, `first_name`, `middle_name`, `last_name`, `suffix`, `company_name`, `modified_date`
- derived `full_name`
- plus standard Silver audit columns

Optional output columns if exact sources exist:
- `email_address`, `phone`, `password_hash`, `password_salt`, `sales_person`

Dedup key: `customer_id`
Dedup rule: keep latest `modified_date`
Cleaning:
- derive `full_name` from available title/first/middle/last/suffix without overwriting source components
- trim `email_address`, lower-case email for matching helper column `email_address_normalized`
- keep `password_hash` and `password_salt` in Silver only if strictly needed downstream; default recommendation is to retain in Silver restricted zone but exclude from Gold
Sensitive data note:
- Gold should not expose `password_hash`, `password_salt`, possibly not even raw email/phone unless required

### `silver.customer_address`
Required Bronze source: `bronze.customer_address`

Required source-to-target mappings:
- `CustomerID` -> `customer_id`
- `AddressID` -> `address_id`
- `AddressType` -> `address_type`
- `ModifiedDate` -> `modified_date`

Required output columns:
- `customer_id`, `address_id`, `address_type`, `modified_date`
- plus standard Silver audit columns

Dedup key: (`customer_id`, `address_id`, `address_type`)
Alternative if duplicate address types appear unexpectedly: dedup on (`customer_id`, `address_id`) and keep latest `modified_date`
Cleaning:
- trim `address_type`
- validate foreign keys against `silver.customer.customer_id` and `silver.address.address_id`

### `silver.product`
Required Bronze source: `bronze.product`

Required source-to-target mappings:
- `ProductID` -> `product_id`
- `Name` -> `name`
- `ProductNumber` -> `product_number`
- `Color` -> `color`
- `Size` -> `size`
- `StandardCost` -> `standard_cost`
- `ListPrice` -> `list_price`
- `ProductCategoryID` -> `product_category_id`
- `ProductModelID` -> `product_model_id`
- `SellStartDate` -> `sell_start_date`
- `SellEndDate` -> `sell_end_date`
- `DiscontinuedDate` -> `discontinued_date`
- `ThumbNailPhoto` -> `thumbnail_photo`
- `ThumbnailPhotoFileName` -> `thumbnail_photo_file_name`
- `ModifiedDate` -> `modified_date`

Required output columns:
- `product_id`, `name`, `product_number`, `standard_cost`, `list_price`, `product_category_id`, `product_model_id`, `sell_start_date`, `sell_end_date`, `discontinued_date`, `modified_date`
- derived `is_discontinued`, `is_currently_sellable`
- plus standard Silver audit columns

Optional output columns if exact sources exist:
- `color`, `size`, `thumbnail_photo`, `thumbnail_photo_file_name`

Dedup key: `product_id`
Dedup rule: keep latest `modified_date`
Cleaning:
- standardize text attributes: `name`, `product_number`, `color`, `size`, `thumbnail_photo_file_name`
- derive flags:
  - `is_discontinued` from `discontinued_date is not null`
  - `is_currently_sellable` from sell start/end/discontinued dates
- keep `thumbnail_photo` in Silver but exclude from Gold and semantic model
FK expectations:
- `product_category_id` -> `silver.product_category.product_category_id`
- `product_model_id` -> `silver.product_model.product_model_id`

### `silver.product_category`
Required Bronze source: `bronze.product_category`

Required source-to-target mappings:
- `ProductCategoryID` -> `product_category_id`
- `ParentProductCategoryID` -> `parent_product_category_id`
- `Name` -> `name`
- `ModifiedDate` -> `modified_date`

Required output columns:
- `product_category_id`, `parent_product_category_id`, `name`, `modified_date`
- plus standard Silver audit columns

Dedup key: `product_category_id`
Dedup rule: keep latest `modified_date`
Cleaning:
- preserve `parent_product_category_id` for hierarchy
- derive hierarchy helper later in Gold

### `silver.product_description`
Required Bronze source: `bronze.product_description`

Required source-to-target mappings:
- `ProductDescriptionID` -> `product_description_id`
- `Description` -> `description`
- `ModifiedDate` -> `modified_date`

Required output columns:
- `product_description_id`, `description`, `modified_date`
- plus standard Silver audit columns

Dedup key: `product_description_id`
Dedup rule: keep latest `modified_date`
Cleaning:
- trim `description`

### `silver.product_model`
Required Bronze source: `bronze.product_model`

Required source-to-target mappings:
- `ProductModelID` -> `product_model_id`
- `Name` -> `name`
- `CatalogDescription` -> `catalog_description`
- `ModifiedDate` -> `modified_date`

Required output columns:
- `product_model_id`, `name`, `modified_date`
- plus standard Silver audit columns

Optional output columns if exact source exists:
- `catalog_description`

Dedup key: `product_model_id`
Dedup rule: keep latest `modified_date`
Cleaning:
- retain `catalog_description` as large text
- trim `name`

### `silver.product_model_product_description`
Required Bronze source: `bronze.product_model_product_description`

Required source-to-target mappings:
- `ProductModelID` -> `product_model_id`
- `ProductDescriptionID` -> `product_description_id`
- `Culture` -> `culture`
- `ModifiedDate` -> `modified_date`

Required output columns:
- `product_model_id`, `product_description_id`, `culture`, `modified_date`
- plus standard Silver audit columns

Dedup key: (`product_model_id`, `product_description_id`, `culture`)
Dedup rule: keep latest `modified_date`
Cleaning:
- normalize `culture` to lower-case
- validate FKs to `product_model` and `product_description`

### `silver.sales_order_header`
Required Bronze source: `bronze.sales_order_header`

Required source-to-target mappings:
- `SalesOrderID` -> `sales_order_id`
- `SalesOrderNumber` -> `sales_order_number`
- `PurchaseOrderNumber` -> `purchase_order_number`
- `AccountNumber` -> `account_number`
- `CustomerID` -> `customer_id`
- `SalesPerson` -> `sales_person`
- `OrderDate` -> `order_date`
- `DueDate` -> `due_date`
- `ShipDate` -> `ship_date`
- `ShipMethod` -> `ship_method`
- `BillToAddressID` -> `bill_to_address_id`
- `ShipToAddressID` -> `ship_to_address_id`
- `OnlineOrderFlag` -> `online_order_flag`
- `SubTotal` -> `sub_total`
- `TaxAmt` -> `tax_amt`
- `Freight` -> `freight`
- `TotalDue` -> `total_due`
- `ModifiedDate` -> `modified_date`

Required output columns:
- `sales_order_id`, `sales_order_number`, `purchase_order_number`, `account_number`, `customer_id`, `sales_person`, `order_date`, `due_date`, `ship_date`, `ship_method`, `bill_to_address_id`, `ship_to_address_id`, `online_order_flag`, `sub_total`, `tax_amt`, `freight`, `total_due`, `modified_date`
- derived `order_date_key`, `due_date_key`, `ship_date_key`, `order_year`, `order_month`
- plus standard Silver audit columns

Dedup key: `sales_order_id`
Dedup rule: keep latest `modified_date`
Cleaning:
- derive `order_date_key`, `due_date_key`, `ship_date_key` as `yyyyMMdd` integers
- derive `order_year`, `order_month`
- normalize `online_order_flag`
- trim `sales_order_number`, `purchase_order_number`, `account_number`, `ship_method`
FK expectations:
- `customer_id` -> `silver.customer.customer_id`
- `ship_to_address_id` -> `silver.address.address_id`
- `bill_to_address_id` -> `silver.address.address_id`

### `silver.sales_order_detail`
Required Bronze source: `bronze.sales_order_detail`

Required source-to-target mappings:
- `SalesOrderID` -> `sales_order_id`
- `SalesOrderDetailID` -> `sales_order_detail_id`
- `ProductID` -> `product_id`
- `OrderQty` -> `order_qty`
- `UnitPrice` -> `unit_price`
- `UnitPriceDiscount` -> `unit_price_discount`
- `LineTotal` -> `line_total`
- `ModifiedDate` -> `modified_date`

Required output columns:
- `sales_order_id`, `sales_order_detail_id`, `product_id`, `order_qty`, `unit_price`, `unit_price_discount`, `line_total`, `modified_date`
- derived `net_unit_price`
- plus standard Silver audit columns

Dedup key: (`sales_order_id`, `sales_order_detail_id`)
Dedup rule: keep latest `modified_date`
Cleaning:
- cast numeric fields explicitly
- derive `net_unit_price = unit_price - unit_price_discount`
- validate FKs:
  - `sales_order_id` -> `silver.sales_order_header.sales_order_id`
  - `product_id` -> `silver.product.product_id`

### `silver.v_get_all_categories`
- Dedup key: `product_category_id`
- Use as optional helper table only.
- Cleaning:
  - rename to snake_case
- Modeling note:
  - Because this is a view over category data, prefer base tables as system-of-record and use this only to simplify category rollups if values reconcile.

### `silver.v_product_and_description`
- Dedup key: (`product_id`, `culture`) preferred
- Use as optional helper table only.
- Modeling note:
  - Prefer normalized base path `product -> product_model -> product_model_product_description -> product_description` when multilingual detail is needed.
  - Use this view if it materially simplifies description retrieval and reconciles row counts.

### `silver.v_product_model_catalog_description`
- Dedup key: `product_model_id`
- Use as optional helper table only.
- Cleaning:
  - snake_case all descriptive attributes
- Modeling note:
  - This view is useful for rich product model attributes, but because it overlaps with `product_model`, treat it as enrichment rather than primary source.

## Gold
Materialize exactly four Gold tables in schema `gold`: `dim_customer`, `dim_product`, `dim_sales`, and `fact_sales`.

General Gold rules:
- Use Silver tables only as Gold inputs.
- Use explicit aliases for every table in every join.
- Do not rely on helper views for Gold if base-table joins satisfy the requirement; prefer normalized Silver base tables.
- Immediately project final columns after joins and assign canonical output names defined below.
- Gold tables must be idempotent full-overwrite builds.
- Exclude audit/lineage columns unless specifically needed for reporting.
- Exclude `password_hash`, `password_salt`, binary columns, and any accidental duplicate technical columns.
- Build each Gold table as its own isolated dataframe pipeline and write independently.
- Before building each Gold table, validate that the exact required source columns named below exist; if a required column is missing, fail fast with a clear error naming that column and table.
- Do not reference any Silver column not explicitly named in the table specification below.
- Use dataframe transforms or SQL CTEs with one named stage per join path; do not embed all joins, filters, deduplication, and final projection in one statement.

### `gold.dim_customer`
Business intent:
- One row per `customer_id`.
- Enrich customer with the `Main Office` address only.

Required source tables and aliases:
- `silver.customer` as `c`
- `silver.customer_address` as `ca`
- `silver.address` as `a`

Required source columns:
- From `c`: `customer_id`, `full_name`, `company_name`, `name_style`, `title`, `first_name`, `middle_name`, `last_name`, `suffix`
- Optional from `c`: `email_address`, `phone`
- From `ca`: `customer_id`, `address_id`, `address_type`
- Optional from `ca`: `modified_date`
- From `a`: `address_id`, `address_line1`, `address_line2`, `city`, `state_province`, `country_region`, `postal_code`
- Optional from `a`: `modified_date`

Required staged logic:
1. Create `ca_main_office` from `ca` filtered to `address_type = 'Main Office'` and projected to only `customer_id`, `address_id`, `address_type`, optional `modified_date`.
2. If multiple `Main Office` rows exist for the same `customer_id`, reduce `ca_main_office` to one row per `customer_id` using deterministic precedence:
   1. latest `ca.modified_date` if present
   2. latest `a.modified_date` after the address join if needed
   3. lowest `address_id`
3. Left join `c.customer_id = ca_main_office.customer_id`.
4. Left join `ca_main_office.address_id = a.address_id`.
5. Immediately project only the canonical output columns listed below.

Canonical output columns:
- `customer_id` from `c.customer_id`
- `full_name` from `c.full_name`
- `company_name` from `c.company_name`
- `name_style` from `c.name_style`
- `customer_name_style` from `c.name_style`
- `title` from `c.title`
- `first_name` from `c.first_name`
- `middle_name` from `c.middle_name`
- `last_name` from `c.last_name`
- `suffix` from `c.suffix`
- `email_address` from `c.email_address` only if that exact column exists and is intentionally exposed; otherwise omit
- `phone` from `c.phone` only if that exact column exists and is intentionally exposed; otherwise omit
- `address_id` from `a.address_id`
- `address_type` from `ca_main_office.address_type`
- `address_line1` from `a.address_line1`
- `address_line2` from `a.address_line2`
- `city` from `a.city`
- `state_province` from `a.state_province`
- `country_region` from `a.country_region`
- `postal_code` from `a.postal_code`

Explicit constraint:
- Final table grain must be exactly one row per `customer_id`.

### `gold.dim_product`
Business intent:
- One row per `product_id`.
- Enrich product with product model, English description, and category/subcategory from the product-category self-join.

Required source tables and aliases:
- `silver.product` as `p`
- `silver.product_category` as `pc_child`
- `silver.product_category` as `pc_parent`
- `silver.product_model` as `pm`
- `silver.product_model_product_description` as `pmpd`
- `silver.product_description` as `pd`

Required source columns:
- From `p`: `product_id`, `name`, `product_number`, `color`, `size`, `standard_cost`, `list_price`, `sell_start_date`, `sell_end_date`, `discontinued_date`, `is_discontinued`, `is_currently_sellable`, `product_category_id`, `product_model_id`
- From `pc_child`: `product_category_id`, `parent_product_category_id`, `name`
- From `pc_parent`: `product_category_id`, `name`
- From `pm`: `product_model_id`, `name`
- From `pmpd`: `product_model_id`, `product_description_id`, `culture`
- From `pd`: `product_description_id`, `description`
- Optional timestamps for deterministic ranking: `pmpd.modified_date`, `pd.modified_date`

Required staged logic:
1. Create `product_category_stage` starting from `p` and left joining `pc_child` on `p.product_category_id = pc_child.product_category_id`.
2. Self-join `pc_parent` on `pc_child.parent_product_category_id = pc_parent.product_category_id`.
3. Immediately project this stage to:
   - `product_id`
   - `product_name`
   - `product_number`
   - `color`
   - `size`
   - `standard_cost`
   - `list_price`
   - `sell_start_date`
   - `sell_end_date`
   - `discontinued_date`
   - `is_discontinued`
   - `is_currently_sellable`
   - `product_model_id`
   - `product_category_id` from `pc_child.product_category_id`
   - `parent_product_category_id` from `pc_parent.product_category_id`
   - `subcategory_name` from `pc_child.name`
   - `category_name` from `pc_parent.name`
4. Create `product_description_stage` from `p`, `pm`, `pmpd`, and `pd` only:
   - left join `pm` on `p.product_model_id = pm.product_model_id`
   - left join `pmpd` on `p.product_model_id = pmpd.product_model_id AND pmpd.culture = 'en'`
   - left join `pd` on `pmpd.product_description_id = pd.product_description_id`
5. Reduce `product_description_stage` to one row per `product_id` using deterministic precedence:
   1. `pmpd.culture = 'en'`
   2. lowest `pmpd.product_description_id`
   3. latest `pd.modified_date` if present
6. Project `product_description_stage` to only:
   - `product_id`
   - `product_model_id` from `pm.product_model_id`
   - `product_model_name` from `pm.name`
   - `product_description_id` from `pd.product_description_id`
   - `product_description` from `pd.description`
   - `culture` from `pmpd.culture`
7. Join `product_category_stage.product_id = product_description_stage.product_id`.
8. Immediately project only the canonical output columns below.

Category naming rule:
- `subcategory_name` = `pc_child.name`
- `category_name` = `pc_parent.name`
- `parent_product_category_id` = `pc_parent.product_category_id`
- `product_category_id` = `pc_child.product_category_id`
- If a product is directly assigned to a top-level category with no parent:
  - keep `subcategory_name = pc_child.name`
  - allow `category_name` to be null

Description rule:
- Use only the normalized path through `product_model_product_description` and `product_description`.
- Do not use `silver.v_product_and_description` in Gold for this table.
- If no English description exists, retain the product row with null `product_description_id`, `product_description`, and `culture`.

Canonical output columns:
- `product_id`
- `product_name`
- `product_number`
- `color`
- `size`
- `standard_cost`
- `list_price`
- `sell_start_date`
- `sell_end_date`
- `discontinued_date`
- `is_discontinued`
- `is_currently_sellable`
- `product_model_id`
- `product_model_name`
- `product_description_id`
- `product_description`
- `culture`
- `product_category_id`
- `parent_product_category_id`
- `subcategory_name`
- `category_name`

Explicit constraint:
- Final table grain must be exactly one row per `product_id`.

### `gold.dim_sales`
Business intent:
- One row per cleaned salesperson value derived from `silver.sales_order_header.sales_person`.

Required source table:
- `silver.sales_order_header` as `soh`

Required source columns:
- `sales_person`

Cleaning rule for salesperson:
- Source column is exactly `soh.sales_person`.
- Derive `sales_person` as:
  1. remove leading `adventureworks/` if present
  2. remove trailing digits at the end of the string
  3. trim whitespace
- Preserve null when the source salesperson is null or empty after cleaning.

Canonical output columns:
- `sales_person` as cleaned value
- optionally `sales_person_raw` from `soh.sales_person` for validation only; hide from semantic model if retained

Dedup rule:
- Keep one row per cleaned `sales_person`
- Exclude null cleaned salesperson rows from `dim_sales`

Explicit constraint:
- Relationship key for the semantic model is the cleaned `sales_person`, not the raw source string.

### `gold.fact_sales`
Business intent:
- Line-grain fact built from sales order detail joined to sales order header.

Required source tables and aliases:
- `silver.sales_order_detail` as `sod`
- `silver.sales_order_header` as `soh`

Required source columns:
- From `sod`: `sales_order_id`, `sales_order_detail_id`, `product_id`, `order_qty`, `unit_price`, `unit_price_discount`, `net_unit_price`, `line_total`
- From `soh`: `sales_order_id`, `customer_id`, `sales_person`, `order_date`, `due_date`, `ship_date`, `order_date_key`, `due_date_key`, `ship_date_key`, `sales_order_number`, `purchase_order_number`, `account_number`, `ship_method`, `online_order_flag`, `sub_total`, `tax_amt`, `freight`, `total_due`

Required staged logic:
1. Start from `sod` as the driving table.
2. Left join `soh` on `sod.sales_order_id = soh.sales_order_id`.
3. Derive cleaned `sales_person` using the exact same logic as `gold.dim_sales` from `soh.sales_person`.
4. Immediately project only the canonical output columns below.
5. Assert uniqueness of (`sales_order_id`, `sales_order_detail_id`) before write.

Canonical output columns:
- `sales_order_id` from `sod.sales_order_id`
- `sales_order_detail_id` from `sod.sales_order_detail_id`
- `customer_id` from `soh.customer_id`
- `product_id` from `sod.product_id`
- `sales_person` as cleaned value from `soh.sales_person`
- `order_date` from `soh.order_date`
- `due_date` from `soh.due_date`
- `ship_date` from `soh.ship_date`
- `order_date_key` from `soh.order_date_key`
- `due_date_key` from `soh.due_date_key`
- `ship_date_key` from `soh.ship_date_key`
- `sales_order_number` from `soh.sales_order_number`
- `purchase_order_number` from `soh.purchase_order_number`
- `account_number` from `soh.account_number`
- `ship_method` from `soh.ship_method`
- `online_order_flag` from `soh.online_order_flag`
- `order_qty` from `sod.order_qty`
- `unit_price` from `sod.unit_price`
- `unit_price_discount` from `sod.unit_price_discount`
- `net_unit_price` from `sod.net_unit_price`
- `line_total` from `sod.line_total`
- `sub_total` from `soh.sub_total`
- `tax_amt` from `soh.tax_amt`
- `freight` from `soh.freight`
- `total_due` from `soh.total_due`

Explicit constraint:
- Final table grain must be exactly one row per (`sales_order_id`, `sales_order_detail_id`).
- Do not aggregate during Gold fact creation.
- Header totals (`sub_total`, `tax_amt`, `freight`, `total_due`) are expected to repeat across detail rows.

## Semantic model
Publish a Direct Lake semantic model over the Gold star schema exactly as defined by the edited Gold layer.

### Tables
- `dim_customer`
- `dim_product`
- `dim_sales`
- `fact_sales`

### Relationships
- `fact_sales[customer_id]` -> `dim_customer[customer_id]`
- `fact_sales[product_id]` -> `dim_product[product_id]`
- `fact_sales[sales_person]` -> `dim_sales[sales_person]`

Relationship notes:
- Use single-direction filtering from dimensions to `fact_sales`.
- `dim_sales[sales_person]` must contain the cleaned salesperson value derived from `sales_order_header.sales_person`, with the `adventureworks/` prefix removed and trailing numeric suffix removed, so it matches the cleaned `fact_sales[sales_person]`.
- Do not introduce separate address, product category, product model, order header, or date dimension tables unless they are physically materialized in Gold later.

### Table design notes
- `dim_customer` should expose the customer grain from Gold, with one row per `customer_id`, enriched from customer + customer_address filtered to `address_type = 'Main Office'` + address.
- `dim_product` should expose one row per `product_id`, enriched from product + product_category self-join + product_model + product_model_product_description filtered to `culture = 'en'` + product_description.
- `dim_product` should include both category levels:
  - `category_name` for the parent category
  - `subcategory_name` for the child/subcategory
- `dim_sales` should expose the cleaned salesperson dimension and any other meaningful salesperson-related attributes carried into Gold.
- `fact_sales` should remain the primary reporting fact and should contain joined order header + detail columns needed for measures and slicing.

### Recommended exposed columns
Use actual Gold columns where available. Prefer the following business columns if present after Gold shaping.

- `dim_customer`
  - `customer_id`
  - `full_name`
  - `company_name`
  - `customer_name_style`
  - `title`
  - `first_name`
  - `middle_name`
  - `last_name`
  - `suffix`
  - `address_type`
  - `address_line1`
  - `address_line2`
  - `city`
  - `state_province`
  - `country_region`
  - `postal_code`

- `dim_product`
  - `product_id`
  - `product_name`
  - `product_number`
  - `color`
  - `size`
  - `standard_cost`
  - `list_price`
  - `sell_start_date`
  - `sell_end_date`
  - `discontinued_date`
  - `product_model_id`
  - `product_model_name`
  - `product_description`
  - `culture`
  - `product_category_id`
  - `parent_product_category_id`
  - `category_name`
  - `subcategory_name`

- `dim_sales`
  - `sales_person`
  - optional descriptive columns from Gold if available

- `fact_sales`
  - `sales_order_id`
  - `sales_order_detail_id`
  - `customer_id`
  - `product_id`
  - `sales_person`
  - `order_date`
  - `due_date`
  - `ship_date`
  - `sales_order_number`
  - `purchase_order_number`
  - `account_number`
  - `ship_method`
  - `online_order_flag`
  - `order_qty`
  - `unit_price`
  - `unit_price_discount`
  - `line_total`
  - `sub_total`
  - `tax_amt`
  - `freight`
  - `total_due`

### Explicit measures
Create measures only from fields implied by the Gold design.

- `Total Sales = SUM(fact_sales[total_due])`
- `Subtotal Sales = SUM(fact_sales[sub_total])`
- `Total Tax = SUM(fact_sales[tax_amt])`
- `Total Freight = SUM(fact_sales[freight])`
- `Total Line Sales = SUM(fact_sales[line_total])`
- `Total Order Quantity = SUM(fact_sales[order_qty])`
- `Order Count = DISTINCTCOUNT(fact_sales[sales_order_id])`
- `Average Order Value = DIVIDE([Total Sales], [Order Count])`
- `Average Selling Price = DIVIDE([Total Line Sales], [Total Order Quantity])`
- `Discount Amount = SUMX(fact_sales, fact_sales[unit_price_discount] * fact_sales[order_qty])`
- `Online Order Count = CALCULATE([Order Count], fact_sales[online_order_flag] = TRUE())`

### Modeling notes
- Hide audit, lineage, and technical columns.
- Keep only the cleaned salesperson value visible to report authors and the Data Agent.
- Exclude restricted/sensitive fields from the semantic model, especially `password_hash`, `password_salt`, binary columns, and any raw PII not intentionally surfaced in Gold.
- Because `fact_sales` is a header-detail joined fact, document clearly that header totals such as `total_due`, `sub_total`, `tax_amt`, and `freight` may repeat across line rows. Report authors should use the provided measures carefully and avoid mixing repeated header totals with line-grain visuals unless the Gold build deduplicates those amounts appropriately.
- If a proper date dimension is added to Gold later, extend the semantic model with it; for now use `fact_sales[order_date]` and other date columns directly.

## Report
Create a Power BI report aligned to the four Gold tables: `dim_customer`, `dim_product`, `dim_sales`, and `fact_sales`.

### Page 1: Sales overview
Visuals:
- KPI cards:
  - `Total Sales`
  - `Order Count`
  - `Average Order Value`
  - `Total Order Quantity`
- Line chart:
  - `Total Sales` by `fact_sales[order_date]`
- Clustered column chart:
  - `Total Line Sales` by `dim_product[category_name]`
- Stacked column chart:
  - `Total Line Sales` by `dim_product[category_name]` and `dim_product[subcategory_name]`
- Donut chart:
  - `Order Count` by `fact_sales[online_order_flag]`
- Bar chart:
  - `Total Sales` by `dim_sales[sales_person]`

### Page 2: Product analysis
Visuals:
- Bar chart:
  - top products by `Total Line Sales` using `dim_product[product_name]`
- Matrix:
  - rows: `dim_product[category_name]`, `dim_product[subcategory_name]`, `dim_product[product_name]`
  - values: `Total Line Sales`, `Total Order Quantity`, `Average Selling Price`
- Scatter chart:
  - `dim_product[list_price]` vs `dim_product[standard_cost]`
  - size: `Total Order Quantity`
  - details: `dim_product[product_name]`
- Table:
  - `product_name`, `product_number`, `product_model_name`, `product_description`, `color`, `size`, `list_price`, `standard_cost`
- Slicers:
  - `dim_product[category_name]`
  - `dim_product[subcategory_name]`
  - `dim_product[color]`
  - `dim_product[size]`

### Page 3: Customer analysis
Visuals:
- Bar chart:
  - top customers by `Total Sales` using `dim_customer[full_name]` or `dim_customer[company_name]`
- Table:
  - `dim_customer[customer_id]`
  - `dim_customer[full_name]`
  - `dim_customer[company_name]`
  - `dim_customer[address_type]`
  - `dim_customer[city]`
  - `dim_customer[state_province]`
  - `dim_customer[country_region]`
  - `Total Sales`
  - `Order Count`
  - `Average Order Value`
- Map or filled map:
  - location from `dim_customer[country_region]`, `dim_customer[state_province]`, `dim_customer[city]`
  - value: `Total Sales`
- Column chart:
  - `Order Count` by `dim_customer[country_region]`
- Slicers:
  - `fact_sales[order_date]`
  - `dim_customer[country_region]`
  - `dim_customer[state_province]`

### Page 4: Salesperson performance
Visuals:
- Bar chart:
  - `Total Sales` by `dim_sales[sales_person]`
- Column chart:
  - `Order Count` by `dim_sales[sales_person]`
- Table:
  - `dim_sales[sales_person]`, `Total Sales`, `Order Count`, `Average Order Value`, `Total Order Quantity`
- Matrix:
  - rows: `dim_sales[sales_person]`
  - columns: `dim_product[category_name]`
  - values: `Total Line Sales`
- Slicers:
  - `fact_sales[order_date]`
  - `dim_product[category_name]`
  - `fact_sales[ship_method]`

### Authoring notes
- Prefer `Total Line Sales` for product/category visuals because product performance is naturally line-grain.
- Use `Total Sales` and `Order Count` for order-level summaries, with validation that the Gold fact handles header totals correctly at the joined grain.
- Use the cleaned `dim_sales[sales_person]` field everywhere; do not expose the raw `adventureworks/...` salesperson values.
- Do not use fields excluded from Gold such as password or binary columns.
- Keep report filters synchronized across pages for `order_date`, `sales_person`, `category_name`, and geography where useful.

## Data Agent
Create an AISkill grounded on the Direct Lake semantic model built over `dim_customer`, `dim_product`, `dim_sales`, and `fact_sales`.

### Role
You are a sales analytics copilot for customer, product, category, subcategory, salesperson, and order analysis based on the Gold sales model. Answer business questions using the semantic model only. Prefer certified measures and use the cleaned salesperson field from `dim_sales`.

### Domain hints
- `fact_sales` is the central fact table and contains joined sales order header and sales order detail information.
- `dim_customer` represents customers enriched with their `Main Office` address only.
- `dim_product` represents products enriched with product model, English description (`culture = 'en'`), and category hierarchy.
- `dim_product[category_name]` is the parent category.
- `dim_product[subcategory_name]` is the child category/subcategory from the product category self-join.
- `dim_sales[sales_person]` is a cleaned salesperson value derived from the source by removing the `adventureworks/` prefix and the trailing numeric suffix.
- Use `Total Line Sales` and `Total Order Quantity` for product mix and product/category analysis.
- Use `Total Sales`, `Subtotal Sales`, `Total Tax`, `Total Freight`, and `Order Count` for order-level analysis, subject to the joined-fact grain rules documented in the model.

### Starter questions
- What were total sales and order count by month?
- Which salespersons generated the highest total sales?
- Which parent categories and subcategories generated the most line sales?
- Who are the top 10 customers by total sales?
- Which products sold the highest quantity?
- What is the average order value by salesperson?
- How do sales break down by ship method?
- Which countries or states have the highest customer sales?
- Which English product descriptions correspond to the best-selling products?
- Which discontinued products still appear in sales history?

### Guardrails
- Use only semantic-model tables and measures; do not fabricate fields.
- Prefer explicit measures over implicit aggregation.
- Distinguish carefully between order-level totals and line-level sales because `fact_sales` is built from joined header and detail data.
- If a user asks for product or category performance, prefer `Total Line Sales` and `Total Order Quantity`.
- If a user asks for salesperson analysis, use `dim_sales[sales_person]`, not raw source strings.
- If a user asks about customer address, answer using the `Main Office` address context only, because that is how `dim_customer` is defined in Gold.
- Do not expose restricted fields such as password hashes, salts, binary images, or raw PII not included in the semantic model.
- Do not estimate profit, margin, or customer lifetime value unless such measures are explicitly added.
- If a question depends on fields not present in Gold or the semantic model, say so clearly and ask for a model extension instead of guessing.