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

For **all silver tables**, preserve business/source columns from Bronze and add metadata columns without dropping source fields unless there is an explicit rename. In particular, for `SalesOrderHeader`, the original SalesLT salesperson field must be retained and standardized for downstream gold use:
- treat the Bronze/source SalesLT column name as **exactly `SalesPerson`** for `SalesOrderHeader`;
- when reading Bronze `SalesOrderHeader`, inspect the actual dataframe columns before any generic snake_case rename/drop logic; if `SalesPerson` is present, immediately materialize silver column **`salesperson`** from it and carry that column forward through the rest of the transformation;
- do **not** rely on automatic snake_case conversion to create this field; `SalesPerson` must map explicitly to `salesperson`;
- in the silver `SalesOrderHeader` build, explicitly map `SalesPerson` -> `salesperson`;
- `salesperson` is a **required** silver output column for `SalesOrderHeader`, even if many other columns are converted to snake_case;
- do not rename `salesperson` to `sales_person` in silver;
- do not drop `salesperson` during silver standardization, metadata enrichment, final select, deduplication, or column reordering;
- if the Bronze `SalesOrderHeader` dataframe does not contain `SalesPerson`, fail fast before silver write with a clear validation message that includes the Bronze column list; do not proceed to write a silver `SalesOrderHeader` without `salesperson`;
- after silver `SalesOrderHeader` transformation and before write, validate that both `sales_order_id` and `salesperson` are present in the dataframe columns; if either is missing, fail fast and include the final silver column list in the error;
- the silver `SalesOrderHeader` final projection must explicitly enumerate and include `salesperson`; do not use a derived column list that can omit it accidentally;
- the current known-good silver `SalesOrderHeader` shape must include all standard header fields plus metadata **and** `salesperson`; a silver shape that contains `sales_order_id`, `revision_number`, `order_date`, `due_date`, `ship_date`, `status`, `online_order_flag`, `sales_order_number`, `purchase_order_number`, `account_number`, `customer_id`, `ship_to_address_id`, `bill_to_address_id`, `ship_method`, `sub_total`, `tax_amt`, `freight`, `total_due`, `rowguid`, `modified_date`, metadata columns, but no `salesperson`, is invalid and must not be written.

In gold I want to have the following tables:

- **Dim_Customer** — join customer, customer address, address and leave all relevant/meaningful fields.
- **Dim_Product** — join product, productcategory, productdescription, productmodel, productmodelproductdescription. Please notice that in productcategory there is a self join: in the end I have the category (which is the parent category) and a subcategory.
- **Dim_Sales** — build this from `SalesOrderHeader` in silver. The source column is the original salesperson column from SalesLT; in silver it should normally be present as `salesperson` and in gold must be standardized to a snake_case column named `sales_person`. Do not assume `sales_person` already exists in silver. Materialize it in gold from silver using this precedence:
  1. if `salesperson` exists, rename/copy it to `sales_person`;
  2. else if `sales_person` already exists, use it;
  3. else if neither exists but the available silver columns match the known SalesLT header shape and there is no salesperson field at all, fail fast with a clear validation message stating that silver `SalesOrderHeader` was built incorrectly because the original salesperson source column was dropped; include the available columns in the error.
  Clean `sales_person` by removing the `adventureworks/` prefix and removing the trailing numeric suffix at the end of the name. The result is a degenerate dimension with one row per distinct cleaned salesperson. If the source salesperson column is absent in silver, do not silently continue or infer from another field; fail fast as above.
- **Fact_Sales** — join `SalesOrderDetail` and `SalesOrderHeader` and keep all relevant fields. Join on `sales_order_id`. This table **must also include the cleaned `sales_person` column (same cleaning rule as Dim_Sales)** so that downstream reports can aggregate sales and discounts by salesperson.

  Build `Fact_Sales` in these explicit steps:
  1. Create a header-prep dataset from silver `SalesOrderHeader`.
  2. In header-prep, validate `sales_order_id` exists.
  3. In header-prep, materialize exactly one canonical salesperson column named `sales_person` using this precedence:
     - if `salesperson` exists, rename/copy it to `sales_person`;
     - else if `sales_person` already exists, use it;
     - else fail fast with a clear validation message that silver `SalesOrderHeader` is missing the original SalesLT salesperson field because it was not preserved from source; include the available `SalesOrderHeader` columns.
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
- `SalesOrderHeader` silver did not contain a `sales_person` column; available columns were snake_case fields plus ingestion metadata. The root issue was that the original source column `SalesPerson` was dropped before silver write. In the next build, preserve it in silver as **`salesperson`**.
- A prior silver `SalesOrderHeader` column list was `['sales_order_id', 'revision_number', 'order_date', 'due_date', 'ship_date', 'status', 'online_order_flag', 'sales_order_number', 'purchase_order_number', 'account_number', 'customer_id', 'ship_to_address_id', 'bill_to_address_id', 'ship_to_address_id', 'bill_to_address_id', 'ship_method', 'sub_total', 'tax_amt', 'freight', 'total_due', 'rowguid', 'modified_date', 'ingestion_timestamp', 'source_path', 'batch_id', 'ingestion_date', 'ingestion_dt', 'source_dt', '_silver_ts']` with no salesperson field. This exact outcome is invalid; silver must include `salesperson` before any gold notebook runs.
- The specific failure to avoid is: `ValueError: silver SalesOrderHeader validation failed; missing columns ['salesperson']`. Prevent this by explicitly creating `salesperson` from Bronze `SalesPerson` before any standardization step that can remove or overlook it.
- Validate required columns before gold transformations, especially on `SalesOrderHeader`: `sales_order_id` and the salesperson source column (`salesperson` preferred, or explicitly created `sales_person`).
- A prior gold run produced `Fact_Sales` without `sales_person` even though the salesperson summary depends on it. The final fact projection must explicitly include `h.sales_person AS sales_person`; do not rely on inherited columns or `select *`.