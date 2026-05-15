# Run Spec 20260515-173816-082107

Generated at 2026-05-15 17:38:16Z.

## Inputs
- Workspace: `6ed3eac0-555b-4eff-b6ce-8767ea0dd530`
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
Additionally apply these Power BI authoring skills:
- https://github.com/microsoft/skills-for-fabric/tree/main/skills/powerbi-authoring-cli (semantic models + reports via Fabric CLI)
- https://github.com/microsoft/skills-for-fabric/tree/main/skills/powerbi-consumption-cli (DAX query validation)
- https://github.com/RuiRomano/powerbi-agentic-plugins/tree/main/plugins/powerbi/skills/powerbi-semantic-model-authoring (Direct Lake guidelines, TMDL/TMSL syntax, DAX modeling best practices, star-schema rules)
- https://github.com/RuiRomano/powerbi-agentic-plugins/tree/main/plugins/powerbi/skills/powerbi-report-authoring (PBIR report format)

In silver I want to add fields for `ingestion_dt` and `source_dt`.

In gold I want to have the following tables:

- **Dim_Customer** — join customer, customer address, address and leave all relevant/meaningful fields.
- **Dim_Product** — join product, productcategory, productdescription, productmodel, productmodelproductdescription. Please notice that in productcategory there is a self join: in the end I have the category (which is the parent category) and a subcategory.
- **Dim_Sales** — Use the salesperson field from salesorderheader, but remove the `adventureworks/` in front of the name and remove the last number at the end of the name. The result is a degenerate dimension with one row per distinct cleaned salesperson.
- **Fact_Sales** — join salesorderdetail and salesorderheader and keep all relevant fields. **MUST also include the cleaned `sales_person` column (same cleaning rule as Dim_Sales)** so that downstream reports can aggregate sales and discounts by salesperson.

Use only one OneLake for bronze, silver and gold; use schemas to separate them. Also use 4 different notebooks (one per layer + one for reporting).

## Data quality tests

Add a fourth schema called `test` and write a results table `test.test_results` after the gold layer completes. The table must have these columns: `run_id` (string), `test_name` (string), `layer` (string: bronze/silver/gold), `table_name` (string), `status` (string: PASS/FAIL), `actual` (string), `expected` (string), `details` (string), `checked_at` (timestamp).

Run AT LEAST these tests (append rows; do not overwrite):
1. **Row counts per layer** — bronze and silver counts must match within 1%.
2. **No-null primary keys in gold dims** — `dim_customer.customer_id`, `dim_product.product_id`, `dim_sales.sales_person`.
3. **Unique primary keys in gold dims**.
4. **Referential integrity** — every `fact_sales` FK exists in its dim.
5. **Salesperson cleaning sanity** — no `adventureworks/` substring; no trailing digit.

## Semantic model, Report, and Data Agent (the `reporting` notebook)

Create three Fabric items via REST API, all bound to the gold layer of this lakehouse:

1. **Direct Lake Semantic Model** named `{target_lakehouse_name}_sm` — with tables for dim_customer, dim_product, dim_sales, fact_sales; relationships fact→dim on customer_id, product_id, sales_person; measures: `Total Sales`, `Total Discount Amount`, `Distinct Salespersons`, `Sales Count`.
2. **Power BI Report** named `{target_lakehouse_name}_rpt` — page 1 'Sales overview' (cards + bar charts for top salespersons by sales and by discounts); page 2 'Data quality' (table of latest test_results, color-coded by status).
3. **Data Agent (AISkill)** named `{target_lakehouse_name}_agent` — over the gold tables, with these starter questions:
   - Who are the top 5 salespersons by total sales?
   - Which salespersons give the largest discounts?
   - How many distinct salespersons are there?
   - Show me the latest data quality results.

If the AISkill REST endpoint is not available in this tenant (preview feature), the reporting notebook should log a clear message and continue without failing.
