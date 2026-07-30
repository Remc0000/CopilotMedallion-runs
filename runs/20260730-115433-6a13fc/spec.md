# Run Spec 20260730-115210-7c6906

## Updated specs

### Iteration 1 — 2026-07-30 12:00:06Z — failed layer: silver (run: 20260730-115433-6a13fc)
- **Root cause (1-line summary)**: Silver completed without producing any catalog-discoverable Delta tables in the `silver` schema, so Gold refused to build from spec-only context.
- **Cross-table audit**:
  - `customeraddress`: yes — every Silver table can be affected if writes use an unqualified name/path, failures are swallowed, or the table is not verified in the catalog.
  - `salesorderdetail`: yes — subject to the same Silver write and discoverability contract.
  - `productdescription`: yes — subject to the same Silver write and discoverability contract.
  - `customer`: yes — subject to the same Silver write and discoverability contract.
  - `productcategory`: yes — subject to the same Silver write and discoverability contract.
  - `productmodel`: yes — subject to the same Silver write and discoverability contract.
  - `salesorderheader`: yes — subject to the same Silver write and discoverability contract.
  - `productmodelproductdescription`: yes — subject to the same Silver write and discoverability contract.
  - `product`: yes — subject to the same Silver write and discoverability contract.
  - `address`: yes — subject to the same Silver write and discoverability contract.
- **Fix approach**: GENERALIZE — the failure is a uniform layer-output problem rather than a table-specific schema issue, so one mandatory catalog-write and post-write verification contract now applies to all ten Silver tables.
- **What was changed**:
  - Tightened `## Silver` to require schema-qualified `saveAsTable('silver.<table>')` writes and prohibit path-only or temporary-view outputs.
  - Added immediate per-table catalog/provider verification plus a final exact expected-versus-discovered table assertion.
  - Required Silver to raise after recording results if any expected table is missing, preventing a false successful completion with zero or partial discoverable outputs.

### Iteration 2 — 2026-07-30 12:02:08Z — failed layer: silver (run: 20260730-115433-6a13fc)
- **Root cause (1-line summary)**: The Silver authoring response contained zero notebook cells, so no executable Silver notebook was available to transform or write any table.
- **Cross-table audit**:
  - `customeraddress`: yes — the notebook-level empty response prevents its read, transformation, write, and verification.
  - `salesorderdetail`: yes — the same missing-cell failure prevents all processing for this table.
  - `productdescription`: yes — the same missing-cell failure prevents all processing for this table.
  - `customer`: yes — the same missing-cell failure prevents all processing for this table.
  - `productcategory`: yes — the same missing-cell failure prevents all processing for this table.
  - `productmodel`: yes — the same missing-cell failure prevents all processing for this table.
  - `salesorderheader`: yes — the same missing-cell failure prevents all processing for this table.
  - `productmodelproductdescription`: yes — the same missing-cell failure prevents all processing for this table.
  - `product`: yes — the same missing-cell failure prevents all processing for this table.
  - `address`: yes — the same missing-cell failure prevents all processing for this table.
- **Fix approach**: GENERALIZE — this is a notebook-emission failure that uniformly blocks all ten source tables, so the fix mandates a non-empty executable cell contract and a minimal Silver cell plan rather than introducing table-specific changes.
- **What was changed**:
  - Tightened `## Generic guidance` with a mandatory notebook-emission contract that prohibits empty or prose-only authoring responses.
  - Tightened `## Silver` to require at least four executable Python cells covering parameters/helpers, schema setup, the all-table processing loop, and final catalog validation.
  - Required optional work such as `OPTIMIZE` to be omitted or deferred rather than allowing complexity to result in zero emitted cells.

## Inputs
- Workspace: `b4dc08af-f88c-47ab-aa71-7d33d2c473e9`
- Source Lakehouse: **SalesLake** (`040b6dbc-1c93-4448-9b22-cb2c26c79ee9`)
- Tables to ingest into Bronze:
  - `customeraddress`
  - `salesorderdetail`
  - `productdescription`
  - `customer`
  - `productcategory`
  - `productmodel`
  - `salesorderheader`
  - `productmodelproductdescription`
  - `product`
  - `address`
- Target Lakehouse: **SalesAnalytics**

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
- Notebook authoring responses must always contain a non-empty collection of executable notebook cells. Never return an empty cell list, prose-only guidance, a plan without cells, or only markdown/code fences when the build requests notebook cells.
- Every layer notebook must include, at minimum, an executable parameter/setup cell and an executable processing or validation cell. If the full implementation is large, split it into additional cells; never omit all cells because of implementation complexity.
- Prefer a smaller complete executable notebook over a comprehensive but non-executable response. Optional maintenance or presentation work must be deferred rather than causing the required transformation/write cells to be omitted.
- Use defensive column references and assert required columns before every join, filter, `withColumn`, `groupBy`, aggregation, window, and projection that depends on named columns.
- After every join, use aliases and alias-prefixed references in the immediate join projection; materialize uniquely named flat columns before subsequent transformations.
- Before `groupBy` or `agg`, assert that every grouping and aggregation column exists.
- Handle REST responses defensively: use `if x is None: raise RuntimeError(...)` before calling `.get()` or accessing response members.
- Create schemas with `CREATE SCHEMA IF NOT EXISTS bronze|silver|gold|test`, then write with schema-qualified `saveAsTable('<schema>.<table>')`.
- Never use raw ABFSS `.save()` for target tables on the schema-enabled target Lakehouse.
- Put run ID, workspace ID, source/target Lakehouse identifiers, table lists, and configurable behavior in notebook parameter cells.
- Use idempotent overwrite patterns with Delta and `overwriteSchema=true`; overwrite only the table or partition owned by the current run.
- Use error-loud `try/except` blocks that call `_save_error(layer, e, table=tbl)` and re-raise after required per-table recovery/attempts.
- Do not log secrets or retain `password_hash`, `password_salt`, or `credit_card_approval_code` beyond the source-preserving Bronze layer.
- Every generated notebook code cell must begin with a Python `# ---` divider and one to three human-readable `# ` comment lines explaining what the cell does and why. Never emit an uncommented code cell.

### Global Spark column-reference rules (apply to ALL layers: Bronze, Silver, Gold)
These rules exist to prevent recurring `UNRESOLVED_COLUMN` / `AnalysisException` analyzer errors. They are layer-agnostic — apply them anywhere a Spark DataFrame is transformed.

Rule A — No dotted alias strings.
- Never pass dotted strings like "c.customer_id", "ca.address_type", "h.sales_person", or "pc_child.name" to F.col(...), withColumn(...), Window.partitionBy(...), Window.orderBy(...), or select(...). Spark treats "c.customer_id" as a single column literally named c.customer_id, which does not resolve once any projection or rename has been applied.
- Alias scope (.alias("c"), .alias("ca"), ...) is only valid inside the SAME select / join expression that introduces it. Once you produce a new DataFrame via select(...) or withColumn(...), the dotted alias form is gone and you must reference plain column names.

Rule B — Materialize helper columns before they are needed downstream.
- For any column that will later be referenced by a Window, a withColumn, or a downstream join after a projection, first materialize it as a flat, unambiguous helper column (e.g. rank_customer_id, rank_address_type, sales_person_source) in the same select that introduces the join aliases.

Rule C — Do not drop a column before its last consumer has run.
- Before adding a withColumn, verify every F.col(...) referenced by that expression still exists on the DataFrame at that step. If a previous select(...) projection removed it, either:
  - (preferred) move the withColumn BEFORE the projection that drops the source column, OR
  - keep the source column in the projection, OR
  - re-derive the value from a column that IS still present (often a boolean/flag that was computed earlier from the same source).
- Example of the failure to avoid: dropping discontinued_date in a select(...) and then later writing F.when(F.col('discontinued_date').isNotNull(), ...) inside withColumn('is_sellable_currently', ...). The column is gone and Spark raises UNRESOLVED_COLUMN.
- When a boolean flag derived from a raw column already exists on the DataFrame (e.g. is_discontinued derived from discontinued_date), prefer reusing the flag (F.col('is_discontinued')) over re-reading the dropped raw column.

Rule D — Order of derived-column computations matters.
- When building several derived columns where one depends on another (e.g. is_discontinued, then is_sellable_currently which uses is_discontinued), add them in dependency order with sequential withColumn calls, and reference the already-derived flag in the next expression — do NOT reach back to a raw source column that may have been dropped.

Rule E — Validate schema between non-trivial transformation steps.
- After any select(...) / drop(...) / heavy withColumn chain, and BEFORE the next step that depends on specific columns, assert those columns exist. Fail fast with an error message that names the missing column and the DataFrame variable, so the auto-fixer gets an actionable diagnostic instead of a deep analyzer stack trace.

Rule F — Self-check pattern for every withColumn / Window.
- For every withColumn(name, expr) and every Window definition, confirm: "Every column referenced inside expr / inside the window's partitionBy / orderBy exists on the DataFrame at this exact point." If not, fix per Rule C before generating the code.

Rule G — Optional-column helpers must return typed Column nulls, not Python None.
- When defining a helper like `_maybe(df, name)` that returns the column if it exists on the DataFrame and a fallback otherwise, NEVER return Python `None`. Spark functions (`F.coalesce`, `F.greatest`, `F.least`, `F.concat`, `F.when(...).otherwise(...)`, etc.) reject `None` arguments with `PySparkTypeError: [NOT_COLUMN_OR_STR]` and the cell crashes BEFORE any later fallback (e.g. `F.current_timestamp()`) gets a chance to satisfy the call.
- Correct pattern — return a typed null literal as a Spark Column:
  ```
  def _maybe(df, name, dtype='timestamp'):
      return F.col(name) if name in df.columns else F.lit(None).cast(dtype)
  ```
  Pick the `dtype` to match the surrounding expression (`'timestamp'` for date/time coalesces, `'string'` for text, `'double'` for numeric, etc.) so Spark can resolve the result type without ambiguity.
- Alternative pattern (when the helper genuinely cannot know the dtype) — filter `None`s at the call site BEFORE invoking the Spark function:
  ```
  candidates = [c for c in (_maybe(df, 'modified_date'), _maybe(df, 'order_date')) if c is not None]
  df = df.withColumn('source_dt', F.to_date(F.coalesce(*candidates, F.current_timestamp())))
  ```
  Either approach is acceptable, but **never pass Python `None` directly into a Spark function**.
- Applies to ALL optional-column lookups across Bronze, Silver, Gold — including audit-timestamp coalesces, optional-key joins, fallback string formatting, etc. This is a layer-agnostic rule.

Rule H — Per-table isolation; one table's failure must not cancel the Spark session for the rest.
- Spark cancels the entire session when one statement crashes. If your notebook builds a single chained plan that touches every source table (one big SELECT, one big DataFrame, one big SQL script), any one table's failure kills ALL tables.
- ALWAYS process source tables in a `for tbl in source_tables:` loop where each iteration is a SELF-CONTAINED unit: read → transform → write → record-result → recover. Wrap the loop body in `try/except` that calls `_save_error(layer, e, table=tbl)` and APPENDS the failure to a results dict, then **re-raises only AFTER the loop has attempted all tables** (or, if your spec says "fail-fast-first-table", re-raise immediately — but per-table-isolated by default).
- Do NOT build a single multi-CTE Spark SQL statement that joins/transforms many source tables in one shot. Each table's transform is its own DataFrame chain with its own `.write` call.
- Do NOT share intermediate temp views across tables. Temp views from one iteration must not be assumed to exist in the next. If you need cross-table joins (typical for Gold), do them in a SECOND loop AFTER all per-table Silver/Gold writes are complete.

Rule I — Optional audit columns on junction / bridge / view tables.
- In typical operational sources (AdventureWorksLT, Northwind, AdventureWorks2019, etc.), entity tables (Customer, Product, SalesOrderHeader) have system audit columns: `ModifiedDate`, `rowguid`. **Junction / bridge tables** (CustomerAddress, ProductModelProductDescription, SalesTerritoryHistory) typically have only the FK columns and may have NO ModifiedDate and NO rowguid. **Views** (vGetAllCategories, vProductAndDescription) may have whatever columns the underlying query projects — frequently NO audit columns.
- When you write Silver dedup / tie-break / audit logic, you MUST NOT assume `modified_date` (or any other audit column) exists on every table. Use `'modified_date' in df.columns` as a guard and fall back to:
  - For dedup: a deterministic ranking expression that uses only the natural-key columns (`row_number().over(Window.partitionBy(*pk_cols).orderBy(*pk_cols))`), OR a literal F.lit(timestamp).
  - For `source_dt` / `_silver_ts`: a typed null literal (`F.lit(None).cast('timestamp')`) or `F.current_timestamp()`.
- Junction tables: dedupe on the COMPOSITE FK key (e.g. `(customer_id, address_id, address_type)`) — never on a non-existent surrogate key.
- View tables: project ONLY the columns actually returned by the view. Do not assume any standard naming.

Rule J — Validate column existence BEFORE the expensive transform.
- For every join, withColumn, groupBy, agg, or filter that names a specific column, ASSERT the column exists in `df.columns` BEFORE the line that uses it. Pattern:
  ```
  for required in ('customer_id', 'order_date'):
      if required not in df.columns:
          raise RuntimeError(f"[{layer}] {tbl}: required column '{required}' missing; available={df.columns}")
  # … now the join / withColumn that uses customer_id and order_date
  ```
- Catches missing-column bugs in a SPECIFIC cell with a SPECIFIC table name, instead of a session-wide Spark cancellation 30 minutes later that the auto-fixer can't pinpoint.
- Especially important AFTER a select(), drop(), or rename() — re-validate before the next consumer of those columns.

Rule K — Resilience to partial output: every layer MUST write Delta tables the next layer can discover.
- The build pipeline runs each layer's notebook then inspects the lakehouse for the layer's output Delta tables before generating the next layer. If Bronze runs "successfully" (Spark Completed) but writes zero discoverable tables in the `bronze` schema, the build hard-fails with "prior layer produced no discoverable tables".
- To guarantee discoverability, the Bronze notebook MUST:
  - Write via `df.write.format('delta').mode('overwrite').option('overwriteSchema','true').partitionBy(...).saveAsTable(f"bronze.<flat>")` (after `spark.sql('CREATE SCHEMA IF NOT EXISTS bronze')`) for every source table, where `<flat>` is the lowercased last segment of `table_relative_path`. NEVER write target tables with abfss `.save(path)` — on this SCHEMA-ENABLED lakehouse a raw .save() to `Tables/bronze/<t>` lands at a broken nested `Tables/Tables/bronze/<t>` path the discovery + SQL endpoint cannot see.
  - Print a final summary line `print(json.dumps({{"bronze_results": {{<table>: {{"rows": N, "path": ...}}, ...}}}}))` listing every table actually written. Use this as a self-check.
  - Raise (not just log) if zero tables were written by the end of the notebook.
- Same rule applies recursively to Silver (`silver.<table>`) and Gold (`gold.<table>` + `test.test_results`) — each via `CREATE SCHEMA IF NOT EXISTS` + saveAsTable.

Rule L — Disambiguate shared columns in join projections (avoid AMBIGUOUS_REFERENCE).
- When you `select(...)` directly off a join whose sides share a column name, selecting that column as a BARE string raises `[AMBIGUOUS_REFERENCE]` and cancels the whole Spark session. Typical Gold offenders: joining product `p` with product_model `m` (both expose `product_model_id`), or product `p` with product_category `pc` (both expose `product_category_id`), or any dimension built from several aliased source tables that carry the same key.
- Inside the SAME join+select expression the alias scope is still live, so reference EVERY shared/overlapping column with its alias and rename it explicitly: `F.col('p.product_model_id').alias('product_model_id')`, `F.col('pc.parent_product_category_id').alias('category_id')`. A bare string in a join `select` is ONLY safe for a column that exists on EXACTLY ONE side of the join.
- Before emitting a join's select list, enumerate the columns on each side; for any name present on more than one side, alias-qualify the side you want. When in doubt in a multi-table join, alias-qualify ALL columns in the select list — it is always safe and never ambiguous.
- This is the in-join counterpart to Rule A: dotted alias references (`F.col('p.col')`) are valid ONLY inside the join/select that introduces the alias; once the joined DataFrame has been materialized by that select, switch back to plain, already-renamed column names (Rule A).
- Concrete failure to avoid: `.select(F.col('p.product_id').alias('product_id'), 'product_number', 'product_model_id', ...)` after `p.join(m, ...)` — `product_model_id` exists on both `p` and `m`, so it MUST be `F.col('p.product_model_id').alias('product_model_id')` (or the `m.` side), never the bare `'product_model_id'`.

## Bronze
- Create the `bronze` schema and land all ten selected source tables as source-faithful Delta snapshots.
- Preserve original business columns and data types; add `_ingest_run_id`, `_ingested_at`, `_source_lakehouse`, and `_source_table`.
- Write each table independently with `mode('overwrite')`, `overwriteSchema=true`, and `saveAsTable('bronze.<table>')`.
- Treat ingestion as a reproducible full snapshot because no reliable source change-tracking field beyond mutable `ModifiedDate` is supplied.
- Do not partition small master or junction tables. Partition `salesorderheader` by derived `_order_year` and `_order_month`; partition `salesorderdetail` only if runtime volume justifies it, preferably by a bounded hash bucket of `SalesOrderID` rather than high-cardinality IDs.
- Bronze retains source-sensitive fields for traceability, but access to `customer.PasswordHash`, `customer.PasswordSalt`, and `salesorderheader.CreditCardApprovalCode` must be restricted. They must not flow into Silver or Gold.
- Record row counts and target table names in `bronze_results`; attempt every table and raise after the loop if any table failed or if zero tables were written.

## Silver
- Resume from the existing `bronze` Delta tables; do not re-ingest or rewrite Bronze during this retry.
- The Silver authoring response must emit a non-empty notebook containing at least four executable Python code cells through the notebook-cell authoring mechanism. Do not return an empty cell array, prose-only guidance, a markdown-only plan, or code fences in place of notebook cells.
- Use this minimum executable cell sequence:
  1. Parameters, imports, run metadata, expected table list, result collections, and helper functions.
  2. `silver` schema creation plus validation that all expected `bronze.<table>` inputs are catalog-readable.
  3. The per-table read → snake_case transform → validate → deduplicate → write → read-back verification loop for all ten tables.
  4. Final `SHOW TABLES IN silver` comparison, machine-readable summary, and deferred raise for accumulated failures or missing tables.
- Every one of these cells must begin with the required Python `# ---` divider and explanatory comments. Additional cells may be used, but the four responsibilities above must remain executable and must not be replaced by narrative.
- If response-size or implementation complexity is constrained, prioritize the four required cells and all ten catalog writes. `OPTIMIZE` is optional maintenance for this retry and must be omitted or deferred before omitting any required notebook cell, transformation, write, or catalog verification.
- Create the `silver` schema with `spark.sql("CREATE SCHEMA IF NOT EXISTS silver")` before processing any table.
- Use this exact expected table set for both processing and final validation: `customeraddress`, `salesorderdetail`, `productdescription`, `customer`, `productcategory`, `productmodel`, `salesorderheader`, `productmodelproductdescription`, `product`, and `address`.
- Process all ten expected tables independently. For each `tbl`, read the catalog table `bronze.<tbl>`, create the source-aligned snake_case DataFrame, and write it only as a managed/catalog Delta table using:
  ```
  silver_df.write.format("delta").mode("overwrite").option("overwriteSchema", "true").saveAsTable(f"silver.{tbl}")
  ```
  Do not use `.save(path)`, ABFSS output paths, unqualified `saveAsTable(tbl)`, temporary views, global temporary views, or in-memory DataFrames as substitutes for the required `silver.<tbl>` table.
- Immediately after each write, verify all of the following before recording success:
  - `spark.catalog.tableExists(f"silver.{tbl}")` is true.
  - `spark.table(f"silver.{tbl}")` can be read successfully.
  - `DESCRIBE DETAIL silver.<tbl>` reports `format = 'delta'`.
  - The read-back schema contains the table's required key columns listed below.
  - Record the read-back row count and fully qualified target name in `silver_results`; do not mark a table successful based only on completion of the write call.
- A valid empty source may produce a zero-row Silver table, but the catalog table must still be created and discoverable. Zero rows are not a reason to skip `saveAsTable`.
- Create source-aligned, snake_case Delta tables in the `silver` schema. Trim strings, normalize blank strings to null where appropriate, preserve valid decimals/timestamps, and add `_silver_run_id`, `_silver_ts`, and `_source_modified_at`.
- Deduplicate by the following actual keys, retaining the greatest `modified_date`, then `rowguid` as a deterministic tie-breaker when those audit columns exist:
  - `customer`: `customer_id`.
  - `address`: `address_id`.
  - `product`: `product_id`.
  - `productcategory`: `product_category_id`.
  - `productmodel`: `product_model_id`.
  - `productdescription`: `product_description_id`.
  - `salesorderheader`: `sales_order_id`.
  - `salesorderdetail`: `sales_order_detail_id`; additionally assert `(sales_order_id, sales_order_detail_id)` uniqueness.
  - `customeraddress`: composite `(customer_id, address_id, address_type)`.
  - `productmodelproductdescription`: composite `(product_model_id, product_description_id, culture)`.
- For `customeraddress` and `productmodelproductdescription`, do not require `modified_date` or `rowguid`; if absent, deduplicate deterministically using only the listed composite key. For every table, assert its listed key columns exist after snake_case projection and again in the catalog read-back table.
- Validate actual foreign keys without dropping unresolved rows: order header to customer and bill/ship address; order detail to header and product; product to category/model; product-description bridge to model/description; customer-address bridge to customer/address.
- Normalize `customer.sales_person` into `sales_person_username`: trim, replace `/` with `\` if encountered, take the text after the final backslash, and lowercase. For example, `adventure-works\jillian0` becomes `jillian0`; the requested example ending in `jillian` does not match the supplied sample text, so no trailing digit will be removed.
- Exclude password hash/salt and credit-card approval code from Silver.
- Filter invalid negative `order_qty`, `unit_price`, `unit_price_discount`, or header monetary amounts into an error/quarantine result rather than silently correcting them. Permit null optional dates and descriptive attributes.
- Run `OPTIMIZE` after successful, verified writes when notebook generation and execution capacity permits, prioritizing `silver.salesorderheader` by `order_date`, `silver.salesorderdetail` by `sales_order_id`, and `silver.product` by `product_category_id`; avoid unnecessary optimization of very small tables. An `OPTIMIZE` failure must not erase or unregister an already verified Silver table, but it must be recorded explicitly. If necessary, defer all `OPTIMIZE` statements until after the required tables and validations have completed.
- After all per-table attempts, enumerate the catalog with `SHOW TABLES IN silver` and compare the discovered names against the exact expected table set. Treat the comparison as case-insensitive, but require every expected table to be present; report both `missing_tables` and `unexpected_tables`.
- Print a final machine-readable summary containing `silver_results`, the expected table list, the discovered table list, and `missing_tables`.
- If any per-table transformation/write/read-back verification failed, or if any expected Silver table is missing, call `_save_error('silver', e, table=<affected table or '__layer__'>)` and raise a `RuntimeError` after all ten tables have been attempted. Never allow the Silver notebook to finish successfully when zero or only a partial set of discoverable `silver.*` Delta tables exists.

## Gold
- Build a Direct Lake star schema at sales-order-line grain and write all outputs with schema-qualified Delta tables.
- `gold.dim_order_date`: role-specific date dimension spanning the minimum through maximum `salesorderheader.order_date`, with date, year, quarter, month number/name, year-month, week, and day attributes. Key is integer `yyyymmdd`.
- `gold.dim_ship_date`: separate role-specific date dimension spanning non-null `ship_date`, with the same date attributes and a `Year > Quarter > Month > Date` hierarchy. Use a designated unknown/not-shipped member for null ship dates.
- `gold.dim_customer`: one row per `customer_id`, retaining relevant identity, company, email, phone, and regional address attributes; exclude credentials and raw salesperson.
  - NOTE: `customer` has no `address_id`, so a direct Customer-to-Address join is impossible, and the user explicitly prohibited using `customeraddress`.
  - Implement the requested combination by ranking each customer's orders by `order_date`, `modified_date`, and `sales_order_id`, then joining the latest order's `ship_to_address_id` to `address.address_id`. This yields a deterministic current shipping region without using `customeraddress`.
  - Customers without an order receive an unknown address/region. This is a current-profile dimension and does not preserve historical customer-region changes.
- `gold.dim_sales_person`: distinct normalized `sales_person_username` values sourced from Customer, with a stable hash surrogate key and display name equal to the normalized username. Include an Unknown member.
- `gold.dim_order`: one row per `sales_order_id`, holding suitable non-additive/header descriptors: revision number, status, online-order flag, purchase-order number, account number, ship method, and comment. Exclude credit-card approval code and header monetary measures.
- `gold.dim_product`: one row per `product_id`, combining Product with:
  - The leaf `productcategory` row on `product_category_id`.
  - A self-join from leaf category to `parent_product_category_id`; when a parent exists, expose parent `name` as category and leaf `name` as subcategory. For root products, expose leaf name as category and null/“Uncategorized” as subcategory.
  - `productmodel.name` renamed to `model_name`.
  - `productmodelproductdescription` filtered case-insensitively to `culture='en'`, then joined to `productdescription`; expose only its `description`.
  - If multiple English descriptions exist for one model, retain the greatest bridge `modified_date`, then greatest `product_description_id`.
  - Retain relevant merchandising attributes such as product number, color, size, weight, cost, list price, sell dates, discontinued status, and thumbnail filename; omit binary thumbnail data from the analytical dimension.
- `gold.fact_sales_order`: combine `salesorderheader` and `salesorderdetail` on `sales_order_id`, at one row per `sales_order_detail_id`.
  - Foreign keys: order-date key, ship-date key, customer key, salesperson key derived through the order customer, order key, and product key.
  - Measures: order quantity, unit price, unit discount fraction, gross sales amount (`quantity × unit price`), discount amount (`gross × unit_price_discount`), net sales amount (`gross − discount`), allocated tax, allocated freight, and allocated total amount.
  - Allocate header tax and freight across lines in proportion to line net sales; if an order has zero net sales, allocate evenly across its lines. Reconciliation must equal header tax/freight within decimal rounding tolerance.
  - Keep dates and descriptive header fields in dimensions rather than duplicating them in the fact.
- Do not create a separate address dimension because the requested Gold schema places address attributes in Customer. Regional analysis uses Customer country, state/province, city, and postal code.
- Build dimensions first, then the fact in a second isolated phase. Include explicit Unknown dimension members so unresolved optional keys remain analyzable.

## Test
- Create `test.test_results` with: `run_id`, `test_name`, `layer`, `table_name`, `status`, `actual`, `expected`, `details`, `checked_at`.
- Each test execution appends exactly one result row using `saveAsTable('test.test_results')`; test exceptions produce `ERROR`, call `_save_error('test', e, table=...)`, and continue so all tests run before the notebook raises for errors.
- Standard test 1 — row-count reconciliation: compare Bronze and Silver per source table; require Silver to be within 1% of Bronze, while reporting expected dedup/quarantine exceptions. Also require fact rows to match the count of valid Silver order-detail rows with a matching header.
- Standard test 2 — no-null dimension PKs: assert non-null keys in `dim_order_date`, `dim_ship_date`, `dim_customer`, `dim_sales_person`, `dim_order`, and `dim_product`.
- Standard test 3 — unique dimension PKs: assert each Gold dimension key occurs exactly once.
- Standard test 4 — referential integrity: verify every fact order-date, ship-date, customer, salesperson, order, and product FK resolves to its dimension, including the designated Unknown members.
- Standard test 5 — business-rule sanity: require `net_sales_amount = gross_sales_amount - discount_amount`, `unit_price_discount` between 0 and 1, non-negative quantity/amounts, and order-level allocated tax/freight to reconcile to header values within `0.01`.
- Add domain checks for `ship_date >= order_date` when shipped, `due_date >= order_date`, category parent IDs not self-referencing, and exactly one Gold product row per `product_id`.
- Fail the run after recording results if any required test has `FAIL` or `ERROR`.

## Semantic model
- Create a Direct Lake semantic model over all six Gold dimensions and `gold.fact_sales_order`.
- Relationships are one-to-many, single-direction from each dimension to fact: OrderDate, ShipDate, Customer, SalesPerson, Order, and Product.
- Mark `dim_order_date` and `dim_ship_date` as date tables. Hide technical keys and raw additive helper columns from report consumers.
- Hierarchies:
  - Order Date: `Year > Quarter > Month > Date`.
  - Ship Date: `Year > Quarter > Month > Date`.
  - Customer Geography: `Country Region > State Province > City > Postal Code > Customer`.
  - Product: `Category > Subcategory > Model Name > Product`.
  - Sales Person: `Sales Person`.
  - Order: `Online Order Flag > Status > Order`.
- Explicit DAX measures:
  - `Net Sales = SUM(fact_sales_order[net_sales_amount])`
  - `Gross Sales = SUM(fact_sales_order[gross_sales_amount])`
  - `Discount Amount = SUM(fact_sales_order[discount_amount])`
  - `Discount Percentage = DIVIDE([Discount Amount], [Gross Sales])`
  - `Average Order Sales = AVERAGEX(VALUES(dim_order[order_key]), [Net Sales])`
  - `Maximum Order Sales = MAXX(VALUES(dim_order[order_key]), [Net Sales])`
- Also expose `Order Count = DISTINCTCOUNT(fact_sales_order[order_key])` and `Average Unit Discount Percentage = AVERAGE(fact_sales_order[unit_price_discount])` if model scope permits.
- Format sales/discount amounts as currency and percentage measures as percentages. Set default summarization of IDs and keys to “Do not summarize.”

## Report
- Use a polished retail/product-sales theme derived from the dominant `productcategory.name` values discovered at runtime. Use category imagery/colors only when they remain accessible and avoid hard-coding a theme before profiling the actual category names.
- Page 1 — **Regional Sales Performance**:
  - KPI cards for Net Sales, Average Order Sales, Maximum Order Sales, Order Count, and Discount Percentage.
  - Azure Maps bubble or filled-map visual using country/state/city, sized by Net Sales and conditionally colored from low to high performance.
  - Ranked regional bar chart for Net Sales with Average Order Sales and Maximum Order Sales in tooltips.
  - Geography hierarchy slicer and product-category slicer.
- Page 2 — **Sales Trends and Products**:
  - Monthly line/area chart using the Order Date hierarchy and Net Sales, with Gross Sales as a comparison.
  - Category/subcategory treemap and product/model ranked bar chart.
  - Cards for Average Order Sales and Maximum Order Sales.
  - Drill-through product detail panel showing description, model, category, quantity, sales, and discount percentage.
- Page 3 — **Orders, Salespeople, and Discounts**:
  - Salesperson leaderboard by Discount Amount and Discount Percentage, with Net Sales and Order Count tooltips.
  - Order-status and online/offline distribution visuals.
  - Order-level matrix with order date, ship date, customer, salesperson, sales, and discount metrics.
  - Conditional formatting to identify unusually high discount percentages.
- Page 4 — **Data Quality**:
  - PASS/FAIL/ERROR cards, test-results trend, failures by layer/table, and a detailed results table sourced from `test.test_results`.
- Use a consistent grid, accessible contrast, restrained category-inspired accent colors, dynamic titles, report-page tooltips, and drill-through. Do not imply geographic precision below the available country/state/city/postal attributes.

## Data Agent
- Create an AISkill grounded only on the certified SalesAnalytics semantic model and its documented measures.
- Role: “Sales Performance Analyst” specializing in regional performance, monthly trends, product-category performance, order behavior, and salesperson discount analysis.
- Domain hints:
  - “Sales” means the explicit `Net Sales` measure unless the user asks for gross sales.
  - “Average sales” means `Average Order Sales`; “maximum sales” means `Maximum Order Sales`.
  - “Discount percentage” means `Discount Percentage`, weighted by gross sales.
  - Regions follow the Customer Geography hierarchy based on the latest shipping address derived from orders.
  - Salespeople are normalized usernames originating from Customer.
  - Product rollups use Category, Subcategory, Model, and Product.
- Starter questions:
  - Which regions have the highest and lowest net sales?
  - What are average and maximum order sales by country and state?
  - How have monthly net sales changed over the last twelve months?
  - Which product categories and subcategories generate the most sales?
  - Which products are sold within each category?
  - Which salespeople provide the largest discount amounts?
  - Which salespeople have the highest discount percentage?
  - How do online and offline order sales compare?
  - Which orders have unusually high discounts?
  - Are any data-quality tests currently failing?
- Guardrails:
  - Use only semantic-model tables, relationships, hierarchies, and explicit measures; never query Bronze/Silver directly.
  - Never expose password hashes, password salts, credit-card approval codes, row GUIDs, or hidden technical keys.
  - State the date range, geography level, and active date role used in every time-sensitive answer.
  - Use Order Date by default; use Ship Date only when explicitly requested and clearly label it.
  - Do not sum percentages or average pre-aggregated regional percentages; use the certified measures.
  - Explain that Customer geography is the latest known shipping geography and is not a historical address snapshot.
  - Do not infer salesperson identity beyond the normalized username.
  - If a requested region, metric, or product attribute is absent, say so rather than inventing it.
  - Surface relevant data-quality failures when they could affect an answer and avoid presenting failed reconciliations as authoritative.