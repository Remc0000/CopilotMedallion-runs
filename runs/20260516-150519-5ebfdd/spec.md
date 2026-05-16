# Run Spec 20260516-145843-ade58d

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
- Binary handling:
  - `SalesLT/Product.ThumbNailPhoto` should be retained in bronze only.
  - Do not propagate `ThumbNailPhoto` to silver/gold unless the user later requests image analytics.

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
- `SalesLT/vProductAndDescription` -> `bronze.v_product_and_description`
  - Treat as source view snapshot.
- `SalesLT/vProductModelCatalogDescription` -> `bronze.v_product_model_catalog_description`
  - Treat as source view snapshot.

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
- `silver.v_product_and_description`
  - Source: `bronze.v_product_and_description`
  - Dedup key: `product_id`, `culture`
  - Useful for multilingual/descriptive enrichments
- `silver.v_product_model_catalog_description`
  - Source: `bronze.v_product_model_catalog_description`
  - Dedup key: `product_model_id`
  - Useful for product attributes not present in base tables

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

### Proposed gold dimensions
- `gold.dim_customer`
  - Grain: one row per `customer_id`
  - Columns:
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
    - `modified_date`
  - Optional enrichments:
    - default billing/shipping indicators via `customer_address` if a rule is later defined
- `gold.dim_address`
  - Grain: one row per `address_id`
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
  - Grain: one row per `product_id`
  - Base from `silver.product`
  - Enrich with:
    - category names from `silver.product_category` or `silver.v_get_all_categories`
    - model attributes from `silver.product_model` and `silver.v_product_model_catalog_description`
    - description from `silver.v_product_and_description` where `culture` is available
  - Columns:
    - `product_id`
    - `product_name` from source `name`
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
    - `parent_category_name`
    - `product_model_name`
    - `culture`
    - `product_description`
    - optional model attributes: `manufacturer`, `material`, `product_line`, `style`, `rider_experience`
  - Preferred rule: choose one culture for primary description if multiple exist; default to `'en'` if present, else first non-null
  - Alternative: keep descriptions in a separate multilingual dimension if multilingual reporting is required
- `gold.dim_date`
  - Grain: one row per date
  - Built from min/max of `order_date`, `due_date`, `ship_date`, `sell_start_date`, `sell_end_date`, `discontinued_date`
  - Standard attributes: date_key, full_date, year, quarter, month, month_name, week, day, weekday, is_month_end

### Proposed gold facts
- `gold.fact_sales_order_line`
  - Grain: one row per `sales_order_detail_id`
  - Source: `silver.sales_order_detail` joined to `silver.sales_order_header`
  - Keys:
    - `sales_order_detail_id`
    - `sales_order_id`
    - `order_date_key`
    - `due_date_key`
    - `ship_date_key`
    - `customer_id`
    - `product_id`
    - `ship_to_address_id`
    - `bill_to_address_id`
  - Degenerate dimensions:
    - `sales_order_number`
    - `purchase_order_number`
    - `account_number`
    - `ship_method`
    - `credit_card_approval_code`
    - `status`
    - `online_order_flag`
  - Measures/analytics columns:
    - `order_qty`
    - `unit_price`
    - `unit_price_discount`
    - `line_total`
    - allocated or repeated order-level amounts:
      - `header_sub_total`
      - `header_tax_amt`
      - `header_freight`
      - `header_total_due`
  - Preferred note: keep header totals repeated at line grain only for convenience, but measures must avoid summing repeated header totals without distinct-order logic
- `gold.fact_sales_order_header`
  - Grain: one row per `sales_order_id`
  - Source: `silver.sales_order_header`
  - Keys:
    - `sales_order_id`
    - `order_date_key`
    - `due_date_key`
    - `ship_date_key`
    - `customer_id`
    - `ship_to_address_id`
    - `bill_to_address_id`
  - Measures:
    - `sub_total`
    - `tax_amt`
    - `freight`
    - `total_due`
  - Use case:
    - clean order-level reporting without double counting
- `gold.bridge_customer_address`
  - Grain: one row per `customer_id`, `address_id`, `address_type`
  - Use for customer-address analysis outside transactional facts

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
- If `dim_customer` contains repeated rows because of multiple customer addresses, do not use it as the only customer lookup on facts unless gold enforces one row per `customer_id`.
- Preferred semantic-model handling for customer-address enrichment:
  - Keep `dim_customer` at one row per `customer_id`
  - Use `dim_address` plus fact ship/bill relationships for geography
  - Expose `bridge_customer_address` as hidden helper only if needed for advanced customer-address analysis
- If the gold implementation keeps multiple address rows in `dim_customer`, then create a separate report-facing customer table or constrain the visible fields to avoid ambiguous filtering.
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
- If `dim_customer` includes multiple rows per customer because of joined addresses, avoid visuals that assume a unique customer grain unless the gold table enforces one row per `customer_id`.
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
- `dim_customer` is enriched from customer, customer-address, and address data, so customer views may include address-related attributes such as `address_type`, `city`, `state_province`, `country_region`, and `postal_code`.
- Orders come from `fact_sales_order_header`; order lines come from `fact_sales_order_line`.
- Products are analyzed through `dim_product`, which combines product, product category hierarchy, product model, and description data.
- Product hierarchy should use:
  - `parent_category_name` as the higher level
  - `category_name` as the lower category/subcategory level
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
- When users ask for category performance, use the hierarchy `parent_category_name` and `category_name` correctly.
- If a question is ambiguous about date or geography role, ask whether to use order date vs ship date, or shipping vs billing address.
- If customer geography can come from either enriched customer-address fields or shipping/billing address relationships, clarify which one the user wants when needed.
- If multilingual product descriptions exist, clarify which culture to analyze when relevant.
- If a requested metric is not modeled, say so explicitly and suggest the nearest available measure.
- Surface data quality caveats when latest DQ tests show failures.
- Avoid unsupported causal claims; describe observed patterns only.
