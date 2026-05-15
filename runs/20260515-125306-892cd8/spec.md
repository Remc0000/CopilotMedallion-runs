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
- **Fact_Sales** — join `SalesOrderDetail` and `SalesOrderHeader` and keep all relevant fields. Join on `sales_order_id`. This table **must also include the cleaned `sales_person` column (same cleaning rule as Dim_Sales)** so that downstream reports can aggregate sales and discounts by salesperson.

  Build `Fact_Sales` in these explicit steps:
  1. Create a header-prep dataset from silver `SalesOrderHeader`.
  2. In header-prep, validate `sales_order_id` exists.
  3. In header-prep, materialize exactly one canonical salesperson column named `sales_person` using this precedence:
     - if `salesperson` exists, rename/copy it to `sales_person`;
     - else if `sales_person` already exists, use it;
     - else fail fast with a clear validation message listing available `SalesOrderHeader` columns.
  4. Clean the canonical `sales_person` in header-prep by removing the `adventureworks/` prefix and removing any trailing numeric suffix at the end of the value.
  5. Keep `sales_person` in header-prep all the way through the final fact projection; it is a required output column and must not be dropped during renaming or column selection.

  When joining detail and header:
  - alias detail as `d` and header-prep as `h`;
  - join on `d.sales_order_id = h.sales_order_id`;
  - do not use `select *` after the join;
  - enumerate the final `Fact_Sales` columns explicitly;
  - explicitly project `h.sales_person AS sales_person` into the final schema, even if other header columns are renamed with `header_` prefixes;
  - if you rename header columns with `header_` prefixes, do **not** rename `sales_person` to `header_sales_person`; keep the final output column name exactly `sales_person`.

  The final `Fact_Sales` output must contain at least:
  - `sales_order_id`
  - `sales_order_detail_id`
  - `customer_key`
  - `product_key`
  - `sales_person`
  - relevant header attributes such as `order_date`, `due_date`, `ship_date`, `status`, `online_order_flag`, `sales_order_number`, `purchase_order_number`, `account_number`, `ship_method`, `bill_to_address_id`, `ship_to_address_id`
  - relevant measures such as `header_sub_total`, `header_tax_amt`, `header_freight`, `header_total_due`, `order_qty`, `unit_price`, `unit_price_discount`, `line_total`
  - lineage/metadata fields such as `ingestion_dt`, `source_dt`, and any derived date key if created

  After building `Fact_Sales`, assert that both `sales_order_id` and `sales_person` are present in the final dataframe columns before writing gold. If either is missing, fail fast with a clear validation message that includes the final projected column list.

  Keep all relevant header/detail measures and attributes, and retain the cleaned salesperson field for downstream salesperson reporting.

Use only one OneLake for bronze, silver and gold; use schemas to separate them. Also use 3 different notebooks (one per layer).

Create a semantic model on top of the gold layer where you can count the distinct salespersons. Also I want to know which salespersons sell the most, but also give the most discounts. Create some reports on top of this!

## Known constraints (from prior runs)
- `SalesOrderHeader` silver did not contain a `sales_person` column; available columns were snake_case fields plus ingestion metadata. Standardize the raw salesperson field to `sales_person` explicitly before gold logic depends on it.
- Validate required columns before gold transformations, especially on `SalesOrderHeader`: `sales_order_id` and the salesperson source column (`salesperson` or explicitly created `sales_person`).
- A prior gold run produced `Fact_Sales` without `sales_person` even though the salesperson summary depends on it. The final fact projection must explicitly include `h.sales_person AS sales_person`; do not rely on inherited columns or `select *`.