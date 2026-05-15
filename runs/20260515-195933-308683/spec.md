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
  - **General rule for all gold joins/projections** — when joining silver tables, never use `select("*")`, `a.*`, `b.*`, or any projection that carries forward all columns from more than one source. Gold DataFrames must be built with an explicit final select list only, with unique output column names. Before every gold write, validate that the projected schema has no duplicate column names (case-insensitive), especially shared business/audit columns such as `rowguid`, `modified_date`, `ingestion_dt`, `source_dt`, `_silver_ts`, `ingestion_timestamp`, `ingestion_date`, `batch_id`, and `source_path`. If the same logical field from multiple inputs is needed, keep exactly one canonical output column name and rename the others with table-specific prefixes. Never reference a shared column name without an alias qualifier inside a joined gold DataFrame. Also, never project a column that is not present in the source silver schema: first validate the source column exists, and if it does not, either omit it from the gold output or log the condition to `test.test_results` and continue/skip as specified for that table. Every gold table build must end in one of two states only: (1) the Delta table is successfully created/written at `Tables/gold/<table_name>`, or (2) the table is intentionally skipped and a status row is appended to `test.test_results` naming the table, reason, and skip/fail status. Downstream semantic-model/reporting logic must not assume the physical Delta path exists unless the table was actually created.
  - **Dim_Customer** — join customer + customeraddress + address; keep all relevant fields. Use explicit DataFrame aliases for every joined table (`c`, `ca`, `a`) and qualify every selected/joined column with its alias. Join keys must be `c.customer_id = ca.customer_id` and `ca.address_id = a.address_id`. In the final projection, do not reference bare `address_id`; materialize it explicitly as `ca.address_id AS address_id` for the dimension key/bridge reference, and if the physical address table key is also needed expose it only as a separately named column such as `a.address_id AS source_address_id`. This join must not contain any unqualified `address_id` reference. Also, because `customer`, `customer_address`, and `address` share several audit/system columns, the final `dim_customer` select must explicitly choose only one version of each duplicated name. At minimum, do not project duplicate output names for `rowguid`, `modified_date`, `ingestion_dt`, `source_dt`, `_silver_ts`, `ingestion_timestamp`, `ingestion_date`, `batch_id`, or `source_path`; if retaining these from multiple sources, rename them distinctly (for example `customer_rowguid`, `address_rowguid`, `customer_modified_date`, `address_modified_date`, etc.). Specifically for `rowguid`, do not select bare `rowguid` anywhere in the joined `dim_customer` build. If a canonical `rowguid` is required, materialize exactly one qualified source such as `c.rowguid AS rowguid`; any additional `rowguid` values must be renamed (for example `ca.rowguid AS customer_address_rowguid`, `a.rowguid AS address_rowguid`). Apply the same qualified-and-renamed rule to `modified_date` and all shared audit/metadata columns.
  - **Dim_Product** — join product + productcategory (self-join: subcategory → parent category) + productdescription + productmodel + productmodelproductdescription; expose `category` and `subcategory`. Use aliases consistently on self-joins and shared key names to avoid ambiguous references. Build with aliases `p` = `silver.product`, `pc` = subcategory `silver.product_category`, `parent` = parent category `silver.product_category`, `pm` = `silver.product_model`, `pmpd` = `silver.product_model_product_description`, `pd` = `silver.product_description`. Join keys must be `p.product_category_id = pc.product_category_id`, `pc.parent_product_category_id = parent.product_category_id`, `p.product_model_id = pm.product_model_id`, `pm.product_model_id = pmpd.product_model_id`, and `pmpd.product_description_id = pd.product_description_id`. For category labeling, use `parent.name AS category` and `CASE WHEN parent.name IS NOT NULL THEN pc.name ELSE CAST(NULL AS STRING) END AS subcategory` so parent rows become categories and child rows become subcategories. Critically, do not reference `p.discontinued_date` unless that exact column exists in `silver.product`; the current SalesLT product schema does not include `discontinued_date`, so `dim_product` must omit that field rather than selecting a non-existent column. Before the `dim_product` select, validate the available columns in `silver.product`; only project product attributes that physically exist there. Use the actual snake_case product columns available in silver, including `product_id`, `name`, `product_number`, `color`, `standard_cost`, `list_price`, `size`, `weight`, `product_category_id`, `product_model_id`, `sell_start_date`, `sell_end_date`, `thumb_nail_photo`, `thumbnail_photo_file_name`, `rowguid`, `modified_date`, `source_dt`, `ingestion_dt`, `_silver_ts`, and bronze metadata columns if needed. If a desired attribute is absent from `silver.product` (specifically `discontinued_date`), omit it from `dim_product` and write a WARN/ERROR-style row to `test.test_results` noting that the column was not present in source schema instead of raising an exception. As with all gold outputs, do not project duplicate shared audit/system column names from multiple silver inputs; explicitly select and rename any retained duplicates so the final schema is unique.
  - **Dim_Sales** — build from `silver.sales_order_header` only after first verifying the cleaned silver column `sales_person` exists in that DataFrame. Apply salesperson cleaning only to `soh.sales_person` (not to any guessed variant such as `salesperson` or `sales_person_name`): strip leading `adventureworks/`, `adventure-works/`, or `adventure works/` case-insensitively, then strip trailing digits, then trim. If `sales_person` is absent in `silver.sales_order_header`, do not create/write `gold.dim_sales`, and append a FAIL/ERROR row to `test.test_results` with `table_name = 'dim_sales'` and details explaining that the source schema does not provide a salesperson column. Do not call `select`, cleaning transforms, or `write` against a non-existent `sales_person` column. The reporting/semantic-model steps must treat `dim_sales` as optional and must first verify that `gold.dim_sales` was actually materialized before trying to read its Delta path or include it in model relationships.
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
9. Add a gold materialization manifest check at the end of the gold notebook: record for each expected gold table (`dim_customer`, `dim_product`, `dim_sales`, `fact_sales`) whether it was CREATED, SKIPPED, or FAILED, plus the physical table name/path if created. Persist this status to `test.test_results` or a small manifest table. Any downstream notebook must consult this manifest/status before attempting to read `Tables/gold/<table_name>`.
10. Before any notebook reads a gold Delta path directly, first verify the table/path exists. If a gold table was skipped or the path does not exist, do not call `spark.read.format('delta').load(...)` for that table; instead exclude it from schema introspection/model generation and append an ERROR/WARN result naming the missing table and reason.

## Semantic model

Apply: https://github.com/RuiRomano/powerbi-agentic-plugins/tree/main/plugins/powerbi/skills/powerbi-semantic-model-authoring (Direct Lake guidelines, modeling-guidelines, TMSL/TMDL syntax).

Create a **Direct Lake** semantic model named `{target_lakehouse_name}_sm` on the gold tables, with:
- Tables: include `dim_customer`, `dim_product`, and `fact_sales` unconditionally; include `dim_sales` only if `gold.dim_sales` was actually created and its Delta path exists. Do not attempt schema discovery or model binding for any missing/skipped gold table.
- Relationships:
  - `fact_sales[customer_id]` → `dim_customer[customer_id]` (many-to-one)
  - `fact_sales[product_id]` → `dim_product[product_id]` (many-to-one)
  - Add `fact_sales[sales_person]` → `dim_sales[sales_person]` (many-to-one) only when both the `dim_sales` table exists in gold and `fact_sales` physically contains `sales_person`.
- Explicit measures (PascalCase, with format strings):
  - `[Total Sales] = SUM(fact_sales[line_total])` — currency
  - `[Total Discount Amount] = SUMX(fact_sales, fact_sales[line_total] * fact_sales[unit_price_discount])` — currency
  - `[Discount %] = DIVIDE([Total Discount Amount], [Total Sales])` — percent
  - Add `[Distinct Salespersons] = DISTINCTCOUNT(dim_sales[sales_person])` only when `dim_sales` exists; otherwise omit this measure from the model and record the omission in `test.test_results`.
  - `[Sales Count] = COUNTROWS(fact_sales)`
- Mark `fact_sales[order_date]` as a date column so time intelligence works without a separate date dim.
- Hide all FK columns from the report view.
- The semantic-model notebook must not fail if `dim_sales` is missing. It must dynamically derive the set of available gold tables from the materialization manifest / existence checks and generate a valid reduced model from only those tables.

## Report

Apply: https://github.com/RuiRomano/powerbi-agentic-plugins/tree/main/plugins/powerbi/skills/powerbi-report-authoring (PBIR format).

Create a Power BI report named `{target_lakehouse_name}_rpt` bound to the semantic model above. Pages:

1. **Sales overview** — always include card visual for `[Total Sales]` and `[Sales Count]`; include `[Distinct Salespersons]` card and the two salesperson bar charts only if the semantic model contains `dim_sales` and measure `[Distinct Salespersons]`. If `dim_sales` is absent, replace those visuals with a text box noting `Salesperson dimension unavailable for this run because sales_person was missing in source schema.` Keep the line chart sales by month.
2. **Product mix** — matrix of `[Total Sales]` by `dim_product[category]` rows × `dim_product[subcategory]` columns; bar chart top 10 products by `[Total Sales]`.
3. **Data quality** — table visual showing the latest `test_results` (filter to MAX(run_id)), columns: layer, table_name, test_name, status, actual, expected; conditional formatting on `status` (green=PASS, red=FAIL, amber=ERROR).

The reporting notebook must not enumerate or read schemas for a hard-coded gold table list without checking existence first. When building any `schema_map` or similar structure for gold tables, use only the subset of gold tables that were actually materialized and whose Delta paths exist; specifically, do not attempt to load `Tables/gold/dim_sales` when `dim_sales` was skipped.

## Data Agent

Create a **Data Agent (AISkill)** named `{target_lakehouse_name}_agent`, **grounded on the semantic model** created above (not on the raw tables), so it answers through curated measures and relationships.

**System instructions for the agent:**

> You are a sales analytics assistant for the AdventureWorksLT business. Answer questions about sales performance, salespeople, products, customers, and data quality. Always use the curated semantic-model measures (`Total Sales`, `Total Discount Amount`, `Discount %`, `Distinct Salespersons`, `Sales Count`) and the cleaned `dim_sales[sales_person]` dimension when those model objects exist. NEVER aggregate columns directly from `fact_sales` when an equivalent measure exists.
>
> When asked "who", return the cleaned salesperson name (no `adventureworks/` prefix, no trailing digit) if the salesperson dimension is present in the model. When asked "how much", format numbers as currency with two decimals. When asked about data quality, query the latest `test_results` (filter to MAX(run_id)) and summarise PASS/FAIL counts per layer and table. If the salesperson dimension is unavailable for this run, say so clearly rather than inventing it. If the question cannot be answered from this model, say so clearly and suggest a follow-up. Be concise (3-5 sentences) unless the user asks for a deep dive.

**Domain context to inject:**

- "Salesperson names are stored cleaned in `dim_sales[sales_person]` when that table exists; the raw value in `salesorderheader[sales_person]` had an `adventureworks/` prefix and a trailing employee-number digit, both stripped."
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
- If `dim_sales` or salesperson measures are not present in the current semantic model, do not answer salesperson-specific questions as if that data exists; state that the dimension was not built for this run.

If the AISkill REST endpoint is not yet GA in the tenant, the reporting notebook will print these full instructions and starter questions so they can be pasted when creating the Data Agent manually in the Fabric portal.

## Known constraints (from prior runs)
- In gold-layer joins, any column name present in more than one input table must always be referenced with an alias prefix; never use bare shared names like `address_id`, `rowguid`, or `modified_date`.
- For `dim_customer`, `customer_address.address_id` is the canonical joined address key to project as `address_id`. If an additional address-table identifier is retained, it must be renamed to avoid collisions.
- In `dim_customer`, a bare `rowguid` reference is invalid because `customer`, `customer_address`, and `address` each provide `rowguid`. Select exactly one qualified source as canonical, and rename any others with source-specific prefixes.
- Before applying any gold-layer transform to salesperson data, validate that `silver.sales_order_header` actually contains the snake_case column `sales_person`; if not, branch gracefully and log the issue to `test.test_results` instead of raising `UNRESOLVED_COLUMN`.
- The gold write path must never receive a DataFrame with duplicate column names. This was previously hit in `dim_customer` from carrying overlapping columns across `customer`, `customer_address`, and `address` (including `rowguid`, `modified_date`, silver audit columns, and bronze metadata columns). Explicitly project a unique final schema before writing Delta.
- `silver.product` in this source does not provide `discontinued_date`. Any `dim_product` logic must validate source columns first and must not select `p.discontinued_date` unless that column is confirmed to exist.
- Reporting/semantic-model code must never hard-code reads of every expected gold path. `dim_sales` may be intentionally skipped when `silver.sales_order_header.sales_person` is absent, so `Tables/gold/dim_sales` may not exist. Always check the gold materialization manifest or Delta path existence before loading a gold table.