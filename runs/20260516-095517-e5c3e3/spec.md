# Run Spec 20260516-095444-fbfa15

## Inputs
- Workspace: `0933e242-69c3-4cfd-a2d8-3cec0be69c5c`
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
- Target Lakehouse: **CopilotMedallion_20260516_095444**

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
- Use defensive column references everywhere, avoid hardcoded column names without validation.
- After every join, use alias-prefixed column references to avoid ambiguity.
- When performing groupBy or aggregations, assert existence of all columns used.
- Defensive REST handling: if a `.get()` call returns None, raise an error immediately.
- Avoid using `saveAsTable`; use idempotent overwrite patterns with proper partitioning.
- Wrap all transformation logic in try/except blocks that call `_save_error(layer, e)` on exceptions and re-raise to ensure error visibility.

## Bronze
- Landing all source tables into the bronze schema with minimal transformation.
- Append mode ingestion with metadata columns added:
  - `_ingest_file_name` (string)
  - `_ingest_timestamp` (timestamp)
- Write mode: append with idempotency ensured by deduplication keys in Silver.
- Partitioning:
  - For large tables with date columns (SalesOrderHeader by OrderDate partition)
  - Others unpartitioned or partitioned by ModifiedDate where applicable.
- Tables:
  - `bronze.address`
  - `bronze.customer`
  - `bronze.customer_address`
  - `bronze.product`
  - `bronze.product_category`
  - `bronze.product_description`
  - `bronze.product_model`
  - `bronze.product_model_product_description`
  - `bronze.sales_order_detail`
  - `bronze.sales_order_header`
  - `bronze.v_get_all_categories`
  - `bronze.v_product_and_description`
  - `bronze.v_product_model_catalog_description`

## Silver
- Cleaning and normalization steps:
  - Rename columns to snake_case.
  - Trim string columns and handle nulls.
  - Deduplicate based on natural keys:
    - `address`: dedup key = AddressID
    - `customer`: dedup key = CustomerID
    - `customer_address`: composite key = (CustomerID, AddressID, AddressType)
    - `product`: dedup key = ProductID
    - `product_category`: dedup key = ProductCategoryID
    - `product_description`: dedup key = ProductDescriptionID
    - `product_model`: dedup key = ProductModelID
    - `product_model_product_description`: composite key = (ProductModelID, ProductDescriptionID, Culture)
    - `sales_order_detail`: dedup key = SalesOrderDetailID
    - `sales_order_header`: dedup key = SalesOrderID
  - Audit columns added:
    - `ingest_timestamp` (timestamp of silver ingestion)
    - `record_source` (string, e.g. 'bronze.address')
  - Partitioning:
    - `sales_order_header` partitioned by year and month of order_date
    - `sales_order_detail` partitioned by sales_order_id hash or left unpartitioned if small
  - OPTIMIZE commands on large tables: `sales_order_header`, `sales_order_detail`, `product`

## Gold
- Modeling approach:

### Dimensions
- `dim_address` (from Address)
- `dim_customer` (from Customer)
- `dim_product` (from Product joined with ProductCategory and ProductModel for descriptive attributes)
- `dim_product_category` (from ProductCategory)
- `dim_product_description` (from ProductDescription)
- `dim_product_model` (from ProductModel)
- `dim_customer_address` (junction table CustomerAddress linking dim_customer and dim_address with address_type)
- `dim_product_model_description` (junction table ProductModelProductDescription linking dim_product_model and dim_product_description with culture)

### Facts
- `fact_sales_order_header` (from SalesOrderHeader)
- `fact_sales_order_detail` (from SalesOrderDetail)

### Notes on modeling alternatives
- The views `vGetAllCategories`, `vProductAndDescription`, and `vProductModelCatalogDescription` provide enriched descriptive data that can be used to enhance dimension tables or as separate reporting views. User can choose to:
  - Use these views to augment dimension tables with additional descriptive columns.
  - Or keep them as separate read-only views for reporting enrichment.
- Please edit the spec to indicate preference.

### Data quality tests
- Row count > 0 for all tables.
- No nulls in primary key columns (e.g., AddressID, CustomerID, ProductID, SalesOrderID).
- Unique constraints on primary keys.
- Referential integrity:
  - SalesOrderHeader.CustomerID exists in Customer.CustomerID.
  - SalesOrderHeader.ShipToAddressID and BillToAddressID exist in Address.AddressID.
  - SalesOrderDetail.ProductID exists in Product.ProductID.
  - CustomerAddress.CustomerID and AddressID exist in Customer and Address respectively.
  - Product.ProductCategoryID exists in ProductCategory.ProductCategoryID.
  - Product.ProductModelID exists in ProductModel.ProductModelID.
- Valid date ranges (e.g., SellStartDate <= SellEndDate or null).

## Semantic model
- Star schema with explicit relationships:

### Tables
- Dimensions:
  - dim_customer
  - dim_address
  - dim_product
  - dim_product_category
  - dim_product_model
- Facts:
  - fact_sales_order_header
  - fact_sales_order_detail

### Relationships
- fact_sales_order_header.CustomerID -> dim_customer.CustomerID
- fact_sales_order_header.ShipToAddressID -> dim_address.AddressID
- fact_sales_order_header.BillToAddressID -> dim_address.AddressID
- fact_sales_order_detail.SalesOrderID -> fact_sales_order_header.SalesOrderID
- fact_sales_order_detail.ProductID -> dim_product.ProductID
- dim_product.ProductCategoryID -> dim_product_category.ProductCategoryID
- dim_product.ProductModelID -> dim_product_model.ProductModelID

### Measures (DAX)
- Total Sales = SUM(fact_sales_order_detail.LineTotal)
- Total Quantity Sold = SUM(fact_sales_order_detail.OrderQty)
- Average Unit Price = AVERAGE(fact_sales_order_detail.UnitPrice)
- Total Orders = DISTINCTCOUNT(fact_sales_order_header.SalesOrderID)
- Total Customers = DISTINCTCOUNT(dim_customer.CustomerID)
- Total Products Sold = DISTINCTCOUNT(fact_sales_order_detail.ProductID)

## Report
- Page 1: Sales Overview
  - Visuals: Card for Total Sales, Total Orders, Total Customers
  - Bar chart: Sales by Product Category
  - Line chart: Sales trend by OrderDate (monthly)
- Page 2: Customer Insights
  - Table: Customer details with total sales and order counts
  - Map visual: Customer shipping addresses by city/state
- Page 3: Product Performance
  - Matrix: Products with sales, quantity sold, average price
  - Pie chart: Sales distribution by Product Model
- Page 4: Data Quality
  - Visuals: Row counts per table, null counts in key columns, referential integrity test results (pass/fail indicators)

## Data Agent
- Role: Sales Data Analyst Assistant
- Domain: Sales and Product Analytics for SalesLT dataset
- Starter questions:
  1. What are the total sales and order counts for the last quarter?
  2. Which product categories have the highest sales revenue?
  3. Show me the top 10 customers by total purchase amount.
  4. How do sales trends vary by month and product model?
  5. Are there any data quality issues in the customer or sales order data?
  6. What is the average discount applied per product?
  7. List all addresses associated with a given customer.
  8. How many products are currently discontinued?
  9. What is the geographic distribution of customers by state or country?
  10. Provide a summary of sales order statuses and their counts.
- Guardrails:
  - Only answer questions based on the modeled semantic layer.
  - Do not expose raw PII such as passwords or password hashes.
  - Validate all user inputs for entity names before querying.
  - Provide clear error messages if data is missing or queries are ambiguous.
  - Limit responses to factual data insights; no speculation.
