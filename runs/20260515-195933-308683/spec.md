# Run Spec 20260515-195921-dffbbe

## Inputs
- Workspace: `4441f767-16e2-443b-870b-7448bcb4ca02`
- Source Lakehouse: **SalesLT** (`efe41f78-82b7-47ee-9780-2d78372bfdf3`)
- Tables to ingest into Bronze:
- `SalesLT/Address`
- `SalesLT/Customer`
- `SalesLT/Product`
- `SalesLT/ProductCategory`
- `SalesLT/CustomerAddress`
- `SalesLT/ProductDescription`
- `SalesLT/ProductModel`
- `SalesLT/ProductModelProductDescription`
- `SalesLT/SalesOrderDetail`
- `SalesLT/SalesOrderHeader`
- Target Lakehouse: **e2egpt54v2**

## Medallion build

You are a data agent following the FabricDataEngineer agent (https://github.com/microsoft/skills-for-fabric/blob/main/agents/FabricDataEngineer.agent.md) and the e2e-medallion-architecture skill.

Build 3 Spark notebooks (one per layer) — bronze, silver, gold — in a single lakehouse with schemas as the layer separator.

- **Bronze**: 1:1 ingestion of the selected source tables into the `bronze` schema, adding `ingestion_timestamp`, `source_path`, `batch_id`, `ingestion_date` metadata columns. Mode: overwrite + overwriteSchema=true. Partitioned by `ingestion_date`.
- **Silver**: dedupe, snake_case rename, drop fully-null columns, trim strings, add `ingestion_dt` and `source_dt` audit columns, `_silver_ts` timestamp. Mode: overwrite. After write run OPTIMIZE. For all downstream gold logic, use the snake_case silver column names exactly as produced from source-system names. In particular, `SalesOrderHeader.SalesPerson` becomes `sales_person`; if the source table does not contain a `SalesPerson` field, do not invent it.
- **Gold** (AdventureWorksLT shape):
  - **General rule for all gold joins/projections** — when joining silver tables, never use `select("*")`, `a.*`, `b.*`, or any projection that carries forward all columns from more than one source. Gold DataFrames must be built with an explicit final select list only, with unique output column names. Before every gold write, validate that the projected schema has no duplicate column names (case-insensitive), especially shared business/audit columns such as `rowguid`, `modified_date`, `ingestion_dt`, `source_dt`, `_silver_ts`, `ingestion_timestamp`, `ingestion_date`, `batch_id`, and `source_path`. If the same logical field from multiple inputs is needed, keep exactly one canonical output column name and rename the others with table-specific prefixes. Never reference a shared column name without an alias qualifier inside a joined gold DataFrame. Also, never project a column that is not present in the source silver schema: first validate the source column exists, and if it does not, either omit it from the gold output or log the condition to `test.test_results` and continue/skip as specified for that table.
  - **Dim_Customer** — join customer + customeraddress + address; keep all relevant fields. Use explicit DataFrame aliases for every joined table (`c`, `ca`, `a`) and qualify every selected/joined column with its alias. Join keys must be `c.customer_id = ca.customer_id` and `ca.address_id = a.address_id`. In the final projection, do not reference bare `address_id`; materialize it explicitly as `ca.address_id AS address_id` for the dimension key/bridge reference, and if the physical address table key is also needed expose it only as a separately named column such as `a.address_id AS source_address_id`. This join must not contain any unqualified `address_id` reference. Also, because `customer`, `customer_address`, and `address` share several audit/system columns, the final `dim_customer` select must explicitly choose only one version of each duplicated name. At minimum, do not project duplicate output names for `rowguid`, `modified_date`, `ingestion_dt`, `source_dt`, `_silver_ts`, `ingestion_timestamp`, `ingestion_date`, `batch_id`, or `source_path`; if retaining these from multiple sources, rename them distinctly (for example `customer_rowguid`, `address_rowguid`, `customer_modified_date`, `address_modified_date`, etc.). Specifically for `rowguid`, do not select bare `rowguid` anywhere in the joined `dim_customer` build. If a canonical `rowguid` is required, materialize exactly one qualified source such as `c.rowguid AS rowguid`; any additional `rowguid` values must be renamed (for example `ca.rowguid AS customer_address_rowguid`, `a.rowguid AS address_rowguid`). Apply the same qualified-and-renamed rule to `modified_date` and all shared audit/metadata columns.
  - **Dim_Product** — join product + productcategory (self-join: subcategory → parent category) + productdescription + productmodel + productmodelproductdescription; expose `category` and `subcategory`. Use aliases consistently on self-joins and shared key names to avoid ambiguous references. Build with aliases `p` = `silver.product`, `pc` = subcategory `silver.product_category`, `parent` = parent category `silver.product_category`, `pm` = `silver.product_model`, `pmpd` = `silver.product_model_product_description`, `pd` = `silver.product_description`. Join keys must be `p.product_category_id = pc.product_category_id`, `pc.parent_product_category_id = parent.product_category_id`, `p.product_model_id = pm.product_model_id`, `pm.product_model_id = pmpd.product_model_id`, and `pmpd.product_description_id = pd.product_description_id`. For category labeling, use `parent.name AS category` and `CASE WHEN parent.name IS NOT NULL THEN pc.name ELSE CAST(NULL AS STRING) END AS subcategory` so parent rows become categories and child rows become subcategories. Critically, do not reference `p.discontinued_date` unless that exact column exists in `silver.product`; the current SalesLT product schema does not include `discontinued_date`, so `dim_product` must omit that field rather than selecting a non-existent column. Before the `dim_product` select, validate the available columns in `silver.product`; only project product attributes that physically exist there. Use the actual snake_case product columns available in silver, including `product_id`, `name`, `product_number`, `color`, `standard_cost`, `list_price`, `size`, `weight`, `product_category_id`, `product_model_id`, `sell_start_date`, `sell_end_date`, `thumb_nail_photo`, `thumbnail_photo_file_name`, `rowguid`, `modified_date`, `source_dt`, `ingestion_dt`, `_silver_ts`, and bronze metadata columns if needed. If a desired attribute is absent from `silver.product` (specifically `discontinued_date`), omit it from `dim_product` and write a WARN/ERROR-style row to `test.test_results` noting that the column was not present in source schema instead of raising an exception. As with all gold outputs, do not project duplicate shared audit/system column names from multiple silver inputs; explicitly select and rename any retained duplicates so the final schema is unique.
  - **Dim_Sales** — build from `silver.sales_order_header` only after first verifying the cleaned silver column `sales_person` exists in that DataFrame. Apply salesperson cleaning only to `soh.sales_person` (not to any guessed variant such as `salesperson` or `sales_person_name`): strip leading `adventureworks/`, `adventure-works/`, or `adventure works/` case-insensitively, then strip trailing digits, then trim. If `sales_person` is absent in `silver.sales_order_header`, skip creation of `dim_sales` and record a FAIL/ERROR row in `test.test_results` explaining that the source schema does not provide a salesperson column; do not call `select` or any transform against a non-existent `sales_person` column.
  - **Fact_Sales** — join salesorderdetail + salesorderheader, keep all relevant fields, and include the cleaned `sales_person` column only if `silver.sales_order_header.sales_person` exists. Use explicit aliases (`sod`, `soh`) and qualify all shared columns. The salesperson cleaning logic here must reuse the same expression as **Dim_Sales** and must be applied specifically to `soh.sales_person`. If `sales_person` is absent, still build `fact_sales` without that column and log the missing-column condition to `test.test_results` instead of failing the notebook. Do not project both `sod.*` and `soh.*`; explicitly select the final fact columns and ensure any shared names (including audit/system columns) appear at most once in the output schema.

### Data quality tests (in the gold notebook)

Add a `test` schema with table `test_results` (run_id, test_name, layer, table_name, status, actual, expected, details, checked_at). APPEND rows so history is preserved. Run at minimum:

1. Row counts per layer (bronze ≈ silver within 1%).
2. No-null primary keys in gold dims.
3. Unique primary keys in gold dims.
4. Referential integrity (every fact_sales FK exists in its dim).
5. Salesperson cleaning sanity (run this only when a gold `sales_person` column exists; otherwise write a FAIL/ERROR result with details `sales_person column missing in silver.sales_order_header`, rather than raising an exception).
6. Gold schema uniqueness check before each gold write: verify no duplicate output column names exist in the DataFrame schema (case-insensitive). If duplicates are found, do not attempt the write; instead log a FAIL/ERROR row in `test.test_results` listing the duplicate columns.
7. For every multi-table gold join, run a pre-write ambiguity guard on the selected output columns: no unqualified shared source columns may be referenced in the final projection. In particular for `dim_customer`, verify the final select contains no bare references to `rowguid`, `modified_date`, `ingestion_dt`, `source_dt`, `_silver_ts`, `ingestion_timestamp`, `ingestion_date`, `batch_id`, `source_path`, or `address_id`.
8. Add a gold source-schema existence check for every optional/non-universal column referenced by name before projection. At minimum, for `dim_product`, verify whether `silver.product` contains `discontinued_date`; if not, log the condition to `test.test_results` and build `dim_product` without that column rather than failing with `UNRESOLVED_COLUMN`.

## Semantic model

Apply: https://github.com/RuiRomano/powerbi-agentic-plugins/tree/main/plugins/powerbi/skills/powerbi-semantic-model-authoring (Direct Lake guidelines, modeling-guidelines, TMSL/TMDL syntax).

Create a **Direct Lake** semantic model named `{target_lakehouse_name}_sm` on the gold tables, with:
- Tables: `dim_customer`, `dim_product`, `dim_sales`, `fact_sales`.
- Relationships:
  - `fact_sales[customer_id]` → `dim_customer[customer_id]` (many-to-one)
  - `fact_sales[product_id]` → `dim_product[product_id]` (many-to-one)
  - `fact_sales[sales_person]` → `dim_sales[sales_person]` (many-to-one)
- Explicit measures (PascalCase, with format strings):
  - `[Total Sales] = SUM(fact_sales[line_total])` — currency
  - `[Total Discount Amount] = SUMX(fact_sales, fact_sales[line_total] * fact_sales[unit_price_discount])` — currency
  - `[Discount %] = DIVIDE([Total Discount Amount], [Total Sales])` — percent
  - `[Distinct Salespersons] = DISTINCTCOUNT(dim_sales[sales_person])`
  - `[Sales Count] = COUNTROWS(fact_sales)`
- Mark `fact_sales[order_date]` as a date column so time intelligence works without a separate date dim.
- Hide all FK columns from the report view.

## Report

Apply: https://github.com/RuiRomano/powerbi-agentic-plugins/tree/main/plugins/powerbi/skills/powerbi-report-authoring (PBIR format).

Create a Power BI report named `{target_lakehouse_name}_rpt` bound to the semantic model above. Pages:

1. **Sales overview** — card visuals for `[Total Sales]`, `[Distinct Salespersons]`, `[Sales Count]`; bar chart "Top salespersons by sales" (axis = `dim_sales[sales_person]`, value = `[Total Sales]`, sort desc, top 10); bar chart "Top salespersons by discount given" (axis = `dim_sales[sales_person]`, value = `[Total Discount Amount]`, sort desc, top 10); line chart sales by month.
2. **Product mix** — matrix of `[Total Sales]` by `dim_product[category]` rows × `dim_product[subcategory]` columns; bar chart top 10 products by `[Total Sales]`.
3. **Data quality** — table visual showing the latest `test_results` (filter to MAX(run_id)), columns: layer, table_name, test_name, status, actual, expected; conditional formatting on `status` (green=PASS, red=FAIL, amber=ERROR).

## Data Agent

Create a **Data Agent (AISkill)** named `{target_lakehouse_name}_agent`, **grounded on the semantic model** created above (not on the raw tables), so it answers through curated measures and relationships.

**System instructions for the agent:**

> You are a sales analytics assistant for the AdventureWorksLT business. Answer questions about sales performance, salespeople, products, customers, and data quality. Always use the curated semantic-model measures (`Total Sales`, `Total Discount Amount`, `Discount %`, `Distinct Salespersons`, `Sales Count`) and the cleaned `dim_sales[sales_person]` dimension. NEVER aggregate columns directly from `fact_sales` when an equivalent measure exists.
>
> When asked "who", return the cleaned salesperson name (no `adventureworks/` prefix, no trailing digit). When asked "how much", format numbers as currency with two decimals. When asked about data quality, query the latest `test_results` (filter to MAX(run_id)) and summarise PASS/FAIL counts per layer and table. If the question cannot be answered from this model, say so clearly and suggest a follow-up. Be concise (3-5 sentences) unless the user asks for a deep dive.

**Domain context to inject:**

- "Salesperson names are stored cleaned in `dim_sales[sales_person]`; the raw value in `salesorderheader[sales_person]` had an `adventureworks/` prefix and a trailing employee-number digit, both stripped."
- "Discount math: line discount amount = `fact_sales[line_total] * fact_sales[unit_price_discount]`. Use `[Total Discount Amount]` rather than inline math."
- "There is no separate date dimension — use `fact_sales[order_date]` for time aggregations."
- "Data quality lives in the `test` schema, table `test_results`, one row per (run_id, test_name)."

**Starter / example questions:**

- "Who are the top 5 salespersons by total sales?"
- "Which salespersons give the largest discounts (by total discount amount and as % of their sales)?"
- "How many distinct salespersons are there?"
- "What is the average order value by salesperson?"
- "Which products generate the most revenue?"
- "Which product category drives the most discount give-away?"
- "Show me sales trend by month over the last year."
- "What is the latest data quality status? Are any tests failing?"

**Guardrails:**

- Refuse questions outside this dataset (no external lookups, no other businesses).
- Do not invent column names that aren't in the semantic model — if a metric isn't exposed, propose adding it and stop.

If the AISkill REST endpoint is not yet GA in the tenant, the reporting notebook will print these full instructions and starter questions so they can be pasted when creating the Data Agent manually in the Fabric portal.

## Known constraints (from prior runs)
- In gold-layer joins, any column name present in more than one input table must always be referenced with an alias prefix; never use bare shared names like `address_id`, `rowguid`, or `modified_date`.
- For `dim_customer`, `customer_address.address_id` is the canonical joined address key to project as `address_id`. If an additional address-table identifier is retained, it must be renamed to avoid collisions.
- In `dim_customer`, a bare `rowguid` reference is invalid because `customer`, `customer_address`, and `address` each provide `rowguid`. Select exactly one qualified source as canonical, and rename any others with source-specific prefixes.
- Before applying any gold-layer transform to salesperson data, validate that `silver.sales_order_header` actually contains the snake_case column `sales_person`; if not, branch gracefully and log the issue to `test.test_results` instead of raising `UNRESOLVED_COLUMN`.
- The gold write path must never receive a DataFrame with duplicate column names. This was previously hit in `dim_customer` from carrying overlapping columns across `customer`, `customer_address`, and `address` (including `rowguid`, `modified_date`, silver audit columns, and bronze metadata columns). Explicitly project a unique final schema before writing Delta.
- `silver.product` in this source does not provide `discontinued_date`. Any `dim_product` logic must validate source columns first and must not select `p.discontinued_date` unless that column is confirmed to exist.