# Run Spec 20260516-073706-963c9f

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
- Use a Fabric medallion pattern with Bronze, Silver, and Gold layers implemented in the target lakehouse.
- Keep all notebook/script configuration in parameter cells:
  - workspace id
  - source lakehouse id/name
  - target lakehouse name
  - source table list
  - optional load mode flags
- Bronze loads must be idempotent:
  - overwrite the full Bronze table for each source object unless the user later edits to incremental logic
  - do not use `saveAsTable`; write by path or lakehouse table APIs appropriate for Fabric
  - preserve source column names and data types as closely as possible
- Add defensive ingestion metadata columns to every Bronze table:
  - `_ingested_at_utc`
  - `_source_workspace_id`
  - `_source_lakehouse_id`
  - `_source_table_name`
  - `_run_id`
- Use alias-prefixed joins in Silver and Gold to prevent ambiguous column references:
  - e.g. `soh.CustomerID = c.CustomerID`, `sod.ProductID = p.ProductID`
- Treat source objects prefixed with `v` as view-like helper datasets, not automatically as authoritative dimensions:
  - `vGetAllCategories`
  - `vProductAndDescription`
  - `vProductModelCatalogDescription`
- Prefer base-table-derived Silver dimensions when the same information can be built from base tables; retain source views for comparison, enrichment, or simplified reporting.
- Defensive REST / metadata operations:
  - handle 404/409/429/5xx with retries where applicable
  - validate source object existence before processing
  - log row counts written per layer
- Silver and Gold writes must also be idempotent:
  - rebuild derived tables with overwrite semantics
  - ensure deterministic surrogate keys only if explicitly introduced
- Standard cross-cutting data quality rules:
  - row-count capture for each table across layers
  - no-null checks on identified primary keys
  - uniqueness checks on identified primary keys
  - referential integrity checks for identified foreign keys
  - accepted-value checks for obvious low-cardinality flags where safe
- Security and privacy:
  - do not surface `PasswordHash` or `PasswordSalt` beyond Bronze unless the user explicitly edits this spec to allow it
  - binary image column `ThumbNailPhoto` should not be exposed in Gold semantic tables unless a reporting need is confirmed
- Naming guidance:
  - Bronze: `br_<source_object>`
  - Silver conformed: `slv_<entity>`
  - Gold star schema: `dim_<entity>`, `fact_<entity>`
- Document all non-obvious modeling choices inline so the user can edit if preferred.

## Medallion build
### Bronze
Load each source object 1:1 from `SalesLT` into Bronze tables with source fidelity plus ingestion metadata.

- `br_address` from `SalesLT/Address`
- `br_customer` from `SalesLT/Customer`
- `br_customeraddress` from `SalesLT/CustomerAddress`
- `br_product` from `SalesLT/Product`
- `br_productcategory` from `SalesLT/ProductCategory`
- `br_productdescription` from `SalesLT/ProductDescription`
- `br_productmodel` from `SalesLT/ProductModel`
- `br_productmodelproductdescription` from `SalesLT/ProductModelProductDescription`
- `br_salesorderdetail` from `SalesLT/SalesOrderDetail`
- `br_salesorderheader` from `SalesLT/SalesOrderHeader`
- `br_vgetallcategories` from `SalesLT/vGetAllCategories`
- `br_vproductanddescription` from `SalesLT/vProductAndDescription`
- `br_vproductmodelcatalogdescription` from `SalesLT/vProductModelCatalogDescription`

Bronze expectations:
- Preserve all columns including `rowguid`, `ModifiedDate`, and view outputs.
- Do not transform business content except basic type normalization if needed.
- Add ingestion metadata columns listed above.

### Silver
Build cleansed, join-ready entities from the actual source columns.

#### 1) `slv_address`
Source: `br_address`

Columns to retain:
- `AddressID`
- `AddressLine1`
- `AddressLine2`
- `City`
- `StateProvince`
- `CountryRegion`
- `PostalCode`
- `rowguid`
- `ModifiedDate`

Transformations:
- trim string columns
- standardize empty strings to null for `AddressLine2`
- keep one row per `AddressID`

Likely key:
- PK: `AddressID`

#### 2) `slv_customer`
Source: `br_customer`

Columns to retain:
- `CustomerID`
- `NameStyle`
- `Title`
- `FirstName`
- `MiddleName`
- `LastName`
- `Suffix`
- `CompanyName`
- `SalesPerson`
- `EmailAddress`
- `Phone`
- `rowguid`
- `ModifiedDate`

Columns to exclude from Silver by default:
- `PasswordHash`
- `PasswordSalt`

Derived columns:
- `CustomerFullName` = concatenate available `Title`, `FirstName`, `MiddleName`, `LastName`, `Suffix`
- `CustomerDisplayName`:
  - use `CompanyName` when populated
  - otherwise use `CustomerFullName`

Likely key:
- PK: `CustomerID`

#### 3) `slv_customeraddress`
Source: `br_customeraddress`

Columns:
- `CustomerID`
- `AddressID`
- `AddressType`
- `rowguid`
- `ModifiedDate`

Purpose:
- bridge table between customer and address
- allows customers to have multiple addresses and address roles

Likely composite key:
- (`CustomerID`, `AddressID`, `AddressType`) or (`CustomerID`, `AddressID`) depending on duplicates discovered

#### 4) `slv_customer_address_enriched`
Source join:
- `slv_customeraddress ca`
- `slv_address a` on `ca.AddressID = a.AddressID`

Columns:
- `CustomerID`
- `AddressID`
- `AddressType`
- `AddressLine1`
- `AddressLine2`
- `City`
- `StateProvince`
- `CountryRegion`
- `PostalCode`

Purpose:
- reusable customer geography bridge for Gold customer and reporting

#### 5) `slv_productcategory`
Source: `br_productcategory`

Columns:
- `ProductCategoryID`
- `ParentProductCategoryID`
- `Name`
- `rowguid`
- `ModifiedDate`

Transformations:
- self-reference retained for hierarchy
- derive optional hierarchy helpers if useful:
  - `IsTopLevelCategory` when `ParentProductCategoryID` is null

Likely key:
- PK: `ProductCategoryID`

#### 6) `slv_productmodel`
Source: `br_productmodel`

Columns:
- `ProductModelID`
- `Name`
- `CatalogDescription`
- `rowguid`
- `ModifiedDate`

Likely key:
- PK: `ProductModelID`

#### 7) `slv_productdescription`
Source: `br_productdescription`

Columns:
- `ProductDescriptionID`
- `Description`
- `rowguid`
- `ModifiedDate`

Likely key:
- PK: `ProductDescriptionID`

#### 8) `slv_productmodelproductdescription`
Source: `br_productmodelproductdescription`

Columns:
- `ProductModelID`
- `ProductDescriptionID`
- `Culture`
- `rowguid`
- `ModifiedDate`

Purpose:
- bridge between product model and localized descriptions

Likely composite key:
- (`ProductModelID`, `ProductDescriptionID`, `Culture`)

#### 9) `slv_product`
Source: `br_product`

Columns:
- `ProductID`
- `Name`
- `ProductNumber`
- `Color`
- `StandardCost`
- `ListPrice`
- `Size`
- `Weight`
- `ProductCategoryID`
- `ProductModelID`
- `SellStartDate`
- `SellEndDate`
- `DiscontinuedDate`
- `ThumbnailPhotoFileName`
- `rowguid`
- `ModifiedDate`

Columns excluded by default:
- `ThumbNailPhoto`

Derived columns:
- `IsDiscontinued` = `DiscontinuedDate is not null`
- `IsCurrentlySellable` = `SellStartDate <= current_date` and (`SellEndDate is null` or `SellEndDate >= current_date`) and `DiscontinuedDate is null`

Likely key:
- PK: `ProductID`

#### 10) `slv_product_category_enriched`
Preferred source route:
- `slv_productcategory` self-join to expose parent/child labels

Join:
- child `pc`
- parent `ppc` on `pc.ParentProductCategoryID = ppc.ProductCategoryID`

Columns:
- `ProductCategoryID`
- `ProductCategoryName` = `pc.Name`
- `ParentProductCategoryID`
- `ParentProductCategoryName` = `ppc.Name`

Alternative route:
- use `br_vgetallcategories` as the authoritative flattened category hierarchy
- user may choose this if the source view is business-approved and more complete

#### 11) `slv_product_description_enriched`
Preferred source route:
- `slv_product p`
- `slv_productmodel pm` on `p.ProductModelID = pm.ProductModelID`
- `slv_productmodelproductdescription pmpd` on `pm.ProductModelID = pmpd.ProductModelID`
- `slv_productdescription pd` on `pmpd.ProductDescriptionID = pd.ProductDescriptionID`

Columns:
- `ProductID`
- `ProductName`
- `ProductModelID`
- `ProductModelName`
- `Culture`
- `ProductDescription`

Alternative route:
- use `br_vproductanddescription`
- user may choose the view-based route if it is simpler and aligns with source business logic

#### 12) `slv_product_model_catalog`
Source:
- `br_vproductmodelcatalogdescription`

Columns:
- `ProductModelID`
- `Name`
- `Summary`
- `Manufacturer`
- `Copyright`
- `ProductURL`
- `WarrantyPeriod`
- `WarrantyDescription`
- `NoOfYears`
- `MaintenanceDescription`
- `Wheel`
- `Saddle`
- `Pedal`
- `BikeFrame`
- `Crankset`
- `PictureAngle`
- `PictureSize`
- `ProductPhotoID`
- `Material`
- `Color`
- `ProductLine`
- `Style`
- `RiderExperience`
- `rowguid`
- `ModifiedDate`

Purpose:
- optional product-model attribute extension dimension
- useful if business wants richer descriptive product reporting

#### 13) `slv_salesorderheader`
Source: `br_salesorderheader`

Columns:
- `SalesOrderID`
- `RevisionNumber`
- `OrderDate`
- `DueDate`
- `ShipDate`
- `Status`
- `OnlineOrderFlag`
- `SalesOrderNumber`
- `PurchaseOrderNumber`
- `AccountNumber`
- `CustomerID`
- `ShipToAddressID`
- `BillToAddressID`
- `ShipMethod`
- `CreditCardApprovalCode`
- `SubTotal`
- `TaxAmt`
- `Freight`
- `TotalDue`
- `Comment`
- `rowguid`
- `ModifiedDate`

Derived columns:
- `OrderYear`, `OrderMonth`, `OrderDateKey`
- `ShipDateKey`, `DueDateKey` where dates are present

Likely key:
- PK: `SalesOrderID`

#### 14) `slv_salesorderdetail`
Source: `br_salesorderdetail`

Columns:
- `SalesOrderID`
- `SalesOrderDetailID`
- `OrderQty`
- `ProductID`
- `UnitPrice`
- `UnitPriceDiscount`
- `LineTotal`
- `rowguid`
- `ModifiedDate`

Derived columns:
- `GrossLineAmount` = `OrderQty * UnitPrice`
- `DiscountAmount` = `OrderQty * UnitPrice * UnitPriceDiscount`
- retain source `LineTotal` as authoritative net line amount

Likely key:
- PK candidate: `SalesOrderDetailID`
- Alternate uniqueness expectation: (`SalesOrderID`, `SalesOrderDetailID`)

#### 15) `slv_salesorder_line`
Primary transactional Silver dataset for analytics

Join:
- `slv_salesorderdetail sod`
- `slv_salesorderheader soh` on `sod.SalesOrderID = soh.SalesOrderID`
- `slv_product p` on `sod.ProductID = p.ProductID`
- optional `slv_product_category_enriched pce` on `p.ProductCategoryID = pce.ProductCategoryID`
- optional `slv_customer c` on `soh.CustomerID = c.CustomerID`

Columns:
- order identifiers: `SalesOrderID`, `SalesOrderDetailID`, `SalesOrderNumber`
- dates: `OrderDate`, `DueDate`, `ShipDate`
- customer keys: `CustomerID`
- address keys: `ShipToAddressID`, `BillToAddressID`
- product keys: `ProductID`, `ProductCategoryID`, `ProductModelID`
- commercial metrics: `OrderQty`, `UnitPrice`, `UnitPriceDiscount`, `LineTotal`, `GrossLineAmount`, `DiscountAmount`
- header metrics: `SubTotal`, `TaxAmt`, `Freight`, `TotalDue`
- operational attrs: `Status`, `OnlineOrderFlag`, `ShipMethod`

Modeling note:
- this table mixes line-level and header-level amounts; in Gold, keep line-grain fact and be careful not to sum header metrics across lines without allocation or distinct-order logic.

### Gold
Proposed star schema based strictly on observed transactional and master tables.

#### Dimensions
##### `dim_customer`
Source:
- `slv_customer`
- optional left join to a preferred address per customer from `slv_customer_address_enriched`

Columns:
- `CustomerID`
- `CustomerDisplayName`
- `CustomerFullName`
- `CompanyName`
- `SalesPerson`
- `EmailAddress`
- `Phone`
- optional address roll-in fields if one default address can be selected:
  - `City`
  - `StateProvince`
  - `CountryRegion`
  - `PostalCode`

Modeling note:
- because `CustomerAddress` is many-to-many/one-to-many with address roles, there are two valid approaches:
  - Option A: keep customer geography outside `dim_customer` and analyze via order ship/bill address dims
  - Option B: derive one preferred customer address (for example `Main Office` or first available)
- Default proposal: Option A for correctness; user may edit to Option B for convenience.

##### `dim_product`
Source:
- `slv_product p`
- left join `slv_product_category_enriched pce` on `p.ProductCategoryID = pce.ProductCategoryID`
- left join `slv_productmodel pm` on `p.ProductModelID = pm.ProductModelID`
- optional left join `slv_product_description_enriched pde` on `p.ProductID = pde.ProductID` and preferred culture
- optional left join `slv_product_model_catalog pmc` on `p.ProductModelID = pmc.ProductModelID`

Columns:
- `ProductID`
- `ProductName` from `p.Name`
- `ProductNumber`
- `Color`
- `Size`
- `Weight`
- `StandardCost`
- `ListPrice`
- `ProductCategoryID`
- `ProductCategoryName`
- `ParentProductCategoryID`
- `ParentProductCategoryName`
- `ProductModelID`
- `ProductModelName`
- optional descriptive fields:
  - `Culture`
  - `ProductDescription`
  - `Manufacturer`
  - `Material`
  - `ProductLine`
  - `Style`
  - `RiderExperience`
- lifecycle fields:
  - `SellStartDate`
  - `SellEndDate`
  - `DiscontinuedDate`
  - `IsDiscontinued`
  - `IsCurrentlySellable`

##### `dim_address`
Source: `slv_address`

Columns:
- `AddressID`
- `AddressLine1`
- `AddressLine2`
- `City`
- `StateProvince`
- `CountryRegion`
- `PostalCode`

Use:
- role-play dimension for both shipping and billing

##### `dim_date`
Generated calendar table

Required date coverage from actual columns:
- `OrderDate`
- `DueDate`
- `ShipDate`
- `SellStartDate`
- `SellEndDate`
- `DiscontinuedDate`
- `ModifiedDate` optional for audit analysis

Columns:
- `Date`
- `DateKey` (YYYYMMDD int)
- `Year`
- `Quarter`
- `MonthNumber`
- `MonthName`
- `YearMonth`
- `WeekOfYear`
- `DayOfMonth`
- `DayName`

#### Facts
##### `fact_sales_order_line`
Grain:
- one row per `SalesOrderDetailID` joined to its order header

Source:
- `slv_salesorder_line`

Keys:
- `SalesOrderDetailID`
- `SalesOrderID`
- `CustomerID`
- `ProductID`
- `OrderDateKey`
- `DueDateKey`
- `ShipDateKey`
- `ShipToAddressID`
- `BillToAddressID`

Measures/columns:
- `OrderQty`
- `UnitPrice`
- `UnitPriceDiscount`
- `GrossLineAmount`
- `DiscountAmount`
- `LineTotal`

Do not include header totals as additive fact columns at line grain unless clearly flagged as non-additive.
- If retained for reference, mark:
  - `OrderSubTotal`
  - `OrderTaxAmt`
  - `OrderFreight`
  - `OrderTotalDue`
- Default recommendation: exclude these from the main line fact to avoid double counting.

##### `fact_sales_order_header`
Grain:
- one row per `SalesOrderID`

Source:
- `slv_salesorderheader`

Keys:
- `SalesOrderID`
- `CustomerID`
- `OrderDateKey`
- `DueDateKey`
- `ShipDateKey`
- `ShipToAddressID`
- `BillToAddressID`

Measures/columns:
- `SubTotal`
- `TaxAmt`
- `Freight`
- `TotalDue`
- `Status`
- `OnlineOrderFlag`

Modeling note:
- Recommended to expose both facts:
  - `fact_sales_order_line` for product analysis
  - `fact_sales_order_header` for order-level monetary totals
- Alternative simpler route:
  - only expose line fact and create distinct-order measures
- Default recommendation: keep both facts for correctness and easier reporting.

### Data quality tests
Apply in Silver and Gold.

#### Ingestion and shape
- Bronze row counts captured for every source object
- Silver row counts captured for every derived table
- Gold row counts captured for every dim/fact

#### Key integrity
- `slv_address.AddressID` not null and unique
- `slv_customer.CustomerID` not null and unique
- `slv_product.ProductID` not null and unique
- `slv_productcategory.ProductCategoryID` not null and unique
- `slv_productmodel.ProductModelID` not null and unique
- `slv_productdescription.ProductDescriptionID` not null and unique
- `slv_salesorderheader.SalesOrderID` not null and unique
- `slv_salesorderdetail.SalesOrderDetailID` not null and unique if source supports it
- `slv_customeraddress` composite key uniqueness check on (`CustomerID`, `AddressID`, `AddressType`)
- `slv_productmodelproductdescription` composite key uniqueness check on (`ProductModelID`, `ProductDescriptionID`, `Culture`)

#### Referential integrity
- `slv_customeraddress.CustomerID` exists in `slv_customer.CustomerID`
- `slv_customeraddress.AddressID` exists in `slv_address.AddressID`
- `slv_product.ProductCategoryID` exists in `slv_productcategory.ProductCategoryID` when not null
- `slv_product.ProductModelID` exists in `slv_productmodel.ProductModelID` when not null
- `slv_productmodelproductdescription.ProductModelID` exists in `slv_productmodel.ProductModelID`
- `slv_productmodelproductdescription.ProductDescriptionID` exists in `slv_productdescription.ProductDescriptionID`
- `slv_salesorderheader.CustomerID` exists in `slv_customer.CustomerID`
- `slv_salesorderheader.ShipToAddressID` exists in `slv_address.AddressID`
- `slv_salesorderheader.BillToAddressID` exists in `slv_address.AddressID`
- `slv_salesorderdetail.SalesOrderID` exists in `slv_salesorderheader.SalesOrderID`
- `slv_salesorderdetail.ProductID` exists in `slv_product.ProductID`

#### Domain and consistency tests
- `OrderQty > 0` in `slv_salesorderdetail`
- `UnitPrice >= 0`
- `UnitPriceDiscount >= 0`
- `LineTotal >= 0` where business rules permit
- `TotalDue >= 0`, `SubTotal >= 0`, `TaxAmt >= 0`, `Freight >= 0`
- `SellEndDate >= SellStartDate` when both present
- `DiscontinuedDate >= SellStartDate` when both present
- `EmailAddress` format sanity check in `slv_customer` if needed
- `OnlineOrderFlag` accepted values boolean only
- `Status` profile distinct values and document meanings if available; do not hardcode business labels unless confirmed

## Semantic model
Use Direct Lake over Gold tables in the target lakehouse.

### Tables
- `dim_customer`
- `dim_product`
- `dim_address`
- `dim_date`
- `fact_sales_order_line`
- `fact_sales_order_header`

### Relationships
Primary recommended relationships:
- `fact_sales_order_line[CustomerID]` -> `dim_customer[CustomerID]` many-to-one
- `fact_sales_order_line[ProductID]` -> `dim_product[ProductID]` many-to-one
- `fact_sales_order_line[ShipToAddressID]` -> `dim_address[AddressID]` many-to-one
- `fact_sales_order_line[BillToAddressID]` -> `dim_address[AddressID]` many-to-one, inactive role-playing relationship
- `fact_sales_order_line[OrderDateKey]` -> `dim_date[DateKey]` many-to-one
- `fact_sales_order_line[DueDateKey]` -> `dim_date[DateKey]` inactive
- `fact_sales_order_line[ShipDateKey]` -> `dim_date[DateKey]` inactive

- `fact_sales_order_header[CustomerID]` -> `dim_customer[CustomerID]` many-to-one
- `fact_sales_order_header[ShipToAddressID]` -> `dim_address[AddressID]` many-to-one
- `fact_sales_order_header[BillToAddressID]` -> `dim_address[AddressID]` many-to-one, inactive
- `fact_sales_order_header[OrderDateKey]` -> `dim_date[DateKey]` many-to-one
- `fact_sales_order_header[DueDateKey]` -> `dim_date[DateKey]` inactive
- `fact_sales_order_header[ShipDateKey]` -> `dim_date[DateKey]` inactive

Modeling note:
- To avoid ambiguity from two facts, do not create direct relationship between facts.
- If report simplicity is preferred, user can edit to publish only one fact table.

### Measures
Create explicit DAX measures only; hide raw numeric columns where appropriate.

Core line-level measures:
- `Sales Amount = SUM(fact_sales_order_line[LineTotal])`
- `Order Quantity = SUM(fact_sales_order_line[OrderQty])`
- `Gross Sales Amount = SUM(fact_sales_order_line[GrossLineAmount])`
- `Discount Amount = SUM(fact_sales_order_line[DiscountAmount])`
- `Average Unit Price = AVERAGE(fact_sales_order_line[UnitPrice])`
- `Distinct Orders = DISTINCTCOUNT(fact_sales_order_line[SalesOrderID])`

Header-level measures:
- `Total Due = SUM(fact_sales_order_header[TotalDue])`
- `Subtotal Amount = SUM(fact_sales_order_header[SubTotal])`
- `Tax Amount = SUM(fact_sales_order_header[TaxAmt])`
- `Freight Amount = SUM(fact_sales_order_header[Freight])`

Derived KPI measures:
- `Average Order Value = DIVIDE([Total Due], DISTINCTCOUNT(fact_sales_order_header[SalesOrderID]))`
- `Average Items per Order = DIVIDE([Order Quantity], [Distinct Orders])`
- `Discount Rate = DIVIDE([Discount Amount], [Gross Sales Amount])`

Time intelligence measures using `dim_date` and `OrderDateKey` active relationship:
- `Sales YTD = TOTALYTD([Sales Amount], dim_date[Date])`
- `Sales PY = CALCULATE([Sales Amount], SAMEPERIODLASTYEAR(dim_date[Date]))`
- `Sales YoY % = DIVIDE([Sales Amount] - [Sales PY], [Sales PY])`

Optional operational measures:
- `Online Orders = CALCULATE(DISTINCTCOUNT(fact_sales_order_header[SalesOrderID]), fact_sales_order_header[OnlineOrderFlag] = TRUE())`
- `Online Order % = DIVIDE([Online Orders], DISTINCTCOUNT(fact_sales_order_header[SalesOrderID]))`

### Semantic model notes
- Mark `dim_date` as the date table.
- Hide technical columns:
  - `rowguid`
  - surrogate/date keys if not needed by users
  - audit columns
- Hide sensitive fields by default:
  - customer contact details may remain visible only if business-approved
  - do not include password-related fields anywhere in the semantic model
- Consider hierarchies:
  - `dim_product`: `ParentProductCategoryName` > `ProductCategoryName` > `ProductName`
  - `dim_address`: `CountryRegion` > `StateProvince` > `City`
  - `dim_date`: `Year` > `Quarter` > `MonthName` > `Date`

## Report
Create a Power BI report over the Direct Lake semantic model.

### Page 1: Sales Overview
Purpose:
- high-level commercial performance over time

Visuals:
- KPI cards:
  - `Sales Amount`
  - `Total Due`
  - `Distinct Orders`
  - `Average Order Value`
- line chart:
  - `Sales Amount` by `dim_date[YearMonth]`
- clustered column chart:
  - `Sales Amount` by `dim_product[ProductCategoryName]`
- donut or stacked column:
  - `Sales Amount` by `fact_sales_order_header[OnlineOrderFlag]`
- matrix:
  - rows: `ParentProductCategoryName`, `ProductCategoryName`, `ProductName`
  - values: `Sales Amount`, `Order Quantity`, `Discount Amount`

Slicers:
- `dim_date[Year]`
- `dim_product[ParentProductCategoryName]`
- `dim_product[ProductCategoryName]`
- `dim_address[CountryRegion]`

### Page 2: Customer and Product Detail
Purpose:
- analyze customers, geography, products, and pricing

Visuals:
- bar chart:
  - top 10 `dim_customer[CustomerDisplayName]` by `Sales Amount`
- map or filled map if geography quality is sufficient:
  - `Sales Amount` by `dim_address[CountryRegion]`, `StateProvince`, `City`
- scatter chart:
  - x: `Average Unit Price`
  - y: `Sales Amount`
  - size: `Order Quantity`
  - details: `dim_product[ProductName]`
- table:
  - `CustomerDisplayName`
  - `CompanyName`
  - `SalesPerson`
  - `Distinct Orders`
  - `Sales Amount`
  - `Average Order Value`
- decomposition tree:
  - analyze `Sales Amount` by category, product, customer, ship method

Slicers:
- `dim_customer[SalesPerson]`
- `dim_product[Color]`
- `dim_product[Size]`
- `fact_sales_order_header[ShipMethod]`

### Page 3: Data Quality
Purpose:
- make data reliability visible to business and engineering users

Visuals:
- cards:
  - source tables ingested count
  - Silver tables built count
  - Gold tables built count
  - latest `_ingested_at_utc`
- matrix of quality checks:
  - entity
  - test name
  - pass/fail
  - failed row count
- bar chart:
  - row counts by layer/table
- table of referential integrity exceptions:
  - orphan `CustomerID`
  - orphan `ProductID`
  - orphan `AddressID`
  - orphan `SalesOrderID`
- table of null/duplicate key findings for:
  - `CustomerID`
  - `ProductID`
  - `SalesOrderID`
  - `SalesOrderDetailID`

Optional Page 4: Order Operations
Purpose:
- operational order lifecycle tracking using actual order dates and statuses

Visuals:
- funnel or bar:
  - order count by `Status`
- line and clustered column:
  - order count and `Total Due` by `OrderDate`
- histogram or column:
  - days from `OrderDate` to `ShipDate`
- matrix:
  - `ShipMethod`, `Status`
  - `Distinct Orders`, `Total Due`, `Freight Amount`

## Data Agent
Create an AISkill grounded on the semantic model.

### Role
You are a sales and product analytics assistant for the `SalesLT` dataset. Help users analyze customers, sales orders, order lines, products, categories, prices, discounts, shipping/billing geography, and order trends using the published semantic model. Prefer explicit measures such as Sales Amount, Total Due, Order Quantity, Discount Amount, and Average Order Value.

### Domain hints
- Main business entities present:
  - customers
  - addresses
  - sales orders
  - sales order lines
  - products
  - product categories
  - product models
- Monetary fields available:
  - `LineTotal`
  - `SubTotal`
  - `TaxAmt`
  - `Freight`
  - `TotalDue`
  - `UnitPrice`
  - `UnitPriceDiscount`
- Time fields available:
  - `OrderDate`
  - `DueDate`
  - `ShipDate`
  - product lifecycle dates including `SellStartDate`, `SellEndDate`, `DiscontinuedDate`
- Geography available through addresses:
  - `City`
  - `StateProvince`
  - `CountryRegion`
  - `PostalCode`

### Starter questions
- What are total sales amount, total due, and order quantity by month?
- Which product categories generate the highest sales amount?
- Who are the top customers by sales amount and by distinct orders?
- How much discount amount are we giving by product and category?
- What is the average order value over time?
- How do online orders compare with non-online orders?
- Which ship methods are associated with the highest total due or freight amount?
- What are sales and order counts by country, state/province, and city?
- Which products appear discontinued, and did they still have recent sales?
- Are there any data quality issues such as orphan order lines or missing customer keys?

### Guardrails
- Only answer using the semantic model tables and measures provided.
- Do not expose or infer anything from `PasswordHash` or `PasswordSalt`.
- Do not invent business definitions for `Status`; if asked, report distinct values unless a confirmed mapping exists.
- For monetary totals:
  - use line-grain measures for product analysis
  - use header-grain measures for order-level totals such as `Total Due`
  - avoid double counting header totals across line rows
- When a question could be answered from either order header or order line facts, state which fact is being used.
- If multiple modeling options exist in this spec, prefer the default recommendation unless the user has edited the model.
- If the question requires unavailable data, say so clearly rather than guessing.
- Flag low-confidence analyses where category flattening or product descriptions depend on optional source views.
