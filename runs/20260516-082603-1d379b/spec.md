# Run Spec 20260516-081758-967a7c

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
  - Alternative: rely only on `sales_order_header.ship_to_address_id` and `bill_to_address_id` for order analytics and keep this table for customer master views.

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
  - Plausible keys:
    - `product_id`, `culture` if multilingual rows exist
    - alternative: `product_id` only if one row per product in practice
- Use:
  - Enrich product dimension with `product_model`, `description`, `culture`.
- User-editable decision:
  - If multiple cultures are present, either keep a separate localized dimension or filter to a preferred culture in Gold.

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
  - Dimensions from customer, product, product category, address, and date.
- Plausible alternative route:
  - Also create an order-header summary fact from `sales_order_header` for order-level KPIs. --> I don't want an extra fact, I want an extra dimension for order number etc
- Plausible alternative route for product descriptions:
  - Option A: enrich `dim_product` with one preferred-culture description from `v_product_and_description`.
  - Option B: create `dim_product_localized` keyed by `product_id` + `culture`.
- User may edit the spec to choose A or B if multilingual reporting is required. --> I don't need multi language, so english only is fine

### Fact tables

#### gold.fact_sales_order_line
- Grain:
  - One row per `sales_order_id` + `sales_order_detail_id`
- Built from:
  - `silver.sales_order_detail` d
  - inner/left join `silver.sales_order_header` h on `sales_order_id`
- Natural transaction keys retained:
  - `sales_order_id`
  - `sales_order_detail_id`
  - `sales_order_number`
- Foreign keys:
  - `customer_id`
  - `product_id`
  - `bill_to_address_id`
  - `ship_to_address_id`
  - `order_date_key`
  - `due_date_key`
  - `ship_date_key`
- Measures columns:
  - `order_qty`
  - `unit_price`
  - `unit_price_discount`
  - `line_total`
  - `sub_total` from header only if needed for reconciliation, otherwise avoid repeating at line grain
  - `tax_amt`
  - `freight`
  - `total_due`
- Derived columns:
  - `gross_line_amount = order_qty * unit_price`
  - `discount_amount = order_qty * unit_price * unit_price_discount` if discount is rate; alternative: if source semantics prove amount, keep raw field only
  - `net_line_amount = line_total`
  - `is_online_order = online_order_flag`
- Modeling caution:
  - Header totals repeat across lines if joined to line grain; use explicit measures to avoid double counting header-level amounts.

#### gold.fact_sales_order
- Grain:
  - One row per `sales_order_id`
- Built from:
  - `silver.sales_order_header`
- Foreign keys:
  - `customer_id`
  - `bill_to_address_id`
  - `ship_to_address_id`
  - `order_date_key`
  - `due_date_key`
  - `ship_date_key`
- Measures columns:
  - `sub_total`
  - `tax_amt`
  - `freight`
  - `total_due`
  - `order_count` as implicit row count
- Use:
  - Clean order-level KPI reporting without line-level duplication.

### Dimension tables

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
  - Role-played as billing and shipping address.

#### gold.dim_product
- Source:
  - `silver.product` p
  - left join `silver.product_category` pc on `p.product_category_id = pc.product_category_id`
  - left join `silver.v_get_all_categories` c on category id
  - left join `silver.product_model` pm on `p.product_model_id = pm.product_model_id`
  - optional left join `silver.v_product_and_description` vpd on `p.product_id = vpd.product_id`
  - optional left join `silver.v_product_model_catalog_description` vmcd on `p.product_model_id = vmcd.product_model_id`
- Key:
  - `product_id`
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
  - `description` or preferred-culture description
  - `manufacturer`
  - `material`
  - `style`
  - `product_line`
  - `rider_experience`
- User-editable decision:
  - If description/model attributes create duplicate product rows, split into `dim_product` and `dim_product_model`.

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
- Gold `fact_sales_order` row count should equal distinct `sales_order_id` from `silver.sales_order_header`.
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
  - `sales_order_id`
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
- Facts:
  - `fact_sales_order_line`
  - `fact_sales_order`
- Dimensions:
  - `dim_customer`
  - `dim_product`
  - `dim_address`
  - `dim_date`
  - optional `bridge_customer_address`

### Relationships
- `fact_sales_order_line[customer_id]` -> `dim_customer[customer_id]` many-to-one
- `fact_sales_order_line[product_id]` -> `dim_product[product_id]` many-to-one
- `fact_sales_order_line[ship_to_address_id]` -> `dim_address[address_id]` many-to-one inactive or via role-playing copy
- `fact_sales_order_line[bill_to_address_id]` -> `dim_address[address_id]` many-to-one inactive or via role-playing copy
- `fact_sales_order_line[order_date_key]` -> `dim_date[date_key]` active
- `fact_sales_order_line[due_date_key]` -> `dim_date[date_key]` inactive
- `fact_sales_order_line[ship_date_key]` -> `dim_date[date_key]` inactive
- `fact_sales_order[same FK pattern]` to the same dimensions
- Recommended semantic simplification:
  - Duplicate `dim_address` into `dim_ship_to_address` and `dim_bill_to_address`
  - Duplicate `dim_date` views/roles for order, due, ship if authoring experience needs simpler active relationships

### Measures
- `Total Sales = SUM(fact_sales_order_line[net_line_amount])`
- `Total Quantity = SUM(fact_sales_order_line[order_qty])`
- `Order Count = DISTINCTCOUNT(fact_sales_order[sales_order_id])`
- `Average Order Value = DIVIDE(SUM(fact_sales_order[total_due]), DISTINCTCOUNT(fact_sales_order[sales_order_id]))`
- `Average Selling Price = DIVIDE(SUM(fact_sales_order_line[net_line_amount]), SUM(fact_sales_order_line[order_qty]))`
- `Discount Amount = SUM(fact_sales_order_line[discount_amount])`
- `Tax Amount = SUM(fact_sales_order[tax_amt])`
- `Freight Amount = SUM(fact_sales_order[freight])`
- `Online Order Count = CALCULATE(DISTINCTCOUNT(fact_sales_order[sales_order_id]), fact_sales_order[is_online_order] = TRUE())`
- `Gross Margin = SUM(fact_sales_order_line[net_line_amount]) - SUMX(fact_sales_order_line, fact_sales_order_line[order_qty] * RELATED(dim_product[standard_cost]))`
- `Gross Margin % = DIVIDE([Gross Margin], [Total Sales])`

### Semantic modeling notes
- Hide technical columns:
  - rowguids, raw modified dates where not analytic, record hashes, run ids.
- Format measures:
  - currency for sales/tax/freight/margin
  - whole number for counts/quantities
  - percentage for margin and discount rates if exposed
- Alternative model:
  - If user wants only a single fact, keep `fact_sales_order_line` and define header-safe measures using distinct order summarization patterns.

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
  - Total Sales by `dim_product[parent_product_category_name]` or `product_category_name`
- Donut chart:
  - Online vs offline order count using `online_order_flag`
- Matrix:
  - Product category -> product name with Total Sales, Quantity, Gross Margin

### Page 2: Customer and Geography
- Map or filled map:
  - Total Sales by `dim_ship_to_address[country_region]`, `state_province`, `city`
- Bar chart:
  - Top customers by Total Sales using `customer_display_name`
- Table:
  - Customer, company, email, phone, order count, total sales
- Stacked bar:
  - Sales by billing vs shipping country if both role dimensions are surfaced
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
  - Analyze Total Sales by category -> model -> product -> customer geography
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
  - Order-level totals live in `fact_sales_order`
  - Line-level sales and quantity live in `fact_sales_order_line`
  - Header amounts should not be summed from line grain without safe measures
  - Product descriptions may be culture-specific if multiple cultures exist

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
- Use `fact_sales_order` for order-level financial totals and order counts.
- Use `fact_sales_order_line` for product, quantity, and line-sales analysis.
- If a request depends on multilingual product descriptions or undefined culture handling, state the ambiguity.
- Do not expose hidden technical metadata or dropped sensitive fields such as password values.
- Flag when results may be affected by inactive relationships, especially due date/ship date and billing/ship address roles.
- If data quality test failures exist, mention they may affect confidence in the answer.
