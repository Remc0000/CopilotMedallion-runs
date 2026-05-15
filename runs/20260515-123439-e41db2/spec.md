# Run Spec 20260515-123422-dfdb17

Generated at 2026-05-15 12:34:22Z.

## Inputs
- Workspace: `13ed3eec-f5c4-422c-a5fd-a43ff8c0848e`
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
- **Dim_Sales** — Use the salesperson field from salesorderheader, but remove the `adventureworks/` in front of the name and remove the last number at the end of the name. The result is a degenerate dimension with one row per distinct cleaned salesperson. Materialize the cleaned value as an explicit column named `sales_person`. Build this directly from `SalesOrderHeader.SalesPerson`, and keep the raw source field available during transformation until `sales_person` is created.
- **Fact_Sales** — join salesorderdetail and salesorderheader and keep all relevant fields. Join `SalesOrderDetail` to `SalesOrderHeader` on `SalesOrderID`. **MUST also include the cleaned `sales_person` column (same cleaning rule as Dim_Sales)** so that downstream reports can aggregate sales and discounts by salesperson. The final gold table must contain a physical column named exactly `sales_person` (lowercase with underscore), derived from the `SalesPerson` field in `SalesOrderHeader`, and it must be present in the saved `Fact_Sales` schema alongside the other fact columns. Do not rely on `Dim_Sales` for this field; compute it directly during `Fact_Sales` construction from the joined header row using the same logic as `Dim_Sales`. Use explicit aliases for the joined tables and materialize `sales_person` from the header alias before the final select/write. In the final projection for `Fact_Sales`, explicitly include `sales_person` in addition to the other fact columns; do not drop it during renaming or column pruning. If you standardize column names to snake_case, map `SalesOrderHeader.SalesPerson` to `sales_person` after cleaning and preserve that exact column name in the persisted gold table.

Use only one OneLake for bronze, silver and gold; use schemas to separate them. Also use 3 different notebooks (one per layer).

Create a semantic model on top of the gold layer where you can count the distinct salespersons. Also I want to know which salespersons sell the most, but also give the most discounts. Create some reports on top of this!

## Known constraints (from prior runs)
- `Fact_Sales` must persist the column `sales_person`; downstream validation will fail if the table only contains order, customer, product, and amount fields without this explicit column.
- Keep the cleaned salesperson logic identical between `Dim_Sales.sales_person` and `Fact_Sales.sales_person`.
- In `Fact_Sales`, deriving `sales_person` is not sufficient unless it is also explicitly selected into the final saved schema.