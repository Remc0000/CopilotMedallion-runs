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
- **Dim_Sales** — build this from `SalesOrderHeader` in silver. The source column is the original salesperson column from SalesLT; in silver it may be present as `salesperson` (expected) and must be standardized to a snake_case column named `sales_person`. Do not assume `sales_person` already exists unless you explicitly create/rename it. Clean `sales_person` by removing the `adventureworks/` prefix and removing the trailing numeric suffix at the end of the name. The result is a degenerate dimension with one row per distinct cleaned salesperson. If the source salesperson column is absent in silver, fail fast with a clear validation message listing the available columns.
- **Fact_Sales** — join `SalesOrderDetail` and `SalesOrderHeader` and keep all relevant fields. Join on `sales_order_id`. This table **must also include the cleaned `sales_person` column (same cleaning rule as Dim_Sales)** so that downstream reports can aggregate sales and discounts by salesperson. Materialize `sales_person` from the header dataset before or during the join: if the silver header has `salesperson`, rename/derive it as `sales_person` first, then apply the cleaning rule; do not reference `sales_person` on `SalesOrderHeader` unless that column has been explicitly created. Use aliases when joining detail and header to avoid ambiguous references.

Use only one OneLake for bronze, silver and gold; use schemas to separate them. Also use 3 different notebooks (one per layer).

Create a semantic model on top of the gold layer where you can count the distinct salespersons. Also I want to know which salespersons sell the most, but also give the most discounts. Create some reports on top of this!

## Known constraints (from prior runs)
- `SalesOrderHeader` silver did not contain a `sales_person` column; available columns were snake_case fields plus ingestion metadata. Standardize the raw salesperson field to `sales_person` explicitly before gold logic depends on it.
- Validate required columns before gold transformations, especially on `SalesOrderHeader`: `sales_order_id` and the salesperson source column (`salesperson` or explicitly created `sales_person`).