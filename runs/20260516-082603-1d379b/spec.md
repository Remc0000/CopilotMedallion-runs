# Run Spec 20260516-081758-967a7c

## Updated specs

### Iteration 1 — 2026-05-16 08:33:34Z — failed layer: reporting (run: 20260516-082603-1d379b)
- **Root cause (1-line summary)**: Reporting step failed while creating the semantic model because the spec allowed multiple conflicting semantic-model shapes and role-playing options, leaving the model definition underspecified.
- **What was changed**:
  - Tightened **Gold** to use exactly one order-level table in addition to the line fact: `gold.dim_sales_order` instead of a second fact table.
  - Tightened **Semantic model** to a single required table list, explicit relationship endpoints, and required role-played address tables so model creation is deterministic.
  - Tightened **Report** and **Data Agent** to reference the finalized semantic model objects and measures only.

## Inputs
- Workspace: `24bf4b77-62a2-4cd3-ba8f-da166af94f79`
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
- Target Lakehouse: **e2egpt41**

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
- Use defensive column references: validate required columns exist before select, join, groupBy, agg, dedup, or rename steps.
- After every join, project columns explicitly using alias-prefixed references to avoid ambiguity.
- Before every groupBy/agg, assert grouped and aggregated columns exist.
- For REST/API handling, use defensive null checks: `if x is None: raise ...` before any `.get()` access.
- Do not use `saveAsTable`; write to lakehouse paths/tables using supported Fabric patterns.
- Put all environment-specific values in parameter cells at the top of each notebook/script.
- Use idempotent overwrite/upsert patterns so reruns are safe.
- Implement error-loud `try/except` blocks that call `_save_error(layer, e)` and then re-raise.
- Add ingestion metadata columns consistently across Bronze/Silver/Gold where appropriate.
- Prefer deterministic dedup logic using business keys plus `modified_date` or ingestion timestamp.
- Optimize Delta tables after major writes; where useful, partition only when there is a clear selective date column and enough volume.
- For semantic model creation, do not leave optional/alternative table names or relationship shapes unresolved at build time. The model definition must reference only tables and columns that are explicitly required by this spec and actually produced in Gold.

## Bronze
Land each selected source object into the `bronze` schema as raw-conformed Delta tables with minimal transformation.

### Bronze ingestion standards
- Source-to-target naming:
  - `SalesLT/Address` -> `bronze.address`
  - `SalesLT/Customer` -> `bronze.customer`
  - `SalesLT/CustomerAddress` -> `bronze.customer_address`
  - `SalesLT/Product` -> `bronze.product`
  - `SalesLT/ProductCategory` -> `bronze.product_category`
  - `SalesLT/ProductDescription` -> `bronze.product_description`
  - `SalesLT/ProductModel` -> `bronze.product_model`
  - `SalesLT/ProductModelProductDescription` -> `bronze.product_model_product_description`
  - `SalesLT/SalesOrderDetail` -> `bronze.sales_order_detail`
  - `SalesLT/SalesOrderHeader` -> `bronze.sales_order_header`
  - `SalesLT/vGetAllCategories` -> `bronze.v_get_all_categories`
  - `SalesLT/vProductAndDescription` -> `bronze.v_product_and_description`
  - `SalesLT/vProductModelCatalogDescription` -> `bronze.v_product_model_catalog_description`
- Write mode:
  - Full snapshot overwrite for all listed tables/views unless incremental source extraction is later added.
  - Because source includes views, treat all objects uniformly as snapshot loads.
- Metadata columns to append to every Bronze table:
  - `bronze_ingested_at` timestamp
  - `bronze_run_id` string
  - `bronze_source_workspace_id` string
  - `bronze_source_lakehouse_id` string
  - `bronze_source_object` string
  - `bronze_record_hash` string over all business columns for change detection
- Schema handling:
  - Preserve source column names and types as-is in Bronze.
  - Keep binary `ThumbNailPhoto` in Bronze only; do not expose it downstream unless explicitly needed.
- Partitioning:
  - No partitioning for small/master tables: `address`, `customer`, `customer_address`, `product_category`, `product_description`, `product_model`, `product_model_product_description`, and all three view-based objects.
  - Candidate partitioning for `sales_order_header` on `OrderDate` month if volume warrants.
  - Candidate partitioning for `sales_order_detail` only if later enriched with header order date; otherwise no partitioning in Bronze.

### Table-specific landing notes
- `bronze.address`
  - Land all columns including `rowguid`, `ModifiedDate`.
  - Expected PK candidate: `AddressID`.
- `bronze.customer`
  - Land sensitive columns `PasswordHash` and `PasswordSalt` in Bronze only for source fidelity.
  - Expected PK candidate: `CustomerID`.
- `bronze.customer_address`
  - Land as junction/satellite-style raw table.
  - Composite key candidate: `CustomerID`, `AddressID`, `AddressType`.
- `bronze.product`
  - Retain all columns including `ThumbNailPhoto`, `ThumbnailPhotoFileName`.
  - Expected PK candidate: `ProductID`.
- `bronze.product_category`
  - Preserve parent-child hierarchy via `ParentProductCategoryID`.
  - Expected PK candidate: `ProductCategoryID`.
- `bronze.product_description`
  - Expected PK candidate: `ProductDescriptionID`.
- `bronze.product_model`
  - Expected PK candidate: `ProductModelID`.
- `bronze.product_model_product_description`
  - Composite key candidate: `ProductModelID`, `ProductDescriptionID`, `Culture`.
- `bronze.sales_order_header`
  - Transaction header table; likely order-level fact source.
  - Expected PK candidate: `SalesOrderID`.
- `bronze.sales_order_detail`
  - Transaction line table; strongest line-level fact source.
  - Composite key candidate: `SalesOrderID`, `SalesOrderDetailID`.
- `bronze.v_get_all_categories`
  - Treat as helper/denormalized hierarchy lookup.
- `bronze.v_product_and_description`
  - Treat as helper/denormalized product-description lookup by `ProductID` and `Culture`.
- `bronze.v_product_model_catalog_description`
  - Treat as helper/extended product model attributes by `ProductModelID`.

## Silver
Create cleaned, typed, deduplicated, snake_case-conformed tables in the `silver` schema.

### Silver standards
- Convert all column names to `snake_case`.
- Preserve source keys as natural keys; do not create surrogate keys in Silver.
- Append audit columns:
  - `silver_loaded_at`
  - `silver_run_id`
  - `record_source`
  - `is_current` for snapshot current-row semantics if needed
- Remove obvious technical noise from downstream analytics:
  - Exclude `password_hash` and `password_salt` from Silver customer by default.
  - Exclude `thumbnail_photo` from Silver product by default.
- Standardize timestamps to timestamp type and decimals to consistent precision.
- Deduplicate using business key + latest `modified_date`, then latest `bronze_ingested_at`.
- Run `OPTIMIZE` on all Silver tables after load.

### silver.address
- Source: `bronze.address`
- Rename examples:
  - `AddressID` -> `address_id`
  - `AddressLine1` -> `address_line1`
  - `StateProvince` -> `state_province`
  - `CountryRegion` -> `country_region`
  - `PostalCode` -> `postal_code`
  - `ModifiedDate` -> `modified_date`
- Dedup key:
  - `address_id`
- Cleaning:
  - Trim text fields.
  - Normalize blank `address_line2` to null.
- Notes:
  - Good candidate dimension source.

### silver.customer
- Source: `bronze.customer`
- Dedup key:
  - `customer_id`
- Cleaning:
  - Build standardized full name fields from `title`, `first_name`, `middle_name`, `last_name`, `suffix`.
  - Lowercase/trim `email_address`.
  - Normalize phone formatting only if rules are agreed.
  - Drop `password_hash`, `password_salt` from Silver for least-privilege analytics.
- Modeling note:
  - This is clearly a dimension/master-data table.

### silver.customer_address
- Source: `bronze.customer_address`
- Dedup key:
  - `customer_id`, `address_id`, `address_type`
- Cleaning:
  - Upper/lower standardization for `address_type`.
- Modeling note:
  - Junction table connecting customers to addresses; can support billing/shipping role analysis.
  - Alternative: rely only on `sales_order_header.ship_to_address_id` and `sales_order_header.bill_to_address_id` for order analytics and keep this table for customer master views.

### silver.product
- Source: `bronze.product`
- Dedup key:
  - `product_id`
- Cleaning:
  - Drop `thumbnail_photo`; retain `thumbnail_photo_file_name`.
  - Standardize nullable descriptive columns: `color`, `size`.
  - Preserve lifecycle dates: `sell_start_date`, `sell_end_date`, `discontinued_date`.
- Modeling note:
  - Dimension/master-data table with links to category and model.

### silver.product_category
- Source: `bronze.product_category`
- Dedup key:
  - `product_category_id`
- Cleaning:
  - Preserve parent-child hierarchy.
- Modeling note:
  - Dimension table; hierarchy can be flattened later.

### silver.product_description
- Source: `bronze.product_description`
- Dedup key:
  - `product_description_id`

### silver.product_model
- Source: `bronze.product_model`
- Dedup key:
  - `product_model_id`
- Cleaning:
  - `catalog_description` likely semi-structured text/XML-like payload; keep as text.

### silver.product_model_product_description
- Source: `bronze.product_model_product_description`
- Dedup key:
  - `product_model_id`, `product_description_id`, `culture`
- Modeling note:
  - Bridge table between model and multilingual descriptions.

### silver.sales_order_header
- Source: `bronze.sales_order_header`
- Dedup key:
  - `sales_order_id`
- Cleaning:
  - Standardize `sales_order_number`, `purchase_order_number`, `account_number`, `ship_method`.
  - Preserve all financials: `sub_total`, `tax_amt`, `freight`, `total_due`.
  - Preserve status fields: `status`, `online_order_flag`.
- Modeling note:
  - Strong order-header fact candidate.
  - Date columns support role-playing dates: order, due, ship.

### silver.sales_order_detail
- Source: `bronze.sales_order_detail`
- Dedup key:
  - `sales_order_id`, `sales_order_detail_id`
- Cleaning:
  - Preserve line economics: `order_qty`, `unit_price`, `unit_price_discount`, `line_total`.
- Modeling note:
  - Strongest transactional fact source at line grain.

### silver.v_get_all_categories
- Source: `bronze.v_get_all_categories`
- Dedup key:
  - `product_category_id`
- Use:
  - Flatten category hierarchy into parent/child labels for downstream dimensions.

### silver.v_product_and_description
- Source: `bronze.v_product_and_description`
- Dedup key:
  - `product_id`, `culture`
- Use:
  - Enrich product dimension with `product_model`, `description`, `culture`.
- Required decision:
  - Gold must filter this table to English only: `culture = 'en'` when present.
  - After filtering to English, Gold `dim_product` must retain at most one row per `product_id`.

### silver.v_product_model_catalog_description
- Source: `bronze.v_product_model_catalog_description`
- Dedup key:
  - `product_model_id`
- Use:
  - Extended product model attributes such as `manufacturer`, `material`, `style`, `rider_experience`, `product_line`.
- User-editable decision:
  - If sparsity is high, keep as a snowflaked dimension; otherwise denormalize into a wide product dimension.

## Gold
Model a sales-oriented star schema because the actual tables show clear transaction headers/details plus customer, product, category, and address masters.

### Modeling route selected
- Primary route:
  - Line-level sales fact centered on `sales_order_detail`, enriched by `sales_order_header`.
  - Dimensions from customer, product, address, date, and sales order.
- Required modeling choice:
  - Do not create `gold.fact_sales_order`.
  - Create `gold.dim_sales_order` instead, as the extra dimension for order number and order-level descriptive attributes.
- Product description choice:
  - Use English only in Gold. Filter `silver.v_product_and_description` to `culture = 'en'` where the column exists.
  - Do not create a multilingual dimension.

### Fact tables

#### gold.fact_sales_order_line
- Grain:
  - One row per `sales_order_id` + `sales_order_detail_id`
- Built from:
  - `silver.sales_order_detail` as `d`
  - left join `silver.sales_order_header` as `h` on `d.sales_order_id = h.sales_order_id`
- Natural transaction keys retained:
  - `sales_order_id`
  - `sales_order_detail_id`
- Foreign keys:
  - `sales_order_id`
  - `customer_id`
  - `product_id`
  - `bill_to_address_id`
  - `ship_to_address_id`
  - `order_date_key`
  - `due_date_key`
  - `ship_date_key`
- Required output columns:
  - From detail:
    - `sales_order_id`
    - `sales_order_detail_id`
    - `product_id`
    - `order_qty`
    - `unit_price`
    - `unit_price_discount`
    - `line_total`
  - From header:
    - `customer_id`
    - `bill_to_address_id`
    - `ship_to_address_id`
    - `online_order_flag`
    - `order_date`
    - `due_date`
    - `ship_date`
    - `sub_total`
    - `tax_amt`
    - `freight`
    - `total_due`
- Derived columns:
  - `order_date_key = cast(date_format(order_date, 'yyyyMMdd') as int)`
  - `due_date_key = cast(date_format(due_date, 'yyyyMMdd') as int)` when `due_date` is not null
  - `ship_date_key = cast(date_format(ship_date, 'yyyyMMdd') as int)` when `ship_date` is not null
  - `gross_line_amount = order_qty * unit_price`
  - `discount_amount = order_qty * unit_price * unit_price_discount`
  - `net_line_amount = line_total`
  - `is_online_order = online_order_flag`
- Modeling caution:
  - Header totals (`sub_total`, `tax_amt`, `freight`, `total_due`) repeat across lines and must be hidden in the semantic model to avoid accidental aggregation from line grain.
  - Use alias-prefixed projection after the join; do not rely on `select("*")`.

### Dimension tables

#### gold.dim_sales_order
- Source:
  - `silver.sales_order_header` as `h`
- Key:
  - `sales_order_id`
- Required attributes:
  - `sales_order_number`
  - `purchase_order_number`
  - `account_number`
  - `revision_number`
  - `status`
  - `ship_method`
  - `customer_id`
  - `bill_to_address_id`
  - `ship_to_address_id`
  - `online_order_flag`
  - `order_date`
  - `due_date`
  - `ship_date`
  - `sub_total`
  - `tax_amt`
  - `freight`
  - `total_due`
  - `order_date_key`
  - `due_date_key`
  - `ship_date_key`
- Derived columns:
  - `order_date_key = cast(date_format(order_date, 'yyyyMMdd') as int)`
  - `due_date_key = cast(date_format(due_date, 'yyyyMMdd') as int)` when `due_date` is not null
  - `ship_date_key = cast(date_format(ship_date, 'yyyyMMdd') as int)` when `ship_date` is not null
  - `is_online_order = online_order_flag`
- Notes:
  - This is a conformed order dimension-like helper table for order number and order-level attributes, not a fact table.
  - Keep exactly one row per `sales_order_id`.

#### gold.dim_customer
- Source:
  - `silver.customer`
- Key:
  - `customer_id`
- Attributes:
  - `title`, `first_name`, `middle_name`, `last_name`, `suffix`
  - `company_name`
  - `sales_person`
  - `email_address`
  - `phone`
  - `name_style`
  - `modified_date`
- Derived:
  - `customer_full_name`
  - `customer_display_name` using company when person name is sparse
- Notes:
  - No sensitive password fields.

#### gold.dim_address
- Source:
  - `silver.address`
- Key:
  - `address_id`
- Attributes:
  - `address_line1`, `address_line2`, `city`, `state_province`, `country_region`, `postal_code`
- Use:
  - Base address dimension used to derive the two role-played address dimensions below.

#### gold.dim_ship_to_address
- Source:
  - `gold.dim_address`
- Key:
  - `address_id`
- Attributes:
  - `address_line1`, `address_line2`, `city`, `state_province`, `country_region`, `postal_code`
- Notes:
  - Physical duplicate/role-played table required for simpler semantic-model relationships.

#### gold.dim_bill_to_address
- Source:
  - `gold.dim_address`
- Key:
  - `address_id`
- Attributes:
  - `address_line1`, `address_line2`, `city`, `state_province`, `country_region`, `postal_code`
- Notes:
  - Physical duplicate/role-played table required for simpler semantic-model relationships.

#### gold.dim_product
- Source:
  - `silver.product` as `p`
  - left join `silver.product_category` as `pc` on `p.product_category_id = pc.product_category_id`
  - left join `silver.v_get_all_categories` as `c` on `p.product_category_id = c.product_category_id`
  - left join `silver.product_model` as `pm` on `p.product_model_id = pm.product_model_id`
  - left join filtered English-only `silver.v_product_and_description` as `vpd_en` on `p.product_id = vpd_en.product_id`
  - left join `silver.v_product_model_catalog_description` as `vmcd` on `p.product_model_id = vmcd.product_model_id`
- Key:
  - `product_id`
- Required row-shape rule:
  - Final `gold.dim_product` must have exactly one row per `product_id`.
  - If `vpd_en` still contains duplicates for a `product_id`, resolve deterministically to one row using the latest available `modified_date`, otherwise latest ingestion metadata.
- Core attributes:
  - `product_name`
  - `product_number`
  - `color`
  - `standard_cost`
  - `list_price`
  - `size`
  - `weight`
  - `sell_start_date`
  - `sell_end_date`
  - `discontinued_date`
  - `product_category_id`
  - `product_model_id`
- Enriched attributes:
  - `product_category_name`
  - `parent_product_category_name`
  - `product_model_name`
  - `description`
  - `manufacturer`
  - `material`
  - `style`
  - `product_line`
  - `rider_experience`

#### gold.dim_date
- Role-playing date dimension for:
  - `order_date`
  - `due_date`
  - `ship_date`
  - optional product lifecycle dates
- Key:
  - `date_key` as `yyyyMMdd`
- Attributes:
  - full date, year, quarter, month, month name, week, day, fiscal placeholders if needed

#### gold.bridge_customer_address
- Source:
  - `silver.customer_address`
- Key:
  - composite `customer_id`, `address_id`, `address_type`
- Use:
  - Customer master reporting and non-order address relationships.
- Alternative:
  - Exclude from semantic model if only order analytics are desired.

### Data quality tests
1. Row count reconciliation
- Bronze to Silver row counts should match after excluding exact duplicates by dedup key.
- Gold `dim_sales_order` row count should equal distinct `sales_order_id` from `silver.sales_order_header`.
- Gold `fact_sales_order_line` row count should equal distinct (`sales_order_id`, `sales_order_detail_id`) from `silver.sales_order_detail`.

2. No-null primary keys
- Assert non-null for:
  - `customer_id` in customer
  - `address_id` in address
  - `product_id` in product
  - `product_category_id` in product_category
  - `product_model_id` in product_model
  - `sales_order_id` in sales_order_header
  - `sales_order_id`, `sales_order_detail_id` in sales_order_detail

3. Unique primary keys
- Assert uniqueness for:
  - `customer_id`
  - `address_id`
  - `product_id`
  - `product_category_id`
  - `product_description_id`
  - `product_model_id`
  - `sales_order_id` in `dim_sales_order`
  - composite `sales_order_id`, `sales_order_detail_id`
  - composite `customer_id`, `address_id`, `address_type`

4. Referential integrity
- `sales_order_header.customer_id` -> `customer.customer_id`
- `sales_order_header.ship_to_address_id` -> `address.address_id`
- `sales_order_header.bill_to_address_id` -> `address.address_id`
- `sales_order_detail.sales_order_id` -> `sales_order_header.sales_order_id`
- `sales_order_detail.product_id` -> `product.product_id`
- `product.product_category_id` -> `product_category.product_category_id`
- `product.product_model_id` -> `product_model.product_model_id` where not null
- `customer_address.customer_id` -> `customer.customer_id`
- `customer_address.address_id` -> `address.address_id`

5. Domain/value checks
- `order_qty > 0`
- `unit_price >= 0`
- `line_total >= 0`
- `sub_total >= 0`, `tax_amt >= 0`, `freight >= 0`, `total_due >= 0`
- `ship_date >= order_date` when ship date is not null
- `due_date >= order_date` when due date is not null
- `status` within observed valid set
- `email_address` basic format check where not null

## Semantic model
Build a Direct Lake semantic model over Gold with a star schema optimized for sales analytics.

### Tables
- Fact:
  - `fact_sales_order_line`
- Dimensions:
  - `dim_sales_order`
  - `dim_customer`
  - `dim_product`
  - `dim_ship_to_address`
  - `dim_bill_to_address`
  - `dim_date`
- Exclude from semantic model by default:
  - `dim_address`
  - `bridge_customer_address`

### Relationships
- `fact_sales_order_line[sales_order_id]` -> `dim_sales_order[sales_order_id]` many-to-one active
- `fact_sales_order_line[customer_id]` -> `dim_customer[customer_id]` many-to-one active
- `fact_sales_order_line[product_id]` -> `dim_product[product_id]` many-to-one active
- `fact_sales_order_line[ship_to_address_id]` -> `dim_ship_to_address[address_id]` many-to-one active
- `fact_sales_order_line[bill_to_address_id]` -> `dim_bill_to_address[address_id]` many-to-one active
- `fact_sales_order_line[order_date_key]` -> `dim_date[date_key]` many-to-one active
- `fact_sales_order_line[due_date_key]` -> `dim_date[date_key]` many-to-one inactive
- `fact_sales_order_line[ship_date_key]` -> `dim_date[date_key]` many-to-one inactive
- Do not create any relationships referencing `fact_sales_order` because that table must not exist in this build.

### Measures
- `Total Sales = SUM(fact_sales_order_line[net_line_amount])`
- `Total Quantity = SUM(fact_sales_order_line[order_qty])`
- `Order Count = DISTINCTCOUNT(fact_sales_order_line[sales_order_id])`
- `Average Order Value = DIVIDE([Total Sales], [Order Count])`
- `Average Selling Price = DIVIDE(SUM(fact_sales_order_line[net_line_amount]), SUM(fact_sales_order_line[order_qty]))`
- `Discount Amount = SUM(fact_sales_order_line[discount_amount])`
- `Tax Amount = SUMX(VALUES(dim_sales_order[sales_order_id]), CALCULATE(MAX(dim_sales_order[tax_amt])))`
- `Freight Amount = SUMX(VALUES(dim_sales_order[sales_order_id]), CALCULATE(MAX(dim_sales_order[freight])))`
- `Online Order Count = CALCULATE(DISTINCTCOUNT(dim_sales_order[sales_order_id]), dim_sales_order[is_online_order] = TRUE())`
- `Gross Margin = SUM(fact_sales_order_line[net_line_amount]) - SUMX(fact_sales_order_line, fact_sales_order_line[order_qty] * RELATED(dim_product[standard_cost]))`
- `Gross Margin % = DIVIDE([Gross Margin], [Total Sales])`

### Semantic modeling notes
- Required implementation rule:
  - Create the semantic model with exactly the tables, relationships, and measures listed above.
  - Do not infer optional copies, alternative facts, or extra bridge tables during model creation.
- Hide technical columns:
  - rowguids, raw modified dates where not analytic, record hashes, run ids.
  - In `fact_sales_order_line`, hide repeated header amount columns: `sub_total`, `tax_amt`, `freight`, `total_due`.
- Format measures:
  - currency for sales/tax/freight/margin
  - whole number for counts/quantities
  - percentage for margin and discount rates if exposed

## Report
Create a Power BI report tailored to the modeled sales/order/product/customer data.

### Page 1: Sales Overview
- KPI cards:
  - Total Sales
  - Order Count
  - Total Quantity
  - Average Order Value
  - Gross Margin %
- Line chart:
  - Total Sales by `dim_date[month]` using order date
- Clustered column chart:
  - Total Sales by `dim_product[parent_product_category_name]` or `dim_product[product_category_name]`
- Donut chart:
  - Online vs offline order count using `dim_sales_order[is_online_order]`
- Matrix:
  - Product category -> product name with Total Sales, Quantity, Gross Margin

### Page 2: Customer and Geography
- Map or filled map:
  - Total Sales by `dim_ship_to_address[country_region]`, `state_province`, `city`
- Bar chart:
  - Top customers by Total Sales using `dim_customer[customer_display_name]`
- Table:
  - Customer, company, email, phone, order count, total sales
- Stacked bar:
  - Sales by billing vs shipping country using `dim_bill_to_address[country_region]` and `dim_ship_to_address[country_region]`
- Slicer set:
  - Order date, country_region, sales_person, product category

### Page 3: Product Performance
- Bar chart:
  - Top products by Total Sales
- Scatter chart:
  - List Price vs Total Quantity with bubble size = Total Sales, color = product category
- Matrix:
  - Product model, product, color, size, standard cost, list price, total sales, gross margin
- Decomposition tree:
  - Analyze Total Sales by category -> model -> product -> shipping geography
- Optional visual:
  - Description or manufacturer drill-through if enriched product attributes are included

### Page 4: Data Quality
- KPI cards:
  - Bronze row count
  - Silver row count
  - Gold fact row counts
  - Failed DQ tests
- Table:
  - Test name, table, severity, failed rows, last run id, last checked at
- Bar chart:
  - Null-count by key column
- Matrix:
  - Referential integrity failures by relationship
- Detail table:
  - Sample failed records for invalid dates, missing keys, orphan FKs

## Data Agent
Create an AISkill grounded on the semantic model.

### Role
- Sales and product performance analytics assistant for the `e2egpt41` Gold semantic model.
- Helps business users analyze order revenue, product demand, customer behavior, category performance, and basic data quality status.

### Domain hints
- Core business entities:
  - customers
  - sales orders
  - sales order lines
  - products
  - product categories
  - billing and shipping addresses
  - order, due, and ship dates
- Preferred business terms:
  - sales, orders, quantity, average order value, gross margin, online orders, category, product model, shipping geography
- Important caveats:
  - Order-level descriptive attributes and header totals live in `dim_sales_order`
  - Line-level sales and quantity live in `fact_sales_order_line`
  - Header amounts repeated onto line grain must not be summed directly from `fact_sales_order_line`
  - Product descriptions are English-only in this build

### Starter questions
- What are total sales, order count, and average order value by month?
- Which product categories generate the highest sales and gross margin?
- What are the top 10 products by quantity sold?
- Which customers have the highest total sales?
- How do online orders compare with offline orders?
- What is the sales distribution by country, state, and city?
- Which products have the highest list price but low sales volume?
- How much tax and freight are we collecting over time?
- Are there orders with long gaps between order date and ship date?
- What data quality issues were detected in the latest run?

### Guardrails
- Answer only from the grounded semantic model; do not invent entities or columns.
- Prefer certified measures over ad hoc aggregations.
- Use `dim_sales_order` for order-level attributes, tax, freight, and online-order analysis.
- Use `fact_sales_order_line` for product, quantity, and line-sales analysis.
- Do not reference `fact_sales_order`; it must not exist in this build.
- Do not expose hidden technical metadata or dropped sensitive fields such as password values.
- Flag when results may be affected by inactive relationships, especially due date/ship date.
- If data quality test failures exist, mention they may affect confidence in the answer.