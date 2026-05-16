# Run Spec 20260516-122223-3d24f4

## Updated specs

### Iteration 3 — 2026-05-16 12:31:33Z — failed layer: bronze (run: 20260516-122334-99df4d)
- **Root cause (1-line summary)**: Missing explicit aliasing and partition column references caused ambiguous or unresolved column errors during Bronze ingestion.
- **What was changed**:
  - In the Bronze section, explicitly require all source tables to be aliased before any operation.
  - For `SalesOrderHeader`, require explicit alias `soh` and assert existence, non-null, and date type of `soh.OrderDate` before partitioning.
  - For `SalesOrderDetail`, require alias `sod` and left join with `soh` on `SalesOrderID` to retrieve `soh.OrderDate`; define `partition_date` as `coalesce(soh.OrderDate, sod.ModifiedDate)` with explicit assertions before partitioning.
  - For `CustomerAddress`, require alias `ca` and assert `ca.ModifiedDate` exists, non-null, and is date type before partitioning.
  - Enforce prefixing of all columns with table aliases in joins and partitionBy to prevent ambiguous references.
  - Add explicit null and type checks on partition columns before write operations.

## Inputs
- Workspace: `f81d9542-fe59-4e24-8ed8-0f73db2693ce`
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
- Target Lakehouse: **CopilotMedallion_20260516_122223**

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
- Use defensive column references with explicit null checks.
- Always alias tables in joins and prefix columns with alias to avoid ambiguity.
- Assert existence of columns before groupBy or aggregation operations.
- Defensive REST API handling: check for None before calling `.get()`.
- Use idempotent overwrite patterns for writes.
- Avoid `saveAsTable`; use lakehouse paths or managed tables with overwrite.
- Error-loud try/except blocks call `_save_error(layer, e)` and re-raise exceptions.
- Each notebook code cell starts with a short markdown comment block describing purpose.
- **New:** Before writing partitioned tables, explicitly assert that partition columns exist, contain no nulls, and are of appropriate date type; if partition column is derived via join (e.g., `OrderDate` for `SalesOrderDetail`), perform left join with `SalesOrderHeader` and fallback to `ModifiedDate` if `OrderDate` is null.
- Always prefix columns with table aliases in joins and partitionBy to prevent ambiguous references.

## Bronze
- Ingest all source tables as-is into the `bronze` schema under the target lakehouse.
- Include metadata columns: `_ingest_timestamp` (current timestamp), `_source_file` (if applicable).
- Use `overwrite` mode with partitioning where beneficial:
  - For `SalesOrderHeader`:
    - Alias source as `soh`.
    - Assert `soh.OrderDate` column exists, is non-null, and is of date type before write.
    - Partition by `soh.OrderDate` (date part).
  - For `SalesOrderDetail`:
    - Alias source as `sod`.
    - Left join `sod` with `SalesOrderHeader` aliased as `soh` on `sod.SalesOrderID = soh.SalesOrderID` to retrieve `soh.OrderDate`.
    - Define a partition column `partition_date` as `coalesce(soh.OrderDate, sod.ModifiedDate)`.
    - Assert `partition_date` exists, is non-null, and is of date type before write.
    - Partition by `partition_date`.
  - For `CustomerAddress`:
    - Alias source as `ca`.
    - Assert `ca.ModifiedDate` column exists, is non-null, and is of date type before write.
    - Partition by `ca.ModifiedDate` (date part).
  - Other tables ingested without partitioning due to smaller size or lack of natural partition key.
- Preserve original column names and types exactly.
- Include full audit columns from source (`ModifiedDate`, `rowguid`).
- Use idempotent writes to allow reruns without duplicates.
- Always alias source tables and prefix columns in joins to avoid ambiguity.
- Example: `spark.read.format("delta").load(source_path).alias("src").write.mode("overwrite").partitionBy("ModifiedDate").save(bronze_path)`

## Silver
- Clean and conform data from bronze with these steps:

### Address
- Deduplication key: `AddressID`
- Rename columns to snake_case: e.g., `address_id`, `address_line1`.
- Trim string columns, nullify empty strings.
- Add audit columns: `load_timestamp` (current timestamp).
- Validate `PostalCode` format where possible.
- Drop rows with null `AddressID`.

### Customer
- Deduplication key: `CustomerID`
- Rename columns to snake_case.
- Mask or exclude sensitive columns like `PasswordHash`, `PasswordSalt`.
- Normalize name fields (trim, proper casing).
- Add audit columns.
- Drop rows with null `CustomerID`.

### CustomerAddress
- Deduplication key: composite of `CustomerID`, `AddressID`, `AddressType`
- Rename columns to snake_case.
- Validate foreign keys exist in `Customer` and `Address`.
- Add audit columns.

### Product
- Deduplication key: `ProductID`
- Rename columns to snake_case.
- Convert decimal and timestamp columns consistently.
- Validate foreign keys `product_category_id`, `product_model_id` exist.
- Add audit columns.

### ProductCategory
- Deduplication key: `ProductCategoryID`
- Rename columns to snake_case.
- Validate self-referencing FK `parent_product_category_id`.
- Add audit columns.

### ProductDescription
- Deduplication key: `ProductDescriptionID`
- Rename columns to snake_case.
- Add audit columns.

### ProductModel
- Deduplication key: `ProductModelID`
- Rename columns to snake_case.
- Add audit columns.

### ProductModelProductDescription
- Deduplication key: composite of `ProductModelID`, `ProductDescriptionID`, `Culture`
- Rename columns to snake_case.
- Validate foreign keys exist.
- Add audit columns.

### SalesOrderHeader
- Deduplication key: `SalesOrderID`
- Rename columns to snake_case.
- Convert timestamps consistently.
- Validate foreign keys: `customer_id`, `ship_to_address_id`, `bill_to_address_id`.
- Add audit columns.
- Partition by `order_date` (date part).

### SalesOrderDetail
- Deduplication key: `SalesOrderDetailID`
- Rename columns to snake_case.
- Validate foreign key `product_id` exists.
- Add audit columns.
- Partition by `modified_date` or join to header for `order_date`.

### Views (vGetAllCategories, vProductAndDescription, vProductModelCatalogDescription)
- Ingest as read-only silver tables.
- Rename columns to snake_case.
- Use for enrichment in gold layer.

- Run `OPTIMIZE` on all silver tables with ZORDER on primary keys.

## Gold
- Build dimensional and fact tables based on actual columns and relationships.

### Dimensions
- `dim_customer` from `Customer`:
  - PK: `customer_id`
  - Attributes: full name fields, company, salesperson, email, phone.
- `dim_address` from `Address`:
  - PK: `address_id`
  - Attributes: address lines, city, state, country, postal code.
- `dim_product` from `Product` joined with `ProductCategory`, `ProductModel`, and descriptions from views:
  - PK: `product_id`
  - Attributes: name, product number, color, size, weight, category name, model name, descriptions.
- `dim_product_category` from `ProductCategory`:
  - PK: `product_category_id`
  - Attributes: name, parent category id.
- `dim_date`:
  - Build a date dimension covering all relevant dates from orders and products.
- `dim_salesperson` (optional):
  - Extract unique salespersons from `Customer.SalesPerson` if meaningful. --> very meaningfull

### Fact tables
- `fact_sales_order` from `SalesOrderHeader` joined with `SalesOrderDetail`:
  - Grain: one row per sales order line (`SalesOrderDetailID`).
  - FK: `sales_order_id`, `customer_id`, `product_id`, `ship_to_address_id`, `bill_to_address_id`.
  - Measures: `order_qty`, `unit_price`, `unit_price_discount`, `line_total`, `subtotal`, `tax_amt`, `freight`, `total_due`.
  - Dates: `order_date`, `due_date`, `ship_date`.
- Alternative modeling:
  - Separate fact tables for header and detail if needed. --> not needed

### Data quality tests
- Row count > 0 for all gold tables.
- No nulls in PK columns.
- Unique PK enforcement.
- Referential integrity checks:
  - `fact_sales_order.customer_id` exists in `dim_customer`.
  - `fact_sales_order.product_id` exists in `dim_product`.
  - `dim_product.product_category_id` exists in `dim_product_category`.
- No negative or zero values in quantity and price measures.
- Consistency checks on dates (e.g., `ship_date` ≥ `order_date`).

## Semantic model
- Star schema with:
  - Fact table: `fact_sales_order`
  - Dimension tables: `dim_customer`, `dim_address`, `dim_product`, `dim_product_category`, `dim_date`
- Relationships:
  - `fact_sales_order.customer_id` → `dim_customer.customer_id`
  - `fact_sales_order.product_id` → `dim_product.product_id`
  - `fact_sales_order.ship_to_address_id` → `dim_address.address_id`
  - `fact_sales_order.bill_to_address_id` → `dim_address.address_id`
  - Date keys linked to `dim_date` on `order_date`, `due_date`, `ship_date`
- Measures (DAX):
  - Total Sales = SUM(`fact_sales_order[line_total]`)
  - Total Quantity = SUM(`fact_sales_order[order_qty]`)
  - Average Unit Price = AVERAGE(`fact_sales_order[unit_price]`)
  - Total Discount = SUM(`fact_sales_order[unit_price_discount] * fact_sales_order[order_qty]`)
  - Total Freight = SUM(`fact_sales_order[freight]`)
  - Total Tax = SUM(`fact_sales_order[tax_amt]`)
- Calculated columns:
  - Order Profit Margin (if cost data available or approximated)
- Use explicit measure formatting and descriptions.

## Report
- Page 1: Sales Overview
  - Visuals: Total Sales by Month (line chart), Sales by Product Category (bar chart), Total Quantity Sold (card)
- Page 2: Customer Insights
  - Visuals: Top Customers by Sales (table), Sales by Salesperson (bar chart), Customer Geographic Distribution (map using address city/state)
- Page 3: Product Performance
  - Visuals: Sales by Product (table), Product List Price vs. Sales Price (scatter), Product Category Hierarchy (tree map)
- Page 4: Data Quality Dashboard
  - Visuals: Row counts per table (cards), Null PK counts (cards), Referential integrity failures (tables), Data freshness (last modified timestamps)
- Use slicers for date range, product category, and customer region.

## Data Agent
- Role: Sales and Product Data Analyst Assistant
- Domain: Sales orders, customers, products, and addresses in SalesLT domain
- Starter questions:
  1. What were the total sales and quantities last quarter?
  2. Which customers generated the highest revenue?
  3. How do sales vary by product category and region?
  4. What is the average discount applied on sales orders?
  5. Show me the sales trend for a specific product over time.
  6. Are there any data quality issues in the sales or customer data?
  7. What is the distribution of orders by shipping method?
  8. How many active products are currently selling?
  9. Which salespersons have the highest sales volumes?
  10. Provide details for orders shipped late compared to due date.
- Guardrails:
  - Only answer questions related to the SalesLT domain data.
  - Do not expose sensitive fields like passwords.
  - Validate question context against semantic model entities.
  - Provide data quality warnings if underlying data issues detected.
  - Avoid speculative or out-of-scope answers.
  - Encourage user to refine ambiguous queries.