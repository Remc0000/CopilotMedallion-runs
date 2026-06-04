# Run Spec 20260604-120003-8620a7

## Updated specs

### Iteration 1 — 2026-06-04 12:07:16Z — failed layer: bronze (run: 20260604-120039-a99037)
- **Root cause (1-line summary)**: Bronze completed without producing discoverable Delta tables under `Tables/bronze/`, causing Silver generation to fail.
- **Cross-table audit**:
  - Address: yes — discoverability depends on write path and table naming.
  - Customer: yes — discoverability depends on write path and table naming.
  - CustomerAddress: yes — discoverability depends on write path and table naming.
  - Product: yes — discoverability depends on write path and table naming.
  - ProductCategory: yes — discoverability depends on write path and table naming.
  - ProductDescription: yes — discoverability depends on write path and table naming.
  - ProductModel: yes — discoverability depends on write path and table naming.
  - ProductModelProductDescription: yes — discoverability depends on write path and table naming.
  - SalesOrderDetail: yes — discoverability depends on write path and table naming.
  - SalesOrderHeader: yes — discoverability depends on write path and table naming.
- **Fix approach**: GENERALIZE — the failure is systemic and can affect every Bronze source table if paths, naming, or write logic are inconsistent.
- **What was changed**:
  - Tightened Bronze output-path requirements and required exact target folder names for every source table.
  - Added mandatory post-write verification that each Delta path exists and is readable.
  - Added a hard-fail rule when any expected Bronze table is missing or when zero tables are successfully written.

## Inputs
- Workspace: `1a90328c-54a4-4905-9c94-fa8e59b2ceb1`
- Source Lakehouse: **SalesLT** (`47f5fdf7-1902-471b-958f-5a1e9430070e`)
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
- Target Lakehouse: **d**

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
- Use defensive column references and validate required columns before use.
- Use alias-prefixed joins only within the join expression; materialize flat columns immediately after joins.
- Assert column existence before every join, filter, withColumn, groupBy, agg, and Window operation.
- Before any REST response `.get()`, use `if x is None: raise RuntimeError(...)`.
- Do not use `saveAsTable`; write Delta directly to Lakehouse paths.
- All notebooks must begin with parameter cells for workspace, source, target, run_id, and layer settings.
- Use idempotent overwrite patterns with `.mode("overwrite").option("overwriteSchema","true")`.
- Wrap table processing in error-loud try/except blocks that call `_save_error(layer, e)` and re-raise appropriately.
- Process source tables independently per-table where possible.
- Every notebook cell must begin with a short comment block explaining purpose and intent.

All existing Global Spark column-reference rules (A–K) remain in force.

## Bronze

- Land each source table unchanged into `Tables/bronze/<table_name_lower>`.
- Preserve source schema and datatypes.
- Add metadata columns:
  - `_run_id`
  - `_bronze_ingested_at`
  - `_source_table`
  - `_source_lakehouse`
- Write format: Delta.
- Write mode: overwrite with schema overwrite enabled.

Required discoverable output names (must match exactly):
- `SalesLT/Address` → `Tables/bronze/address`
- `SalesLT/Customer` → `Tables/bronze/customer`
- `SalesLT/CustomerAddress` → `Tables/bronze/customeraddress`
- `SalesLT/Product` → `Tables/bronze/product`
- `SalesLT/ProductCategory` → `Tables/bronze/productcategory`
- `SalesLT/ProductDescription` → `Tables/bronze/productdescription`
- `SalesLT/ProductModel` → `Tables/bronze/productmodel`
- `SalesLT/ProductModelProductDescription` → `Tables/bronze/productmodelproductdescription`
- `SalesLT/SalesOrderDetail` → `Tables/bronze/salesorderdetail`
- `SalesLT/SalesOrderHeader` → `Tables/bronze/salesorderheader`

Partitioning:
- `salesorderheader`: partition by year derived from `OrderDate`.
- `salesorderdetail`: partition by year derived from `ModifiedDate`.
- Remaining tables: no partitioning due to expected small dimension size.

Mandatory post-write validation for every table:
- Immediately after write, read the written Delta path back.
- Verify row count > 0 when source row count > 0.
- Verify the Delta path exists under `Tables/bronze/`.
- Record `{table_name, row_count, target_path}` in the Bronze results summary.

Completion requirements:
- Maintain a list of successfully written Bronze tables.
- At notebook end, verify all expected source tables were attempted.
- Raise an error if zero Bronze tables were written.
- Raise an error if any expected Bronze table path listed above is not discoverable under `Tables/bronze/`.
- Print a final JSON summary of all successfully written Bronze tables and their paths.

- Record row counts for all landed tables.
- Preserve binary column `ThumbNailPhoto` without transformation.

## Silver

General transformations:
- Rename all columns to snake_case.
- Standardize timestamps.
- Add:
  - `_silver_ts`
  - `_run_id`
  - `_record_source`
- Remove exact duplicate records.
- Retain business and audit fields needed for Gold.

Per-table deduplication keys:
- address: `address_id`
- customer: `customer_id`
- customer_address: `(customer_id, address_id)`
- product: `product_id`
- product_category: `product_category_id`
- product_description: `product_description_id`
- product_model: `product_model_id`
- product_model_product_description: `(product_model_id, product_description_id, culture)`
- sales_order_header: `sales_order_id`
- sales_order_detail: `sales_order_detail_id`

Silver business enrichment:
- Address:
  - Standardize city and postal_code.
- Customer:
  - Normalize email address casing.
  - Create `salesperson_username` derived from `sales_person`.
  - If value contains `\`, keep username portion only.
  - Remove trailing numeric suffixes for reporting display.
- Product:
  - Create `is_discontinued`.
  - Create `is_currently_sellable`.
- Product Category:
  - Materialize parent-child relationship helper columns.
- Sales Order Header:
  - Derive order_year, order_month, ship_year, ship_month.
- Sales Order Detail:
  - Derive line_sales_amount = order_qty * unit_price.
  - Derive line_discount_amount = order_qty * unit_price * unit_price_discount.
  - Derive line_net_sales_amount = line_sales_amount - line_discount_amount.

OPTIMIZE and VACUUM eligible Silver Delta tables after successful write.

Note:
- User requested ProductModel Name → modelname. The actual ProductModel table contains only `ProductModelID`, `rowguid`, and `ModifiedDate`; no Name column exists. Gold will expose ProductModelID and related description data unless a model name source is later provided.

## Gold

Star schema requested by user.

Dimensions:

1. DimOrderDate
- Source: `sales_order_header.order_date`

2. DimShipDate
- Source: `sales_order_header.ship_date`

3. DimCustomer
- Combine Customer and Address using SalesOrderHeader relationships as specified.

4. DimSalesPerson
- Source: customer.sales_person

5. DimOrder
- Source: SalesOrderHeader

6. DimProduct
- Sources:
  - Product
  - ProductCategory
  - ProductModel
  - ProductModelProductDescription
  - ProductDescription

Fact:
- FactSalesOrder from SalesOrderHeader and SalesOrderDetail on `sales_order_id`.

Regional reporting approach:
- Use City and PostalCode from Address.

## Test

All tests append one result row into:
- `Tables/test/test_results`

Required tests:
1. Row Count Consistency
2. Gold Dimension PK Not Null
3. Gold Dimension PK Uniqueness
4. Referential Integrity
5. Business Rule Sanity Check

## Semantic model

Mode:
- Direct Lake

Tables:
- FactSalesOrder
- DimCustomer
- DimProduct
- DimOrder
- DimOrderDate
- DimShipDate
- DimSalesPerson

Relationships:
- FactSalesOrder → DimCustomer
- FactSalesOrder → DimProduct
- FactSalesOrder → DimOrder
- FactSalesOrder → DimOrderDate
- FactSalesOrder → DimShipDate
- FactSalesOrder → DimSalesPerson

Measures:
- Total Sales = SUM(FactSalesOrder[net_sales_amount])
- Gross Sales = SUM(FactSalesOrder[gross_sales_amount])
- Total Discount Amount = SUM(FactSalesOrder[discount_amount])

## Report

Page 1 — Executive Sales Overview
- KPI cards:
  - Total Sales
  - Average Sales
  - Maximum Sales
  - Total Orders
  - Discount %

Page 2 — Geographic Performance
- Bubble map using City.

Page 3 — Orders & Discounts
- Salesperson ranking and discount analysis.

Page 4 — Product Performance
- Product hierarchy and sales analysis.

Page 5 — Data Quality
- Test pass/fail summary.

## Data Agent

Role:
- Sales Performance Intelligence Agent for the SalesLT sales model.

Domain hints:
- Geographic analysis is based on City and PostalCode from customer-associated addresses.
- Sales measures should prioritize net sales amount unless explicitly requested otherwise.

Guardrails:
- Only answer using data available in the semantic model.
- Do not invent regions, territories, or countries that do not exist in the source data.
- Clearly state when requested information is unavailable.