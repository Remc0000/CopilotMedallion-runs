# Run Spec 20260516-111217-3f5993

## Inputs
- Workspace: `c5ed6076-483d-4c4d-90e8-68a349ee4673`
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
- Target Lakehouse: **CopilotMedallion_20260516_111217**

## Generic guidance
Apply these reference skills/agents at all times:
- FabricDataEngineer agent: https://github.com/microsoft/skills-for-fabric/blob/main/agents/FabricDataEngineer.agent.md
- e2e-medallion-architecture skill: https://github.com/microsoft/skills-for-fabric/tree/main/skills/e2e-medallion-architecture
- spark-authoring-cli skill: https://github.com/microsoft/skills-for-fabric/tree/main/skills/spark-authoring-cli
- powerbi-authoring-cli skill: https://github.com/microsoft/skills-for-fabric/tree/main/skills/powerbi-authoring-cli
- powerbi-consumption-cli skill: https://github.com/microsoft/skills-for-fabric/blob/main/skills/powerbi-consumption-cli
- powerbi-semantic-model-authoring: https://github.com/RuiRomano/powerbi-agentic-plugins/tree/main/plugins/powerbi/skills/powerbi-semantic-model-authoring
- powerbi-report-authoring: https://github.com/RuiRomano/powerbi-agentic-plugins/tree/main/plugins/powerbi/skills/powerbi-report-authoring

Cross-cutting code rules:
- Use defensive column references with explicit alias prefixes in all joins and selects.
- Assert existence of columns before groupBy or aggregation operations.
- Defensive REST handling: if a `.get()` call returns None, raise an error immediately.
- Use idempotent overwrite patterns for all writes to prevent duplicates.
- Avoid `saveAsTable`; use Delta Lake or Fabric-native write methods.
- Each generated notebook cell must start with a short markdown comment block describing the purpose.
- Error handling must be loud: catch exceptions, call `_save_error(layer, e)`, then re-raise.
- Include audit columns where appropriate (e.g., ingestion timestamp, source system).

## Bronze
- Ingest all source tables as-is into the `bronze` schema under the target lakehouse.
- Add standard bronze metadata columns: `_ingest_timestamp` (current timestamp), `_source_file` (if applicable).
- Write mode: `overwrite` with partitioning where applicable.
- Partitioning:
  - `SalesOrderHeader` and `SalesOrderDetail` partitioned by `OrderDate` (from `SalesOrderHeader`) or ingestion date if not available.
  - `Product`, `ProductCategory`, `ProductModel`, `ProductDescription`, `Customer`, `Address` unpartitioned due to smaller size.
- Preserve all columns exactly as in source, including binary and GUID columns.
- For views (`vGetAllCategories`, `vProductAndDescription`, `vProductModelCatalogDescription`), ingest as tables for consistency and further processing.

## Silver
- Clean and standardize each table with these steps:

### Address
- Deduplication key: `AddressID`
- Rename columns to snake_case (e.g., `address_id`, `address_line1`).
- Validate non-null `address_id`.
- Audit columns: `_load_timestamp` (current timestamp).
- Remove rows with null `address_id`.
- OPTIMIZE by `state_province` for query performance.

### Customer
- Deduplication key: `CustomerID`
- Rename columns to snake_case.
- Validate non-null `customer_id`.
- Mask sensitive columns (`password_hash`, `password_salt`) or exclude from silver.
- Add audit columns.
- OPTIMIZE by `last_name` for common queries.

### CustomerAddress
- Deduplication key: composite (`CustomerID`, `AddressID`, `AddressType`)
- Rename columns to snake_case.
- Validate non-null keys.
- Audit columns.
- OPTIMIZE by `customer_id`.

### Product
- Deduplication key: `ProductID`
- Rename columns to snake_case.
- Validate non-null `product_id`.
- Convert timestamps to date where possible.
- Audit columns.
- OPTIMIZE by `product_category_id`.

### ProductCategory
- Deduplication key: `ProductCategoryID`
- Rename columns to snake_case.
- Validate non-null `product_category_id`.
- Audit columns.
- OPTIMIZE by `parent_product_category_id`.

### ProductDescription
- Deduplication key: `ProductDescriptionID`
- Rename columns to snake_case.
- Validate non-null `product_description_id`.
- Audit columns.

### ProductModel
- Deduplication key: `ProductModelID`
- Rename columns to snake_case.
- Validate non-null `product_model_id`.
- Audit columns.

### ProductModelProductDescription
- Deduplication key: composite (`ProductModelID`, `ProductDescriptionID`, `Culture`)
- Rename columns to snake_case.
- Validate non-null keys.
- Audit columns.

### SalesOrderHeader
- Deduplication key: `SalesOrderID`
- Rename columns to snake_case.
- Validate non-null `sales_order_id`.
- Convert timestamps to date where appropriate.
- Audit columns.
- OPTIMIZE by `order_date`.

### SalesOrderDetail
- Deduplication key: `SalesOrderDetailID`
- Rename columns to snake_case.
- Validate non-null `sales_order_detail_id`.
- Audit columns.
- OPTIMIZE by `sales_order_id`.

### Views (`vGetAllCategories`, `vProductAndDescription`, `vProductModelCatalogDescription`)
- Rename columns to snake_case.
- Validate key columns for joins.
- Audit columns.
- No partitioning needed.

## Gold
### Modeling approach
- Dimensions:
  - `dim_customer` from `Customer` (master data on customers)
  - `dim_address` from `Address` (master data on addresses)
  - `dim_product` from `Product` joined with `ProductCategory` and `ProductModel` for enriched attributes
  - `dim_product_category` from `ProductCategory` (hierarchical categories)
  - `dim_product_description` from `ProductDescription` joined with `ProductModelProductDescription` for multilingual descriptions
  - `dim_date` generated date dimension (from `OrderDate`, `DueDate`, `ShipDate` in orders)
- Facts:
  - `fact_sales_order` from `SalesOrderHeader` joined with `SalesOrderDetail` for transactional sales data
- Junction tables:
  - `customer_address` from `CustomerAddress` linking customers to addresses with address types

### Alternative modeling options
- Option A: Flatten product descriptive info from views (`vProductAndDescription`, `vProductModelCatalogDescription`) into `dim_product` for richer product attributes.
- Option B: Keep product description and model catalog description as separate dims for modularity.
- Ask user to choose by editing this spec: Use A

### Data quality tests
- Row count > 0 for all tables.
- No nulls in primary keys (`customer_id`, `address_id`, `product_id`, `sales_order_id`, etc.).
- Unique primary keys per dimension and fact.
- Referential integrity:
  - `customer_address.customer_id` exists in `dim_customer.customer_id`
  - `customer_address.address_id` exists in `dim_address.address_id`
  - `fact_sales_order.customer_id` exists in `dim_customer.customer_id`
  - `fact_sales_order.ship_to_address_id` and `bill_to_address_id` exist in `dim_address.address_id`
  - `fact_sales_order_detail.product_id` exists in `dim_product.product_id`
- Date consistency: `order_date <= due_date <= ship_date` in `fact_sales_order`.

## Semantic model
- Star schema with:
  - Fact table: `fact_sales_order`
  - Dimension tables: `dim_customer`, `dim_address`, `dim_product`, `dim_product_category`, `dim_product_description`, `dim_date`
- Relationships:
  - `fact_sales_order.customer_id` → `dim_customer.customer_id`
  - `fact_sales_order.ship_to_address_id` → `dim_address.address_id`
  - `fact_sales_order.bill_to_address_id` → `dim_address.address_id`
  - `fact_sales_order_detail.product_id` → `dim_product.product_id`
  - `dim_product.product_category_id` → `dim_product_category.product_category_id`
  - `dim_product.product_model_id` → `dim_product_model.product_model_id` (if included)
  - Date keys linked to `dim_date` on `order_date`, `due_date`, `ship_date`
- Measures:
  - Total Sales Amount = SUM(`fact_sales_order.total_due`)
  - Total Order Quantity = SUM(`fact_sales_order_detail.order_qty`)
  - Average Unit Price = AVERAGE(`fact_sales_order_detail.unit_price`)
  - Total Discount = SUM(`fact_sales_order_detail.unit_price_discount`)
  - Total Freight = SUM(`fact_sales_order.freight`)
  - Order Count = DISTINCTCOUNT(`fact_sales_order.sales_order_id`)
- Calculated columns:
  - Order Profit Margin = (TotalDue - StandardCost * OrderQty) / TotalDue (requires join with product cost)
- Include role-playing dimensions for billing and shipping addresses if needed.

## Report
- Page 1: Sales Overview
  - Visuals: Total Sales Amount by Month (line chart), Order Count by Customer (bar chart), Average Unit Price by Product Category (column chart)
- Page 2: Customer Insights
  - Visuals: Top Customers by Sales (table), Customer Geographic Distribution (map using address city/state), Customer Order Frequency (histogram)
- Page 3: Product Performance
  - Visuals: Sales by Product Category (treemap), Product Sales Trend (line chart), Discount Analysis (scatter plot of discount vs sales)
- Page 4: Data Quality Dashboard
  - Visuals: Row counts per table, Null key counts, Referential integrity violations (if any), Duplicate key counts, Date consistency checks
- Use slicers for date ranges, product categories, and customer segments.

## Data Agent
- Role: Sales Data Analyst Assistant
- Domain: Sales and Product Analytics for SalesLT dataset
- Starter questions:
  1. What are the total sales and order counts for the last quarter?
  2. Which customers have the highest sales volume?
  3. How do sales vary by product category and region?
  4. What is the average discount applied per product?
  5. Are there any data quality issues in the sales orders?
  6. How many orders were shipped late compared to due dates?
  7. What is the distribution of order quantities across products?
  8. Can you show sales trends over time by product model?
  9. Which addresses have the most associated customers?
  10. What is the profit margin by product category?
- Guardrails:
  - Only answer based on the semantic model data.
  - Do not expose sensitive customer data (e.g., passwords).
  - Validate date ranges and numeric inputs.
  - Provide explanations for measures and calculations.
  - Alert if data quality issues are detected during queries.
