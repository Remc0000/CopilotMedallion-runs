# Run Spec 20260515-161728-b073bf

Generated at 2026-05-15 16:17:28Z.

## Inputs
- Workspace: `b0124a7e-c33d-461e-9d02-eb3effdea4e8`
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
- **Fact_Sales** — join salesorderdetail and salesorderheader and keep all relevant fields. **MUST also include the cleaned `salesperson` column (same cleaning rule as Dim_Sales)** so that downstream reports can aggregate sales and discounts by salesperson.

Use only one OneLake for bronze, silver and gold; use schemas to separate them. Also use 3 different notebooks (one per layer).

## Data quality tests

Add a fourth schema called `test` and write a results table `test.test_results` after the gold layer completes. The table must have these columns: `run_id` (string), `test_name` (string), `layer` (string: bronze/silver/gold), `table_name` (string), `status` (string: PASS/FAIL), `actual` (string), `expected` (string), `details` (string), `checked_at` (timestamp).

Run AT LEAST these tests (append a row per test) and **append**, do not overwrite, so history is preserved across runs:

1. **Row counts per layer** — for every selected source table, count rows in bronze, silver and gold (where applicable). Bronze and silver counts must match each other for the same table (a delta of more than 1% should FAIL). Record both numbers in `actual`.
2. **No-null primary keys in gold dims** — `dim_customer.customer_id`, `dim_product.product_id`, and `dim_sales.sales_person` must have zero nulls. PASS if 0, FAIL with the null count otherwise.
3. **Unique primary keys in gold dims** — same three keys must be unique (count equals distinct count). PASS if equal, FAIL with `actual=<row_count>` and `expected=<distinct_count>` otherwise.
4. **Referential integrity** — every `fact_sales.customer_id` exists in `dim_customer.customer_id`; every `fact_sales.product_id` exists in `dim_product.product_id`; every `fact_sales.sales_person` exists in `dim_sales.sales_person`. PASS if zero orphans, FAIL with the orphan count otherwise.
5. **Salesperson cleaning sanity** — none of the `dim_sales.sales_person` values may contain the substring `adventureworks/` (case-insensitive) and none may end with a digit. PASS if zero violations, FAIL with the count otherwise.

The Power BI report described below should also have a small "Data quality" page showing the latest `test_results` rows (filter `run_id = MAX(run_id)`), with a red/green badge per test.

## Semantic model & reports

Create a semantic model on top of the gold layer where you can count the distinct salespersons. Also I want to know which salespersons sell the most, but also give the most discounts. Create some reports on top of this!
