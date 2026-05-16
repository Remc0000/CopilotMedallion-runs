# Run Spec

## Updated specs

### Iteration 1 — 2026-05-16 19:08:20Z — failed layer: gold (run: 20260516-150519-5ebfdd)
- **Root cause (1-line summary)**: Gold layer likely failed because the spec allowed non-unique customer grain and insufficiently explicit gold join/output keys, causing invalid dimension/fact shaping in the final model build.
- **What was changed**:
  - Tightened **Generic guidance** to require explicit aliasing, deterministic output schemas, and model-safe unique dimension keys in gold.
  - Tightened **Silver** customer and sales prep so the exact customer/address and salesperson columns needed by gold are always present with stable names.
  - Added explicit **Gold** build rules, table schemas, and join keys for `gold_dim_customer`, `gold_dim_product`, `gold_dim_sales`, and `gold_fact_sales`, including a required single-row-per-`CustomerID` customer dimension.

## Inputs
Dim_Customer -> join customer, customer address, address and leave all relevant/meaningfull fields
Dim_Product -> join product, productcategory, producdescription, productmodel, productmodelproductdescription. Please notice that in productcategory there is a self join. So in the end I have the category (which is the parent category) and a subcategory.
Dim_Sales --> Use the salesperson field from salesorderheader, but remove the adventureworks/ in front of the name and remove the last number at the end off the name
Fact_Sales --> join salesorderdetail and salesorderheader and keep all relevant fields

## Generic guidance
Use Medallion architecture
Use Microsoft Fabric
Use dimensional modelling
Keep bronze close to source, silver for preparation, and gold for model-ready dimensional tables.
For every join outside bronze, use explicit table aliases and fully qualified column references to avoid ambiguous or unresolved column selection.
Do not use `select *` in silver or gold. Project the final required columns explicitly and rename them to stable business-friendly names.
Gold tables must be model-safe:
- each dimension must have a deterministic primary business key column
- `gold_dim_customer` must contain exactly one row per `CustomerID`
- `gold_dim_product` must contain exactly one row per `ProductID`
- `gold_dim_sales` must contain exactly one row per `SalesPersonNameClean`
- `gold_fact_sales` must remain at sales order line grain: one row per `SalesOrderDetailID`
If multiple source rows exist for a dimension business key, resolve them in silver/gold with an explicit rule rather than allowing duplicates to flow into the semantic model.
When a requested source attribute does not physically exist in the listed AdventureWorks source tables, do not invent additional joins; only include that attribute if present in the ingested tables.

## Bronze
### goal
Store the source data in its original form in Fabric so it can be consumed by downstream layers with full fidelity.

### source
AdventureWorks tables required to satisfy the requested dimensional model:
- Customer
- CustomerAddress
- Address
- Product
- ProductCategory
- ProductDescription
- ProductModel
- ProductModelProductDescription
- SalesOrderHeader
- SalesOrderDetail

### destination
Fabric Lakehouse bronze layer tables, one table per source object.

### logic
- Ingest each source table as-is from AdventureWorks into the bronze lakehouse.
- Preserve source column names.
- Do not apply business transformations in bronze.
- Add standard ingestion metadata where available, such as load timestamp.

### output tables
- bronze_customer
- bronze_customeraddress
- bronze_address
- bronze_product
- bronze_productcategory
- bronze_productdescription
- bronze_productmodel
- bronze_productmodelproductdescription
- bronze_salesorderheader
- bronze_salesorderdetail

## Silver
### goal
Clean, standardize, and prepare source entities for dimensional modelling in gold.

### entities and transformations

#### Customer preparation
Create a prepared customer dataset from:
- bronze_customer
- bronze_customeraddress
- bronze_address

Transformations:
- Standardize key column names for downstream joins.
- Join customer to customeraddress on `CustomerID`.
- Join customeraddress to address on `AddressID`.
- Use explicit aliases: `c` for customer, `ca` for customeraddress, `a` for address.
- Retain meaningful customer and address attributes needed for a customer dimension.
- Deduplicate only exact technical duplicates.
- Keep one row per valid customer-address relationship in this prepared table.
- Ensure the prepared output contains the exact downstream columns `CustomerID` and `AddressID`.
- If `AddressType` exists in `bronze_customeraddress`, keep it as `AddressType`. If it does not exist, omit it rather than deriving it.
- If address-related columns such as `StateProvince` do not exist in the ingested `Address` table, do not fabricate them.

Expected prepared fields include:
- CustomerID
- AddressID
- AddressType if available
- Customer-level descriptive attributes available in source
- AddressLine1
- AddressLine2
- City
- StateProvince if available through address-related source columns present in the ingested tables
- PostalCode

Suggested output:
- silver_customer_prepared

Additional requirement for downstream gold:
- `silver_customer_prepared` may contain multiple rows per `CustomerID`, but gold must resolve this to a single chosen row per `CustomerID`.
- Include enough columns for deterministic gold row selection. If available, prefer using `AddressID` as the tie-breaker for customer row selection.

#### Product preparation
Create a prepared product dataset from:
- bronze_product
- bronze_productcategory
- bronze_productdescription
- bronze_productmodel
- bronze_productmodelproductdescription

Transformations:
- Join product to productmodel on `ProductModelID`.
- Join productmodel to productmodelproductdescription on `ProductModelID`.
- Join productmodelproductdescription to productdescription on `ProductDescriptionID`.
- Use explicit aliases: `p` for product, `pm` for productmodel, `pmpd` for productmodelproductdescription, `pd` for productdescription, `pc_child` for child category, `pc_parent` for parent category.
- Resolve product category hierarchy using a self-join on productcategory:
  - child category represents `Subcategory`
  - parent category represents `Category`
- Join product to the child productcategory using `p.ProductSubcategoryID = pc_child.ProductCategoryID`.
- Join child category to parent category using `pc_child.ParentProductCategoryID = pc_parent.ProductCategoryID`.
- Rename product name to `ProductName`.
- Rename model name to `ProductModelName`.
- Rename description text to `ProductDescription`.
- Retain one row per `ProductID`. If multiple descriptions exist for the same product model, select one deterministic row per `ProductID` in silver or gold.

Expected prepared fields include:
- ProductID
- ProductName
- ProductNumber
- Color
- StandardCost
- ListPrice
- Size
- Weight
- ProductModelID
- ProductModelName
- ProductDescription
- Subcategory
- Category

Suggested output:
- silver_product_prepared

#### Salesperson preparation
Create a prepared salesperson dataset from:
- bronze_salesorderheader

Transformations:
- Extract distinct salesperson values from `SalesOrderHeader.SalesPerson`.
- Clean the salesperson string by:
  - removing the `adventureworks/` prefix
  - removing trailing numeric suffix at the end of the name
- Trim whitespace after cleansing.
- Exclude null or empty salesperson values from the dimension output if they do not represent valid salespeople.
- Keep the raw source column as `SalesPersonRaw`.
- Output the cleaned value as `SalesPersonNameClean`.

Expected prepared fields include:
- SalesPersonRaw
- SalesPersonNameClean

Suggested output:
- silver_salesperson_prepared

#### Sales fact preparation
Create a prepared sales dataset from:
- bronze_salesorderdetail
- bronze_salesorderheader

Transformations:
- Join salesorderdetail to salesorderheader on `SalesOrderID`.
- Use explicit aliases: `d` for salesorderdetail and `h` for salesorderheader.
- Preserve line-level grain from `salesorderdetail`, with one row per `SalesOrderDetailID`.
- Standardize key names and date fields for downstream fact modelling.
- Carry the raw salesperson value from header as `SalesPersonRaw`.
- Also create `SalesPersonNameClean` in this table using the same cleansing rule used in `silver_salesperson_prepared`, so gold fact can join consistently to `gold_dim_sales` on the cleaned name.
- Explicitly project only the required detail and header fields.

Expected prepared fields include:
- SalesOrderID
- SalesOrderDetailID
- OrderDate
- DueDate
- ShipDate
- CustomerID
- SalesPersonRaw
- SalesPersonNameClean
- ProductID
- OrderQty
- UnitPrice
- UnitPriceDiscount
- LineTotal
- CarrierTrackingNumber if available
- SalesOrderNumber
- PurchaseOrderNumber if available
- AccountNumber if available
- Status if available
- OnlineOrderFlag if available
- SubTotal if available at header level
- TaxAmt if available at header level
- Freight if available at header level
- TotalDue if available at header level

Suggested output:
- silver_sales_prepared

Dim_Customer -> join customer, customer address, address and leave all relevant/meaningfull fields
Dim_Product -> join product, productcategory, producdescription, productmodel, productmodelproductdescription. Please notice that in productcategory there is a self join. So in the end I have the category (which is the parent category) and a subcategory.
Dim_Sales --> Use the salesperson field from salesorderheader, but remove the adventureworks/ in front of the name and remove the last number at the end off the name
Fact_Sales --> join salesorderdetail and salesorderheader and keep all relevant fields

## Gold
### goal
Create model-ready star schema tables in Fabric with unique dimension keys and a line-grain sales fact.

### output tables
- gold_dim_customer
- gold_dim_product
- gold_dim_sales
- gold_fact_sales

### logic

#### gold_dim_customer
Build from `silver_customer_prepared`.

Rules:
- This table must contain exactly one row per `CustomerID`.
- Because `silver_customer_prepared` may contain multiple rows per customer due to multiple addresses, choose a single deterministic representative row per `CustomerID`.
- Use this precedence for row selection:
  1. lowest `AddressID` for the customer, if `AddressID` is present
  2. otherwise a deterministic stable row choice based on available columns
- Do not keep multiple address rows in `gold_dim_customer`.
- Include these columns when available:
  - `CustomerID`
  - `AddressID`
  - `AddressType`
  - customer descriptive attributes from source
  - `AddressLine1`
  - `AddressLine2`
  - `City`
  - `StateProvince`
  - `PostalCode`

Primary key:
- `CustomerID`

#### gold_dim_product
Build from `silver_product_prepared`.

Rules:
- This table must contain exactly one row per `ProductID`.
- If multiple source/prepared rows exist for the same `ProductID` because of multiple model descriptions, choose one deterministic row per `ProductID`.
- Include these columns:
  - `ProductID`
  - `ProductName`
  - `ProductNumber`
  - `Color`
  - `StandardCost`
  - `ListPrice`
  - `Size`
  - `Weight`
  - `ProductModelID`
  - `ProductModelName`
  - `ProductDescription`
  - `Subcategory`
  - `Category`

Primary key:
- `ProductID`

#### gold_dim_sales
Build from `silver_salesperson_prepared`.

Rules:
- This table must contain exactly one row per `SalesPersonNameClean`.
- Exclude null or blank `SalesPersonNameClean`.
- If multiple raw values map to the same cleaned name, keep one deterministic representative raw value and one row per cleaned name.

Include these columns:
- `SalesPersonRaw`
- `SalesPersonNameClean`

Primary key:
- `SalesPersonNameClean`

#### gold_fact_sales
Build from `silver_sales_prepared`.

Rules:
- Grain must remain one row per `SalesOrderDetailID`.
- Do not join gold fact to the multi-row customer prepared table.
- Fact should be built directly from `silver_sales_prepared`, which already contains the required header and detail fields.
- Keep `CustomerID`, `ProductID`, and `SalesPersonNameClean` in the fact exactly with those column names so the semantic model can create deterministic relationships.
- `SalesPersonRaw` may also be retained for lineage/troubleshooting.
- Before publishing, validate that every fact row has at most one matching row in each gold dimension on:
  - `CustomerID`
  - `ProductID`
  - `SalesPersonNameClean`

Include these columns:
- `SalesOrderID`
- `SalesOrderDetailID`
- `SalesOrderNumber`
- `OrderDate`
- `DueDate`
- `ShipDate`
- `CustomerID`
- `ProductID`
- `SalesPersonRaw`
- `SalesPersonNameClean`
- `OrderQty`
- `UnitPrice`
- `UnitPriceDiscount`
- `LineTotal`
- `CarrierTrackingNumber`
- `PurchaseOrderNumber`
- `AccountNumber`
- `Status`
- `OnlineOrderFlag`
- `SubTotal`
- `TaxAmt`
- `Freight`
- `TotalDue`

Primary key:
- `SalesOrderDetailID`

## Semantic model
### goal
Create a Fabric semantic model on top of the gold model for sales analysis by customer, product hierarchy, and cleaned salesperson.

### model tables
- Dim_Customer from gold_dim_customer
- Dim_Product from gold_dim_product
- Dim_Sales from gold_dim_sales
- Fact_Sales from gold_fact_sales

### relationships
Define single-direction, many-to-one relationships from Fact_Sales to dimensions using the business keys present in gold:
- Fact_Sales[CustomerID] -> Dim_Customer[CustomerID]
- Fact_Sales[ProductID] -> Dim_Product[ProductID]
- Fact_Sales[SalesPersonNameClean] -> Dim_Sales[SalesPersonNameClean]

`gold_dim_customer` must already be unique by `CustomerID` before creating the semantic relationship. Do not create the semantic relationship if multiple rows per `CustomerID` remain.
The semantic model must only use active relationships that preserve deterministic filtering.

### dimensions and attributes

#### Dim_Customer
Expose the meaningful customer and address fields carried into gold_dim_customer, including:
- CustomerID
- AddressID
- AddressType
- AddressLine1
- AddressLine2
- City
- StateProvince
- PostalCode
- Any additional descriptive customer attributes included in gold_dim_customer

Default usage:
- Use for slicing sales by customer and customer location.

#### Dim_Product
Expose:
- ProductID
- ProductName
- ProductNumber
- Color
- StandardCost
- ListPrice
- Size
- Weight
- ProductModelID
- ProductModelName
- ProductDescription
- Category
- Subcategory

Recommended hierarchy:
- Category
- Subcategory
- ProductName

#### Dim_Sales
Expose:
- SalesPersonRaw
- SalesPersonNameClean

Default usage:
- Use SalesPersonNameClean in slicers and visuals.
- Hide SalesPersonRaw from report consumers if it is only required for lineage or troubleshooting.

#### Fact_Sales
Expose all relevant joined header and detail fields present in gold_fact_sales, including:
- SalesOrderID
- SalesOrderDetailID
- SalesOrderNumber
- OrderDate
- DueDate
- ShipDate
- CustomerID
- ProductID
- SalesPersonRaw
- SalesPersonNameClean
- OrderQty
- UnitPrice
- UnitPriceDiscount
- LineTotal
- CarrierTrackingNumber
- PurchaseOrderNumber
- AccountNumber
- Status
- OnlineOrderFlag
- SubTotal
- TaxAmt
- Freight
- TotalDue

### measures
Create at minimum these DAX measures:
- Sales Amount = SUM(Fact_Sales[LineTotal])
- Order Quantity = SUM(Fact_Sales[OrderQty])
- Average Unit Price = AVERAGE(Fact_Sales[UnitPrice])
- Discount Amount = SUMX(Fact_Sales, Fact_Sales[OrderQty] * Fact_Sales[UnitPrice] * Fact_Sales[UnitPriceDiscount])
- Distinct Orders = DISTINCTCOUNT(Fact_Sales[SalesOrderID])
- Distinct Customers = DISTINCTCOUNT(Fact_Sales[CustomerID])

Optional additional measures if header-level amounts are intentionally stored on each line and accepted for reporting:
- Total Tax = SUM(Fact_Sales[TaxAmt])
- Total Freight = SUM(Fact_Sales[Freight])
- Total Due = SUM(Fact_Sales[TotalDue])

### formatting and modelling guidance
- Format Sales Amount, Average Unit Price, Discount Amount, Total Tax, Total Freight, SubTotal, and Total Due as currency where applicable.
- Format Order Quantity, Distinct Orders, and Distinct Customers as whole numbers.
- Set ProductName, Category, Subcategory, City, StateProvince, PostalCode, and SalesPersonNameClean as searchable columns where useful for slicers.
- Hide technical or raw fields not intended for end-user reporting.

## Report
### goal
Build a Fabric Power BI report focused on sales performance using Fact_Sales with analysis by customer, product category/subcategory, and cleaned salesperson.

### pages

#### 1. Sales Overview
Visuals:
- KPI card: Sales Amount
- KPI card: Distinct Orders
- KPI card: Order Quantity
- Line chart: Sales Amount by OrderDate
- Clustered column chart: Sales Amount by Category
- Bar chart: Sales Amount by SalesPersonNameClean
- Donut or column chart: Sales Amount by OnlineOrderFlag
- Matrix: Category, Subcategory, ProductName with Sales Amount and Order Quantity

Slicers:
- OrderDate
- Category
- Subcategory
- SalesPersonNameClean

#### 2. Product Performance
Visuals:
- Matrix: Category > Subcategory > ProductName with Sales Amount, Order Quantity, Average Unit Price
- Bar chart: Top 10 ProductName by Sales Amount
- Scatter chart: Sales Amount vs Order Quantity by ProductName
- Table: ProductName, ProductNumber, Color, ProductModelName, ProductDescription, Sales Amount

Slicers:
- Category
- Subcategory
- Color
- ProductModelName

#### 3. Customer and Location Analysis
Visuals:
- Bar chart: Sales Amount by City
- Bar chart: Sales Amount by StateProvince
- Table or matrix: CustomerID, AddressLine1, City, StateProvince, PostalCode, Sales Amount, Distinct Orders
- Map visual if City and StateProvince are suitable for geographic categorization

Slicers:
- City
- StateProvince
- PostalCode
- AddressType

Note:
- Use the single-row-per-customer `Dim_Customer` gold table for customer/location visuals.

#### 4. Salesperson Performance
Visuals:
- Bar chart: Sales Amount by SalesPersonNameClean
- Table: SalesPersonNameClean, Sales Amount, Distinct Orders, Order Quantity, Average Unit Price
- Line chart: Sales Amount by OrderDate split by SalesPersonNameClean

Slicers:
- SalesPersonNameClean
- OrderDate

### report design guidance
- Use SalesPersonNameClean instead of SalesPersonRaw in all end-user visuals.
- Use the product hierarchy Category > Subcategory > ProductName for drill-down.
- Use consistent currency formatting for sales measures.
- Apply interactive filtering across overview, product, customer/location, and salesperson pages.
- Hide raw or technical fields from the field list where not needed.

## Data Agent
### goal
Enable a Fabric Data Agent / Q&A experience over the semantic model so users can ask natural-language questions about sales by customer, product hierarchy, and cleaned salesperson.

### scope
The agent should answer questions about:
- Sales amount
- Order quantity
- Distinct orders
- Distinct customers
- Customer and location sales
- Product category and subcategory performance
- Salesperson performance using cleaned salesperson names

### semantic hints
Provide business-friendly synonyms:

#### measures
- Sales Amount: revenue, sales, line sales
- Order Quantity: quantity, units sold, units
- Distinct Orders: orders, order count
- Distinct Customers: customers, customer count
- Average Unit Price: avg price, average selling price
- Discount Amount: discounts, discount value

#### Dim_Product attributes
- Category: product category
- Subcategory: product subcategory
- ProductName: product, item
- ProductModelName: model
- ProductDescription: description

#### Dim_Customer attributes
- CustomerID: customer
- City: city, town
- StateProvince: state, province, region
- PostalCode: zip, zip code, postcode
- AddressType: address category

#### Dim_Sales attributes
- SalesPersonNameClean: salesperson, sales rep, representative, seller

### behavior guidance
- Prefer cleaned salesperson names from Dim_Sales[SalesPersonNameClean] in all responses.
- Interpret product hierarchy questions using Category and Subcategory before ProductName where relevant.
- Interpret customer geography questions using City, StateProvince, and PostalCode from Dim_Customer.
- Use Sales Amount as the default metric when a user asks general sales questions without specifying a measure.
- When users ask for “top products,” rank by Sales Amount unless another metric is specified.
- When users ask for “top salespeople,” rank by Sales Amount using SalesPersonNameClean.
- Use the unique gold customer dimension grain for customer geography questions.

### example questions
- What are total sales by category?
- Show sales by subcategory and product.
- Who are the top 10 salespeople by sales amount?
- What is sales amount by city?
- Which state or province has the highest sales?
- What are the top products by sales amount?
- How many distinct orders do we have?
- How many distinct customers do we have?
- What is the average unit price by category?
- Show discount amount by salesperson.