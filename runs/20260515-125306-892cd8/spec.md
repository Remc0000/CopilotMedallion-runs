# Run Spec 20260515-125243-b2f127

Generated at 2026-05-15 12:52:43Z.

## Inputs
- Workspace: `c959898f-4b8a-4930-8d2a-d16795d606dd`
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
- Target Lakehouse: **e2egpt54**

## Build instructions

You are a data agent, and you use your skills: https://github.com/microsoft/skills-for-fabric/blob/main/agents/FabricDataEngineer.agent.md

In silver I want to add fields for `ingestion_dt` and `source_dt`.

In gold I want to have the following tables:

- **Dim_Customer** — join customer, customer address, address and leave all relevant/meaningful fields.
- **Dim_Product** — join product, productcategory, productdescription, productmodel, productmodelproductdescription. Please notice that in productcategory there is a self join: in the end I have the category (which is the parent category) and a subcategory.
- **Dim_Sales** — Use the salesperson field from salesorderheader, but remove the `adventureworks/` in front of the name and remove the last number at the end of the name. The result is a degenerate dimension with one row per distinct cleaned salesperson.
- **Fact_Sales** — join salesorderdetail and salesorderheader and keep all relevant fields. **MUST also include the cleaned `sales_person` column (same cleaning rule as Dim_Sales)** so that downstream reports can aggregate sales and discounts by salesperson.

Use only one OneLake for bronze, silver and gold; use schemas to separate them. Also use 3 different notebooks (one per layer).

Create a semantic model on top of the gold layer where you can count the distinct salespersons. Also I want to know which salespersons sell the most, but also give the most discounts. Create some reports on top of this!
