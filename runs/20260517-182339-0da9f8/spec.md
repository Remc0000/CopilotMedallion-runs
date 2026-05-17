# Run Spec 20260517-181731-482a3f

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
- powerbi-report-authoring: https://github.com/RuiRomano/powerbi-agentic-plugins/tree/main/plugins/powerbi/skills/powerbi-report-authoring

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

Silver table specifications:

### `silver.address`
- Rename examples:
  - `AddressID` -> `address_id`
  - `AddressLine1` -> `address_line1`
  - `StateProvince` -> `state_province`
  - `CountryRegion` -> `country_region`
  - `ModifiedDate` -> `modified_date`
- Dedup key: `address_id`
- Dedup rule: keep latest `modified_date`, tie-break on `_ingest_ts`
- Cleaning:
  - trim all string columns
  - standardize empty strings to null for `address_line2`
  - uppercase or title-case location fields only if business approves; default is preserve original values

### `silver.customer`
- Dedup key: `customer_id`
- Dedup rule: keep latest `modified_date`
- Cleaning:
  - derive `full_name` from available title/first/middle/last/suffix without overwriting source components
  - trim `email_address`, lower-case email for matching helper column `email_address_normalized`
  - keep `password_hash` and `password_salt` in Silver only if strictly needed downstream; default recommendation is to retain in Silver restricted zone but exclude from Gold
- Sensitive data note:
  - Gold should not expose `password_hash`, `password_salt`, possibly not even raw email/phone unless required

### `silver.customer_address`
- Dedup key: (`customer_id`, `address_id`, `address_type`)
- Alternative if duplicate address types appear unexpectedly: dedup on (`customer_id`, `address_id`) and keep latest `modified_date`
- Cleaning:
  - trim `address_type`
  - validate foreign keys against `silver.customer.customer_id` and `silver.address.address_id`

### `silver.product`
- Dedup key: `product_id`
- Dedup rule: keep latest `modified_date`
- Cleaning:
  - standardize text attributes: `name`, `product_number`, `color`, `size`, `thumbnail_photo_file_name`
  - derive flags:
    - `is_discontinued` from `discontinued_date is not null`
    - `is_currently_sellable` from sell start/end/discontinued dates
  - keep `thumbnail_photo` in Silver but exclude from Gold and semantic model
- FK expectations:
  - `product_category_id` -> `silver.product_category.product_category_id`
  - `product_model_id` -> `silver.product_model.product_model_id`

### `silver.product_category`
- Dedup key: `product_category_id`
- Dedup rule: keep latest `modified_date`
- Cleaning:
  - preserve `parent_product_category_id` for hierarchy
  - derive hierarchy helper later in Gold

### `silver.product_description`
- Dedup key: `product_description_id`
- Dedup rule: keep latest `modified_date`
- Cleaning:
  - trim `description`

### `silver.product_model`
- Dedup key: `product_model_id`
- Dedup rule: keep latest `modified_date`
- Cleaning:
  - retain `catalog_description` as large text
  - trim `name`

### `silver.product_model_product_description`
- Dedup key: (`product_model_id`, `product_description_id`, `culture`)
- Dedup rule: keep latest `modified_date`
- Cleaning:
  - normalize `culture` to lower-case
  - validate FKs to `product_model` and `product_description`

### `silver.sales_order_header`
- Dedup key: `sales_order_id`
- Dedup rule: keep latest `modified_date`
- Cleaning:
  - derive `order_date_key`, `due_date_key`, `ship_date_key` as `yyyyMMdd` integers
  - derive `order_year`, `order_month`
  - normalize `online_order_flag`
  - trim `sales_order_number`, `purchase_order_number`, `account_number`, `ship_method`
- FK expectations:
  - `customer_id` -> `silver.customer.customer_id`
  - `ship_to_address_id` -> `silver.address.address_id`
  - `bill_to_address_id` -> `silver.address.address_id`

### `silver.sales_order_detail`
- Dedup key: (`sales_order_id`, `sales_order_detail_id`)
- Dedup rule: keep latest `modified_date`
- Cleaning:
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


Gold should look like this:
Dim_Customer -> join customer, customeraddress (only use the AddressType='Main Office'), address and leave all relevant/meaningfull fields
Dim_Product -> join product, productcategory, producdescription, productmodel, productmodelproductdescription (where culture='en'). Please notice that in productcategory there is a self join. So in the end I have the category (which is the parent category) and a subcategory.
Dim_Sales --> Use the salesperson field from salesorderheader, but remove the adventureworks/ in front of the name and remove the last number at the end off the name
Fact_Sales --> join salesorderdetail and salesorderheader and keep all relevant fields


Gold should look like this:
Dim_Customer -> join customer, customeraddress (only use the AddressType='Main Office'), address and leave all relevant/meaningfull fields
Dim_Product -> join product, productcategory, producdescription, productmodel, productmodelproductdescription (where culture='en'). Please notice that in productcategory there is a self join. So in the end I have the category (which is the parent category) and a subcategory.
Dim_Sales --> Use the salesperson field from salesorderheader, but remove the adventureworks/ in front of the name and remove the last number at the end off the name
Fact_Sales --> join salesorderdetail and salesorderheader and keep all relevant fields


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
