# Run Spec 20260516-090018-fca432

## Inputs
- Workspace: `38b615f4-9926-4413-851d-61a2f36096a0`
- Source Lakehouse: **SalesLT** (`efe41f78-82b7-47ee-9780-2d78372bfdf3`)
- Tables to ingest into Bronze:
  - `Address`
  - `Customer`
  - `CustomerAddress`
  - `Product`
  - `ProductCategory`
  - `ProductDescription`
  - `ProductModel`
  - `ProductModelProductDescription`
  - `SalesOrderDetail`
  - `SalesOrderHeader`
  - `vGetAllCategories`
  - `vProductAndDescription`
  - `vProductModelCatalogDescription`
- Target Lakehouse: **CopilotMedallion_20260516_090018**

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
- Use defensive column references everywhere to avoid runtime errors.
- Always alias tables in joins and prefix join keys with aliases.
- Assert existence of columns before groupBy or aggregation operations.
- Defensive REST handling: check if responses are None before calling `.get()`.
- Avoid `saveAsTable`; use idempotent overwrite patterns with partitioning.
- Wrap transformations in try/except blocks that call `_save_error(layer, e)` and re-raise exceptions to fail loudly.

## Bronze
- Landing all source tables as-is into `bronze` schema under the target lakehouse.
- Include metadata columns: `_ingest_timestamp` (current timestamp), `_source_file` (if applicable).
- Write mode: append with deduplication handled in Silver.
- Partitioning:
  - `SalesOrderHeader` and `SalesOrderDetail` partitioned by `OrderDate` (from `SalesOrderHeader`) for efficient time-based queries.
  - Other tables unpartitioned due to smaller size or lack of natural partition keys.
- Tables ingested:
  - `Address`
  - `Customer`
  - `CustomerAddress`
  - `Product`
  - `ProductCategory`
  - `ProductDescription`
  - `ProductModel`
  - `ProductModelProductDescription`
  - `SalesOrderDetail`
  - `SalesOrderHeader`
  - Views (`vGetAllCategories`, `vProductAndDescription`, `vProductModelCatalogDescription`) ingested as raw tables for reference.

## Silver
- Cleaning steps per table:
  - Rename all columns to snake_case for consistency.
  - Add audit columns: `ingest_date` (current date), `source_system` = 'SalesLT'.
  - Deduplication keys:
    - `address`: `address_id`
    - `customer`: `customer_id`
    - `customer_address`: composite key (`customer_id`, `address_id`, `address_type`)
    - `product`: `product_id`
    - `product_category`: `product_category_id`
    - `product_description`: `product_description_id`
    - `product_model`: `product_model_id`
    - `product_model_product_description`: composite key (`product_model_id`, `product_description_id`, `culture`)
    - `sales_order_header`: `sales_order_id`
    - `sales_order_detail`: `sales_order_detail_id`
    - Views: no deduplication, used as reference.
  - Data type enforcement: ensure decimals, timestamps, and booleans are correctly typed.
  - Null handling: fill or flag critical columns if nulls found (e.g., PKs).
  - Optimize tables with `OPTIMIZE` command post-cleaning for performance.
- Defensive joins and column existence checks in all transformations.

## Gold
- Proposed dimensional and fact tables based on actual columns:

### Dimensions
- `dim_address`:
  - PK: `address_id`
  - Attributes: address_line1, address_line2, city, state_province, country_region, postal_code
- `dim_customer`:
  - PK: `customer_id`
  - Attributes: name_style, title, first_name, middle_name, last_name, suffix, company_name, sales_person, email_address, phone
- `dim_product`:
  - PK: `product_id`
  - Attributes: name, product_number, color, size, weight, sell_start_date, sell_end_date, discontinued_date, thumbnail_photo_file_name
  - FKs: `product_category_id` → `dim_product_category`, `product_model_id` → `dim_product_model`
- `dim_product_category`:
  - PK: `product_category_id`
  - Attributes: parent_product_category_id (self-FK), name
- `dim_product_description`:
  - PK: `product_description_id`
  - Attribute: description
- `dim_product_model`:
  - PK: `product_model_id`
  - Attributes: name, catalog_description
- `dim_customer_address` (junction dimension):
  - Composite PK: (`customer_id`, `address_id`, `address_type`)
  - Attributes: address_type

### Facts
- `fact_sales_order_header`:
  - PK: `sales_order_id`
  - FKs: `customer_id` → `dim_customer`, `ship_to_address_id` → `dim_address`, `bill_to_address_id` → `dim_address`
  - Measures: subtotal, tax_amt, freight, total_due
  - Attributes: order_date, due_date, ship_date, status, online_order_flag, sales_order_number, purchase_order_number, account_number, ship_method, credit_card_approval_code, comment
- `fact_sales_order_detail`:
  - PK: `sales_order_detail_id`
  - FK: `sales_order_id` → `fact_sales_order_header`, `product_id` → `dim_product`
  - Measures: order_qty, unit_price, unit_price_discount, line_total

### Alternative modeling notes:
- The views (`vGetAllCategories`, `vProductAndDescription`, `vProductModelCatalogDescription`) provide enriched descriptive info and could be used to build aggregated or denormalized gold tables if desired.
- Could consider a denormalized product catalog fact combining product, model, description, and category for simplified reporting.

### Data quality tests
- Row count > 0 for all gold tables.
- No nulls in PK columns.
- Unique PK enforcement.
- Referential integrity tests:
  - `fact_sales_order_header.customer_id` exists in `dim_customer.customer_id`
  - `fact_sales_order_header.ship_to_address_id` and `bill_to_address_id` exist in `dim_address.address_id`
  - `fact_sales_order_detail.sales_order_id` exists in `fact_sales_order_header.sales_order_id`
  - `fact_sales_order_detail.product_id` exists in `dim_product.product_id`
- No negative or zero values in measures like `order_qty`, `unit_price`, `line_total`.

## Semantic model
- Star schema with:
  - Fact tables: `fact_sales_order_header`, `fact_sales_order_detail`
  - Dimension tables: `dim_customer`, `dim_address`, `dim_product`, `dim_product_category`, `dim_product_description`, `dim_product_model`, `dim_customer_address`
- Relationships:
  - `fact_sales_order_header.customer_id` → `dim_customer.customer_id`
  - `fact_sales_order_header.ship_to_address_id` and `bill_to_address_id` → `dim_address.address_id`
  - `fact_sales_order_detail.sales_order_id` → `fact_sales_order_header.sales_order_id`
  - `fact_sales_order_detail.product_id` → `dim_product.product_id`
  - `dim_product.product_category_id` → `dim_product_category.product_category_id`
  - `dim_product.product_model_id` → `dim_product_model.product_model_id`
  - `dim_customer_address.customer_id` → `dim_customer.customer_id`
  - `dim_customer_address.address_id` → `dim_address.address_id`
- Measures (DAX):
  - Total Sales Amount = SUM(`fact_sales_order_detail[line_total]`)
  - Total Order Quantity = SUM(`fact_sales_order_detail[order_qty]`)
  - Average Unit Price = AVERAGE(`fact_sales_order_detail[unit_price]`)
  - Total Freight = SUM(`fact_sales_order_header[freight]`)
  - Total Tax = SUM(`fact_sales_order_header[tax_amt]`)
  - Total Orders = DISTINCTCOUNT(`fact_sales_order_header[sales_order_id]`)

## Report
- Pages:
  1. **Sales Overview**
     - Visuals: 
       - Card visuals for Total Sales Amount, Total Orders, Total Freight, Total Tax
       - Line chart of Sales Amount by Order Date
       - Bar chart of Sales Amount by Product Category
  2. **Customer Analysis**
     - Visuals:
       - Table of Customers with Total Sales and Order Count
       - Map visual showing Customer Addresses by City/State
       - Filter slicers on Customer attributes (e.g., SalesPerson, CompanyName)
  3. **Product Performance**
     - Visuals:
       - Bar chart of Top Products by Sales Amount
       - Matrix of Product Model and Product Category with Sales and Quantity
       - Trend line of Sales by Product over time
  4. **Data Quality**
     - Visuals:
       - Row counts per gold table
       - Null counts in key columns
       - Referential integrity test results (pass/fail indicators)
       - Duplicate key counts if any

## Data Agent
- Role: Senior Sales Data Analyst assistant for exploring SalesLT data in Microsoft Fabric.
- Domain hints: Sales orders, customers, products, addresses, product categories, and sales performance.
- Starter questions:
  1. What are the total sales and order counts by month?
  2. Which customers have generated the highest revenue?
  3. How do sales vary by product category and product model?
  4. What is the average discount applied on sales orders?
  5. Show me the sales trends for a specific product over time.
  6. Which regions (states/countries) have the most customers?
  7. Are there any data quality issues in the sales or customer data?
  8. How many orders are currently pending shipment?
  9. What is the distribution of order quantities across products?
  10. Can you list customers along with their shipping and billing addresses?
- Guardrails:
  - Only answer questions based on the modeled gold tables and semantic model.
  - Avoid exposing raw PII such as passwords or password salts.
  - Validate all user inputs for filters to avoid injection or invalid queries.
  - Return error messages if requested data is outside the scope of the model.
  - Prioritize performance by limiting large scans and suggesting filters when needed.
