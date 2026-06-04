# Run Spec 20260604-133728-ace6dc

## Updated specs

### Iteration 1 — 2026-06-04 13:47:13Z — failed layer: reporting (run: 20260604-133801-57e9a5)
- **Root cause (1-line summary)**: Reporting-stage Spark session failed without a column-level traceback; the most likely reporting risk is unresolved semantic-model field references caused by display names not matching Gold table column names.
- **Cross-table audit**:
  - Address: no — source table is not directly referenced by report measures.
  - Customer: yes — customer-related report visuals depend on semantic-model field mappings.
  - CustomerAddress: no — intentionally bypassed in Gold and reporting.
  - Product: yes — product visuals require consistent semantic-model field mappings.
  - ProductCategory: yes — category hierarchy feeds product reporting.
  - ProductDescription: yes — product description attributes may surface in reporting.
  - ProductModel: yes — product dimension construction depends on this source.
  - ProductModelProductDescription: yes — product dimension construction depends on this source.
  - SalesOrderDetail: yes — all sales measures originate from fact-level fields.
  - SalesOrderHeader: yes — order, date, customer, and freight-related reporting fields originate here.
- **Fix approach**: GENERALIZE — the failure occurred in the reporting layer and could affect any report visual or measure that references friendly names instead of actual semantic-model fields.
- **What was changed**:
  - Tightened the Semantic model section with explicit table and field naming requirements.
  - Added explicit measure-to-column mappings and required fact-column names.
  - Added report-authoring constraints requiring visuals to bind only to validated semantic-model fields.

## Inputs
- Workspace: `8f7be003-abaa-46b6-8b1c-0dba2cfc3bc6`
- Source Lakehouse: **SalesLT** (`47f5fdf7-1902-471b-958f-5a1e9430070e`)
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
- Target Lakehouse: **g**

## Generic guidance

Apply these reference skills/agents at all times:
- FabricDataEngineer agent: https://github.com/microsoft/skills-for-fabric/blob/main/agents/FabricDataEngineer.agent.md
- e2e-medallion-architecture skill: https://github.com/microsoft/skills-for-fabric/tree/main/skills/e2e-medallion-architecture
- spark-authoring-cli skill: https://github.com/microsoft/skills-for-fabric/tree/main/skills/spark-authoring-cli
- powerbi-authoring-cli skill: https://github.com/microsoft/skills-for-fabric/tree/main/skills/powerbi-authoring-cli
- powerbi-consumption-cli skill: https://github.com/microsoft/skills-for-fabric/tree/main/skills/powerbi-consumption-cli
- powerbi-semantic-model-authoring: https://github.com/RuiRomano/powerbi-agentic-plugins/tree/main/plugins/powerbi/skills/powerbi-semantic-model-authoring
- powerbi-report-authoring: https://github.com/RuiRomano/powerbi-agentic-plugins/tree/main/plugins/powerbi/skills/powerbi-report-authoring

## Bronze

Land each source table into the `bronze` schema as Delta tables with schema preservation.

Common metadata columns:
- `ingestion_timestamp`
- `run_id`
- `source_table`
- `source_file_modified_time` (if available)
- `bronze_loaded_at`

Write mode:
- Initial load: overwrite
- Subsequent loads: merge/upsert where practical using business keys and `ModifiedDate`
- Preserve all source columns including binary fields

Partitioning:
- `SalesOrderHeader`: partition by year/month derived from `OrderDate`
- `SalesOrderDetail`: partition by year/month derived after joining to header during optimization strategy, otherwise unpartitioned initially
- All other tables: unpartitioned due to expected small dimension size

Table-specific landing intent:
- `Address`: retain address attributes for regional analysis
- `Customer`: retain customer profile and salesperson assignment
- `CustomerAddress`: retain for lineage purposes even though Gold design intentionally bypasses it
- `Product`: retain product pricing, category, model, lifecycle dates
- `ProductCategory`: retain parent-child hierarchy
- `ProductDescription`: retain description text
- `ProductModel`: retain model records; NOTE: requested ProductModel.Name is not present in the supplied schema, so Gold will use available keys only unless schema is extended
- `ProductModelProductDescription`: retain culture-specific description mapping
- `SalesOrderHeader`: retain order-level transactional attributes
- `SalesOrderDetail`: retain line-level sales metrics

## Silver

Apply to all tables:
- Rename columns to snake_case
- Standardize timestamps to UTC
- Trim string values
- Add:
  - `silver_created_at`
  - `silver_updated_at`
  - `record_source`
- OPTIMIZE and VACUUM according to Fabric best practices

Deduplication strategy:

- `address`
  - Key: `address_id`
  - Keep latest by `modified_date`

- `customer`
  - Key: `customer_id`
  - Keep latest by `modified_date`
  - Normalize email casing

- `customer_address`
  - Key: (`customer_id`, `address_id`)
  - Keep latest by `modified_date`

- `product`
  - Key: `product_id`
  - Keep latest by `modified_date`

- `product_category`
  - Key: `product_category_id`
  - Keep latest by `modified_date`

- `product_description`
  - Key: `product_description_id`
  - Keep latest by `modified_date`

- `product_model`
  - Key: `product_model_id`
  - Keep latest by `modified_date`

- `product_model_product_description`
  - Key: (`product_model_id`, `product_description_id`, `culture`)
  - Keep latest by `modified_date`

- `sales_order_header`
  - Key: `sales_order_id`
  - Keep latest by `modified_date`

- `sales_order_detail`
  - Key: `sales_order_detail_id`
  - Keep latest by `modified_date`

Business transformations:
- Extract salesperson username from `sales_person`
  - Example: `adventure-works\jillian0` → `jillian0`
  - Remove domain prefix when present
- Create normalized city field from Address
- Filter product-description bridge candidates for `culture='en'` during Gold preparation
- Exclude password-related fields from Gold consumption layers:
  - `password_hash`
  - `password_salt`

## Gold

Create a business-oriented star schema optimized for sales analytics.

Dimension: `dim_order_date`
- Source: `sales_order_header.order_date`
- Grain: one row per calendar date
- Attributes:
  - date key
  - full date
  - year
  - quarter
  - month
  - month name
  - week
  - day
- Hierarchy:
  - Year → Quarter → Month → Date

Dimension: `dim_ship_date`
- Source: `sales_order_header.ship_date`
- Grain: one row per calendar date
- Attributes equivalent to Order Date dimension
- Hierarchy:
  - Year → Quarter → Month → Date

Dimension: `dim_customer`
- Source: Customer + Address
- User requirement implemented: combine Customer and Address directly; CustomerAddress is not used
- Join assumption:
  - No direct key exists between Customer and Address in supplied schema.
  - Gold notebook must attempt direct relationship discovery only if a valid key becomes available at runtime.
  - If no direct relationship exists, populate customer attributes and regional attributes separately and flag lineage note.
- Keep relevant fields:
  - customer identifier
  - company name
  - title
  - suffix
  - email address
  - city
  - postal code
- Exclude:
  - password fields
  - rowguid values
- Hierarchies:
  - City → Customer

Dimension: `dim_salesperson`
- Source: Customer.sales_person
- Grain: one row per salesperson
- Transform:
  - Remove domain prefix
  - Store username value
- Attributes:
  - salesperson key
  - salesperson username
- Hierarchy:
  - Salesperson

Dimension: `dim_order`
- Source: SalesOrderHeader
- Grain: one row per order
- Move descriptive attributes out of fact:
  - sales_order_id
  - revision_number
  - status
  - ship_method
  - credit_card_approval_code
  - comment
- Hierarchy:
  - Status → Order

Dimension: `dim_product`
- Source:
  - Product
  - ProductCategory
  - ProductDescription
  - ProductModel
  - ProductModelProductDescription
- Transformations:
  - Use ProductCategory parent-child relationship to expose category and subcategory
  - Join Product -> ProductModel
  - Join ProductModel -> ProductModelProductDescription
  - Filter ProductModelProductDescription to `culture='en'`
  - Join ProductDescription to retrieve Description
- NOTE:
  - User requested ProductModel.Name as modelname.
  - The supplied ProductModel schema contains only `ProductModelID` and no Name column.
  - Model name cannot be populated unless additional source columns are provided.
- Keep relevant fields:
  - product identifier
  - product number
  - color
  - size
  - weight
  - standard cost
  - list price
  - product description
  - category
  - subcategory
  - product model id
  - sell start/end dates
  - discontinued date
- Hierarchies:
  - Category → Subcategory → Product
  - Category → Subcategory → Product Number

Fact: `fact_sales_order`
- Grain: one row per sales order detail line
- Source:
  - SalesOrderDetail
  - SalesOrderHeader
- Joins:
  - SalesOrderDetail.sales_order_id = SalesOrderHeader.sales_order_id
  - Product via product_id
  - Customer via customer_id
  - Salesperson via customer assignment
  - Order dimension via sales_order_id
  - Order Date via order_date
  - Ship Date via ship_date
- Keep relevant measures:
  - order quantity
  - unit price
  - unit price discount
  - extended sales amount
  - gross sales amount
  - discount amount
  - net sales amount
  - tax amount allocation
  - freight allocation
- Derived calculations:
  - Gross Sales = OrderQty * UnitPrice
  - Discount Amount = OrderQty * UnitPrice * UnitPriceDiscount
  - Net Sales = Gross Sales - Discount Amount
  - Discount Percentage = UnitPriceDiscount * 100
- Fact foreign keys:
  - order_date_key
  - ship_date_key
  - customer_key
  - salesperson_key
  - order_key
  - product_key

## Test

Each test writes one result row to:
- `test/test_results`

Schema:
- run_id
- test_name
- layer
- table_name
- status
- actual
- expected
- details
- checked_at

Standard tests:

1. Row Count Reconciliation
- Layer: Bronze→Silver
- Verify row counts remain within 1%
- Tables:
  - address
  - customer
  - customer_address
  - product
  - product_category
  - product_description
  - product_model
  - product_model_product_description
  - sales_order_header
  - sales_order_detail

2. Gold Dimension PK Not Null
- Validate:
  - dim_customer.customer_key
  - dim_salesperson.salesperson_key
  - dim_product.product_key
  - dim_order.order_key
  - dim_order_date.date_key
  - dim_ship_date.date_key

3. Gold Dimension PK Uniqueness
- Verify uniqueness of all dimension primary keys

4. Referential Integrity
- fact_sales_order.product_key exists in dim_product
- fact_sales_order.customer_key exists in dim_customer
- fact_sales_order.salesperson_key exists in dim_salesperson
- fact_sales_order.order_key exists in dim_order
- fact_sales_order.order_date_key exists in dim_order_date
- fact_sales_order.ship_date_key exists in dim_ship_date

5. Business Rule Sanity Checks
- Net Sales >= 0
- Order Quantity > 0
- Discount Percentage between 0 and 100
- Ship Date is null or Ship Date >= Order Date
- Gross Sales >= Net Sales

## Semantic model

Mode:
- Direct Lake

Tables:
- Fact Sales Order (source table: `fact_sales_order`)
- Customer (source table: `dim_customer`)
- SalesPerson (source table: `dim_salesperson`)
- Product (source table: `dim_product`)
- Order (source table: `dim_order`)
- OrderDate (source table: `dim_order_date`)
- ShipDate (source table: `dim_ship_date`)

Relationships:
- Fact Sales Order → Customer
- Fact Sales Order → SalesPerson
- Fact Sales Order → Product
- Fact Sales Order → Order
- Fact Sales Order → OrderDate
- Fact Sales Order → ShipDate

Field binding requirements:
- Semantic-model measures and report visuals must reference actual Gold columns, not display captions.
- Required fact columns:
  - `sales_order_id`
  - `order_quantity`
  - `gross_sales_amount`
  - `discount_amount`
  - `net_sales_amount`
  - `discount_percentage`
  - `freight_allocation`
  - `product_key`
  - `customer_key`
  - `salesperson_key`
- If a display name is created, retain the underlying column reference and validate existence before publishing.

Hierarchies:

OrderDate:
- Year → Quarter → Month → Date

ShipDate:
- Year → Quarter → Month → Date

Product:
- Category → Subcategory → Product

Customer:
- City → Customer

Measures:

- Total Sales =
  Sum(`net_sales_amount`)

- Gross Sales =
  Sum(`gross_sales_amount`)

- Total Discount Amount =
  Sum(`discount_amount`)

- Average Sales =
  Average(`net_sales_amount`)

- Maximum Sales =
  Max(`net_sales_amount`)

- Total Orders =
  DistinctCount(`sales_order_id`)

- Total Quantity Sold =
  Sum(`order_quantity`)

- Average Discount Percentage =
  Average(`discount_percentage`)

- Maximum Discount Percentage =
  Max(`discount_percentage`)

- Average Order Value =
  Divide([Total Sales], [Total Orders])

- Average Freight =
  Average(`freight_allocation`)

- Maximum Order Value =
  Max(`net_sales_amount`)

- Distinct Customers =
  DistinctCount(`customer_key`)

- Distinct Products Sold =
  DistinctCount(`product_key`)

## Report

Before creating visuals:
- Validate that every referenced table, hierarchy, measure, and field exists in the published semantic model.
- Do not bind visuals to inferred field names, friendly captions, or untranslated business labels unless the corresponding semantic-model object exists.
- If a requested field is unavailable, omit the visual dependency and record the limitation rather than failing report generation.

### Page 1: Executive Sales Overview

Visuals:
- KPI: Total Sales
- KPI: Average Sales
- KPI: Maximum Sales
- KPI: Total Orders
- KPI: Average Order Value
- Monthly sales trend line chart using OrderDate hierarchy
- Sales by Category clustered column chart
- Top 10 Products by Sales bar chart

### Page 2: Regional Performance

Visuals:
- Map visual using Customer city and postal code
- Bubble size: Total Sales
- Color scale: Average Sales
- Tooltip:
  - Total Sales
  - Average Sales
  - Maximum Sales
  - Total Orders
- Ranked bar chart:
  - Cities by Total Sales
- Matrix:
  - City
  - Customer
  - Total Sales
  - Average Sales
  - Maximum Sales

Note:
- Geographic analysis quality depends on successful customer-to-address association. If customer-address linkage cannot be established from available source keys, regional reporting should be flagged for data model review.

### Page 3: Orders and Discounts

Visuals:
- Order status distribution
- Monthly order count trend
- Salesperson ranking by Total Discount Amount
- Salesperson ranking by Average Discount Percentage
- Salesperson ranking by Maximum Discount Percentage
- Scatter chart:
  - X = Average Discount Percentage
  - Y = Total Sales
  - Size = Total Orders
  - Category = Salesperson

### Page 4: Product Performance

Visuals:
- Category → Subcategory drilldown chart
- Product sales contribution treemap
- Product margin proxy:
  - Gross Sales vs Standard Cost comparison
- Top and bottom products by sales

### Page 5: Data Quality

Visuals:
- Test pass/fail summary
- Latest test execution table
- Referential integrity results
- Row count reconciliation results
- Business rule exceptions

## Data Agent

Role:
- Sales Performance Intelligence Agent for the SalesLT sales analytics platform.
- Ground all responses in the semantic model.
- Help users understand sales performance, regional trends, product performance, order activity, discount behavior, and salesperson effectiveness.

Domain hints:
- Sales data is modeled at sales-order-line grain.
- Regional analysis is based on customer/address information when available.
- Product hierarchy includes category and subcategory.
- Date analysis can be performed by Order Date and Ship Date.
- Discount analysis uses line-level discount percentages and discount amounts.

Starter questions:
- Which regions generated the highest total sales?
- Which regions generated the lowest total sales?
- What are the monthly sales trends over time?
- What is the average sales value by region?
- What is the maximum sales value by region?
- Which salespeople provide the largest discounts?
- Which salespeople have the highest average discount percentage?
- Which product categories drive the most revenue?
- What are the top 10 products by sales?
- How do Order Date and Ship Date trends compare?
- Which customers generate the most revenue?
- Are there any unusual discount patterns?

Guardrails:
- Use only data exposed through the semantic model.
- Do not infer missing customer-address relationships not supported by data.
- Clearly identify when requested information is unavailable.
- Prefer aggregated reporting over row-level operational detail.
- Surface date context used in calculations.
- State filters applied in answers.
- Do not expose password-related source fields.
- Do not fabricate product model names because they are not present in the supplied schema.
- When discussing regional performance, indicate any data-quality limitations affecting geographic attribution.
- Provide numerical results with supporting dimensions, measures, and filter context.