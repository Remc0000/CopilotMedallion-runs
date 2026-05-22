# Run Spec 20260522-080700-b92e23

## Updated specs

### Iteration 1 — 2026-05-22 08:10:47Z — failed layer: bronze (run: 20260522-081018-b7952d)
- **Root cause (1-line summary)**: Bronze build failed before execution because the notebook generator returned zero code cells, so the pipeline had no runnable Bronze notebook.
- **Cross-table audit**: Address: yes — would be skipped entirely if no Bronze notebook is generated; Customer: yes — same generator-level failure; CustomerAddress: yes — same generator-level failure; Product: yes — same generator-level failure; ProductCategory: yes — same generator-level failure; ProductDescription: yes — same generator-level failure; ProductModel: yes — same generator-level failure; ProductModelProductDescription: yes — same generator-level failure; SalesOrderDetail: yes — same generator-level failure; SalesOrderHeader: yes — same generator-level failure; vGetAllCategories: yes — same generator-level failure; vProductAndDescription: yes — same generator-level failure; vProductModelCatalogDescription: yes — same generator-level failure.
- **Fix approach**: GENERALIZE — this is a notebook-generation failure that affects all Bronze source tables uniformly, so the spec now requires a concrete minimum notebook skeleton and one explicit per-table loop that guarantees emitted cells.
- **What was changed**:
  - Tightened **Generic guidance** to require a non-empty notebook with mandatory parameter/import/helper/execution/summary cells.
  - Tightened **Bronze** with an explicit implementation skeleton, required per-table loop behavior, and a hard requirement to emit at least one code cell and one write attempt summary.

### Iteration 2 — 2026-05-22 08:13:14Z — failed layer: bronze (run: 20260522-081018-b7952d)
- **Root cause (1-line summary)**: Bronze failed again at generation time because the notebook still returned zero runnable cells; the spec needed a stricter fail-closed Bronze cell contract with concrete cell count and required literals.
- **Cross-table audit**: Address: yes — omitted if Bronze notebook has zero emitted cells; Customer: yes — same generator-level risk; CustomerAddress: yes — same generator-level risk; Product: yes — same generator-level risk; ProductCategory: yes — same generator-level risk; ProductDescription: yes — same generator-level risk; ProductModel: yes — same generator-level risk; ProductModelProductDescription: yes — same generator-level risk; SalesOrderDetail: yes — same generator-level risk; SalesOrderHeader: yes — same generator-level risk; vGetAllCategories: yes — same generator-level risk; vProductAndDescription: yes — same generator-level risk; vProductModelCatalogDescription: yes — same generator-level risk.
- **Fix approach**: GENERALIZE — the failure is still notebook-emission level and can affect every listed Bronze source uniformly, so one stricter generation contract is safer than table-by-table enumeration.
- **What was changed**:
  - Tightened **Generic guidance** with a hard minimum of 4 Bronze code cells and a prohibition on returning prose-only plans, empty placeholders, or deferred implementations.
  - Tightened **Bronze** to require the exact explicit `source_tables` list, at least one concrete `spark.read` and one concrete `df.write.format('delta')...save(...)` statement in emitted code, plus a mandatory final assertion that cell generation produced runnable Bronze logic.

### Iteration 3 — 2026-05-22 08:18:01Z — failed layer: bronze (run: 20260522-081018-b7952d)
- **Root cause (1-line summary)**: Bronze notebook may have run without leaving any discoverable Delta tables under `Tables/bronze/`, so Silver refused to start because prior-layer outputs were not physically present.
- **Cross-table audit**: Address: yes — if written to the wrong physical path or not written at all, it will not be discoverable; Customer: yes — same Bronze output-path/discoverability risk; CustomerAddress: yes — same; Product: yes — same; ProductCategory: yes — same; ProductDescription: yes — same; ProductModel: yes — same; ProductModelProductDescription: yes — same; SalesOrderDetail: yes — same; SalesOrderHeader: yes — same; vGetAllCategories: yes — same; vProductAndDescription: yes — same; vProductModelCatalogDescription: yes — same.
- **Fix approach**: GENERALIZE — the root cause is a uniform Bronze output discoverability contract affecting every listed source object, so one strict all-table path/write/verification rule is safer than table-by-table enumeration.
- **What was changed**:
  - Tightened **Generic guidance** with a mandatory post-write discoverability check: each layer must verify the exact physical Delta path exists immediately after write, and Bronze must raise if any purported success is not discoverable.
  - Tightened **Bronze** to require canonical output paths under `Tables/bronze/<flat_table_name>`, a concrete per-table existence check after each write, a final lakehouse-path audit of all successful writes, and a hard failure if zero discoverable Bronze tables exist.

## Inputs
- Workspace: `4d8fc3ee-8e26-4cf3-b603-e1b238bde935`
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
  - `SalesLT/vGetAllCategories`
  - `SalesLT/vProductAndDescription`
  - `SalesLT/vProductModelCatalogDescription`
- Target Lakehouse: **new**

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
- Use defensive column references everywhere; assert required columns exist before every join, filter, groupBy, agg, window, and withColumn consumer.
- After every join, immediately project/rename to alias-prefixed or flattened output columns so no downstream transform depends on ambiguous duplicate names.
- Before every groupBy/agg, assert the grouping and aggregated columns exist on the current DataFrame.
- For REST/API responses, use defensive handling: `if x is None: raise ...` BEFORE any `.get()` access.
- Do not use `saveAsTable`; write Delta outputs by explicit lakehouse path.
- Put notebook parameters in dedicated parameter cells at the top of each notebook.
- Use idempotent overwrite patterns for deterministic rebuilds.
- Use error-loud `try/except`; call `_save_error(layer, e)` (and `_save_error(layer, e, table=tbl)` in table loops) and then re-raise.
- Each notebook must process source tables in per-table isolated loops where applicable so one table failure does not cancel all unrelated writes.
- Each generated code cell must start with a short markdown-style Python comment block (`# ---` plus 1-3 explanatory comment lines).

### Mandatory notebook-emission rules (apply to EVERY generated notebook)
These rules exist to prevent the generator from returning an empty notebook or zero runnable cells.
- The generator MUST emit a non-empty notebook for every requested layer. Returning zero cells is invalid even if the layer is simple.
- Minimum required code-cell skeleton for every notebook:
  - Cell 1: parameter/setup cell
  - Cell 2: imports/helpers cell
  - Cell 3+: execution cell(s) that read/transform/write data for the layer
  - Final cell: summary/assertion cell that prints structured results and raises if the layer produced no outputs
- Every required cell must begin with the mandated `# ---` comment block.
- If a layer processes multiple tables, the execution logic MUST be expressed as an explicit iterable list plus a `for tbl in ...` loop in emitted code; do not leave table processing implied only in prose.
- If the model cannot infer some optional implementation detail, it must still emit the notebook skeleton with `RuntimeError(...)` placeholders for the unresolved detail rather than returning no cells.
- The final emitted notebook must contain at least one concrete read path/table reference and at least one concrete write path under `Tables/<layer>/...`; a notebook made only of comments or pseudocode is invalid.
- The final summary cell must assert that at least one output was attempted for the layer and must raise a clear error if zero outputs were written/discovered.
- For Bronze specifically, the notebook MUST contain at least 4 runnable code cells. Fewer than 4 code cells is invalid.
- Never return a prose-only implementation plan, TODO list, or placeholder notebook. Emit runnable Python cells first; unresolved details may only appear as explicit runtime `raise RuntimeError(...)` statements inside a code cell.
- At least one execution cell in every layer notebook must contain a concrete DataFrame assignment and a concrete write statement, not just helper-function definitions.

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

Rule K — Resilience to partial output: Bronze MUST write Delta tables that the next layer can discover.
- The build pipeline runs each layer's notebook then inspects the lakehouse for the layer's output Delta tables before generating the next layer. If Bronze runs "successfully" (Spark Completed) but writes zero discoverable tables under `Tables/bronze/`, the build hard-fails with "prior layer produced no discoverable tables".
- To guarantee discoverability, the Bronze notebook MUST:
  - Write via `df.write.format('delta').mode('overwrite').option('overwriteSchema','true').partitionBy(...).save(f"{tgt_base}/Tables/bronze/<flat>")` for every source table, where `<flat>` is the lowercased last segment of `table_relative_path`.
  - Print a final summary line `print(json.dumps({{"bronze_results": {{<table>: {{"rows": N, "path": ...}}, ...}}}}))` listing every table actually written. Use this as a self-check.
  - Raise (not just log) if zero tables were written by the end of the notebook.
- Same rule applies recursively to Silver (`Tables/silver/<table>`) and Gold (`Tables/gold/<table>` + `Tables/test/test_results`).

Rule L — Post-write physical discoverability check is mandatory.
- A write call completing without exception is NOT sufficient. Immediately after every `df.write...save(path)` for Bronze/Silver/Gold/Test outputs, verify the exact expected physical path exists and is non-empty from the lakehouse file system perspective.
- For Bronze specifically, after writing `Tables/bronze/<flat_table_name>`, the same loop iteration MUST confirm that exact folder now exists at the target lakehouse path and only then record the table as `success`.
- Never mark a table as written based only on an attempted write, a row count, or a log line. Mark success only after the physical path verification passes.
- The final layer summary must distinguish:
  - `attempted`
  - `write_succeeded`
  - `discoverable`
  so the pipeline can fail loudly when writes were attempted but not discoverable.

ALSO REQUIRE for every generated notebook: EACH code cell must start with a short markdown comment block (Python `# ---` divider + 1-3 lines of `# ` comments) describing what the cell is doing and why — never emit a cell with no leading comment. Example:
```
# ---
# Read raw bronze table and apply schema enforcement.
# Drops rows where any required key is null.
df = spark.read.table(...)
```
Keep the comments human-readable, not the code repeated in prose. The goal: someone scrolling through the notebook in Fabric can understand each block at a glance.

## Bronze
- Land each listed source object 1:1 into `Tables/bronze/<flat_table_name>` using lowercased last segment names:
  - `address`, `customer`, `customeraddress`, `product`, `productcategory`, `productdescription`, `productmodel`, `productmodelproductdescription`, `salesorderdetail`, `salesorderheader`, `vgetallcategories`, `vproductanddescription`, `vproductmodelcatalogdescription`.
- Read each source table independently from the source lakehouse and write Delta to Bronze with overwrite + overwriteSchema enabled.
- Add standard Bronze metadata to every table:
  - `_run_id`
  - `_ingested_at`
  - `_source_lakehouse_id`
  - `_source_table`
  - `_bronze_ts`
- Preserve source column names as-is in Bronze; no business renames yet.
- Partitioning:
  - Partition large transactional tables by derived ingest date or business date if materialized in the notebook:
    - `salesorderheader`: partition by `orderdate_date` or `_ingest_date`
    - `salesorderdetail`: partition by `_ingest_date`
  - For smaller master tables and views, do not partition unless row volume justifies it.
- Validate that each table produced non-zero discoverable output path even if source row count is zero.
- Views (`vGetAllCategories`, `vProductAndDescription`, `vProductModelCatalogDescription`) are ingested exactly as exposed; no assumption of hidden audit columns beyond those actually present.
- Capture per-table row counts and output a final Bronze summary JSON.
- If any table read fails, log per-table failure, continue other Bronze tables, then raise at end if one or more failed.
- Canonical write location rule:
  - For source object `SalesLT/<Name>`, derive `<flat_table_name>` as the exact lowercased `<Name>` only.
  - Write ONLY to `Tables/bronze/<flat_table_name>`.
  - Do NOT write to any alternative folder such as `Files/...`, `Tables/Bronze/...`, `Tables/bronze/SalesLT_<Name>`, nested source-schema folders, or mixed-case variants.
- Discoverability rule:
  - A Bronze table counts as successful only if the notebook verifies the exact canonical path `Tables/bronze/<flat_table_name>` exists after the write.
  - Row count may be zero; physical table-path existence is the required success criterion.
  - If a write succeeds but the canonical path is not discoverable, record that table as failure and do not include it in the successful Bronze outputs list.

### Bronze notebook generation contract
- The Bronze notebook MUST NOT be omitted or left empty. It must contain, at minimum, these runnable code cells in this order:
  - Parameter cell defining `run_id`, source workspace/lakehouse identifiers, target base path, and the explicit `source_tables` list containing all 13 input objects.
  - Imports/helper cell defining shared helpers such as `_save_error(...)`, optional schema assertion helpers, and a `<source path> -> <flat table name>` mapper.
  - Main execution cell containing one explicit `for tbl in source_tables:` loop that:
    - reads exactly that source object from the source lakehouse,
    - adds the Bronze metadata columns,
    - derives `<flat_table_name>` from the last path segment lowercased,
    - writes to `Tables/bronze/<flat_table_name>`,
    - verifies the exact physical output path exists before recording success,
    - records row count, output path, attempted/write/discoverable status, and error details in a results structure.
  - Final summary cell that prints the Bronze results JSON and raises if zero tables were successfully written.
- Use the exact explicit `source_tables` list in the notebook code; do not rely on lakehouse auto-discovery for this run.
- The explicit list to embed in the emitted Bronze code is:
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
  - `SalesLT/vGetAllCategories`
  - `SalesLT/vProductAndDescription`
  - `SalesLT/vProductModelCatalogDescription`
- The main Bronze loop must make one isolated write attempt per listed source object, even if some earlier object fails.
- At least one execution cell must include a concrete write path pattern under `Tables/bronze/` for discoverability.
- The emitted Bronze code MUST contain an actual `spark.read...` statement for one of the listed tables and an actual `df.write.format("delta").mode("overwrite").option("overwriteSchema","true").save(...)` statement under `Tables/bronze/...`; helper-only code is insufficient.
- The generator MUST emit at least 4 runnable Bronze code cells: setup, imports/helpers, execution loop, summary/assertion. Zero-cell, one-cell, or prose-only output is invalid.
- If the notebook generator cannot infer a non-essential implementation detail, it must still emit the full Bronze notebook and fail inside the relevant code cell with a clear runtime error; returning no cells is prohibited.
- The final Bronze summary cell must explicitly assert all of the following:
  - the `results` structure contains at least one attempted table,
  - at least one table was physically written under `Tables/bronze/`,
  - at least one table passed the post-write discoverability check at its exact canonical path.
- Do not defer Bronze implementation to later layers, helper libraries, or notebook attachments. All runnable Bronze ingestion logic for this run must appear directly in the emitted notebook cells.
- Required per-table result fields in the emitted code:
  - `table`
  - `flat_table_name`
  - `output_path`
  - `attempted`
  - `write_succeeded`
  - `discoverable`
  - `rows`
  - `status`
  - `error`
- Required final Bronze self-audit:
  - Build a list of expected canonical Bronze paths from the 13 input objects.
  - Build a list of actually discoverable successful Bronze paths from the per-table results.
  - Raise if the discoverable list is empty, even if write attempts were made.
  - Include both lists in the final summary JSON so downstream failures are diagnosable.

## Silver
- Standardize all Bronze tables into snake_case column names and write to `Tables/silver/<table>`.
- Add Silver metadata columns:
  - `_run_id`
  - `_silver_ts`
  - `_source_dt` from `modified_date` when available, else typed null/current timestamp fallback per rules
  - `_is_current` default true for this full-refresh build
- Remove exact duplicate rows and apply deterministic deduplication by business key, keeping the latest `modified_date` where present.

Per-table Silver intent and dedup keys:
- `address`
  - Dedup key: `address_id`
  - Clean text fields: trim address lines, city, state_province, country_region, postal_code
  - Keep location fields used later for customer geography and maps.
- `customer`
  - Dedup key: `customer_id`
  - Clean person/company text fields, normalize email casing, preserve `sales_person`
  - Derive helper full-name fields only if useful, but retain raw name components.
- `customeraddress`
  - Dedup key: `(customer_id, address_id, address_type)`
  - Keep as a conformed support table in Silver only.
  - NOTE: user requested Gold customer to combine Customer and Address without using CustomerAddress in between. Because no direct address key exists on `Customer`, Gold `dim_customer` cannot physically combine address fields without some linkage rule. The only relational path available in the provided schema is through `CustomerAddress`. Implement Gold per user intent as closely as possible by selecting one “primary” customer address from `CustomerAddress` and not exposing the bridge itself in Gold. If the user wants a different address-selection rule, edit this spec.
- `product`
  - Dedup key: `product_id`
  - Derive helper flags in dependency order:
    - `is_discontinued` from `discontinued_date`
    - `is_currently_sellable` using sell dates and `is_discontinued`
  - Exclude binary `thumb_nail_photo` from Silver if not needed downstream to reduce model size; keep filename for reference.
- `productcategory`
  - Dedup key: `product_category_id`
  - Preserve parent-child structure via `parent_product_category_id`.
- `productdescription`
  - Dedup key: `product_description_id`
- `productmodel`
  - Dedup key: `product_model_id`
- `productmodelproductdescription`
  - Dedup key: `(product_model_id, product_description_id, culture)`
  - Clean `culture`, retain for English filter in Gold.
- `salesorderheader`
  - Dedup key: `sales_order_id`
  - Standardize date/timestamp fields.
  - Keep order-level attributes used for order dimension and fact assembly: order dates, ship method, status, flags, addresses, customer, monetary totals.
  - Derive `order_month_start`, `ship_month_start`, and date keys for Gold joins.
- `salesorderdetail`
  - Dedup key: `(sales_order_id, sales_order_detail_id)`
  - Preserve numeric precision for `unit_price`, `unit_price_discount`, `line_total`.
  - Derive `discount_pct_line = unit_price_discount / nullif(unit_price,0)` where meaningful.
- `vgetallcategories`
  - Dedup key: `product_category_id`
  - Useful optional helper only; Gold product dimension can instead use base `productcategory`.
- `vproductanddescription`
  - Dedup key: `(product_id, culture)`
  - Useful optional cross-check source for product/description joins; not primary Gold source because user asked for base-table composition.
- `vproductmodelcatalogdescription`
  - Dedup key: `product_model_id`
  - Keep as optional enrichment helper for future product analytics; not required in user-requested Gold star.

Silver optimization:
- OPTIMIZE large Silver tables after write:
  - `salesorderheader`
  - `salesorderdetail`
  - final Gold fact table
- Z-ORDER candidates where supported:
  - `sales_order_id`, `customer_id`, `product_id`, `order_date`
- Ensure every post-rename transform revalidates column existence before subsequent joins/windows.

## Gold
Build the user-requested sales reporting star schema in `Tables/gold/` with only relevant analytics fields retained.

Modeling note driven by actual schema:
- `SalesOrderHeader` + `SalesOrderDetail` clearly form the transactional fact grain.
- `Customer`, `Address`, `Product`, `ProductCategory`, `ProductDescription`, `ProductModel`, and `ProductModelProductDescription` are descriptive/master data sources.
- `CustomerAddress` and `ProductModelProductDescription` behave as bridge/junction tables.
- User intent is followed exactly where possible; where schema forces an implementation detail, it is called out below.

Gold tables:
- `dim_order_date`
- `dim_ship_date`
- `dim_customer`
- `dim_salesperson`
- `dim_order`
- `dim_product`
- `fact_sales_order`

### dim_order_date
- Source: `silver.salesorderheader`
- Grain: one row per calendar date from `order_date`
- Key: `order_date_key` as `yyyyMMdd` integer; alternate actual date column `order_date`
- Attributes:
  - year, quarter, month number, month name, month short, year-month, week, day, day name, is_month_end
- Hierarchy:
  - Year > Quarter > Month > Date
- Required because user explicitly asked for an OrderDate date dimension.

### dim_ship_date
- Source: `silver.salesorderheader`
- Grain: one row per distinct non-null `ship_date`
- Key: `ship_date_key` as `yyyyMMdd` integer; alternate actual date column `ship_date`
- Attributes:
  - same calendar breakdown as order date
- Hierarchy:
  - Year > Quarter > Month > Date
- Include unknown row/key for orders with null ship date.

### dim_customer
- Primary source: `silver.customer`
- Geography enrichment: `silver.customeraddress` + `silver.address`
- User intent says “Combine Customer and Address. Don’t use CustomerAddress in between.” Actual schema does not provide address keys on `Customer`, so implement Gold `dim_customer` by using `customeraddress` only as hidden transformation logic to assign one relevant address per customer, then do not expose the bridge table in Gold.
- Address selection rule for one-row-per-customer dimension:
  - Prefer `address_type = 'Main Office'`
  - Else `address_type = 'Home'`
  - Else alphabetically first `address_type`
  - Tie-break latest `modified_date`, then lowest `address_id`
- Grain: one row per customer
- Key: `customer_key` surrogate or durable integer based on `customer_id`; keep `customer_id` as business key
- Relevant fields:
  - customer name components / display name
  - company_name
  - email_address
  - phone
  - sales_person_key (derived from cleaned salesperson)
  - city, state_province, country_region, postal_code
  - selected_address_type
- Hierarchies:
  - Geography: CountryRegion > StateProvince > City > PostalCode
  - Customer name hierarchy if useful: CompanyName or LastName > FirstName
- Exclude passwords and salts from Gold entirely.

### dim_salesperson
- Source: distinct `sales_person` values from `silver.customer`
- User rule:
  - If stored as `<domain>/username` or `domain\username`, keep username only.
  - Example requested: `adventure-works\jillian0` becomes `Jillian`
- Standardized transformation:
  - split on `\` first, else `/`
  - take final token
  - trim
  - proper-case for display name
  - keep raw source string as audit column if desired
- Grain: one row per distinct cleaned salesperson
- Key: `salesperson_key`
- Fields:
  - cleaned username/display name
  - raw_sales_person
  - optional domain if parsable
- Hierarchy:
  - Domain > Salesperson display name when domain exists
- NOTE: salespeople are only available through customer assignments in the provided schema, so discount analysis by salesperson will attribute order lines to the salesperson associated with the ordering customer.

### dim_order
- Source: `silver.salesorderheader`
- Purpose: move non-measure, low-cardinality order attributes out of the fact per user request.
- Grain: one row per sales order
- Key: `order_key` or business key `sales_order_id`
- Relevant attributes:
  - sales_order_number
  - purchase_order_number
  - account_number
  - revision_number
  - status
  - online_order_flag
  - ship_method
  - credit_card_approval_code presence flag rather than sensitive detail where appropriate
  - bill_to_address_id
  - ship_to_address_id
  - comment presence/length classification rather than free-text if model size matters
- Hierarchies:
  - Status grouping hierarchy if implemented: Order Channel (online/offline) > Status
- Keep order-level monetary columns out of this dimension if they are used as additive/semi-additive analytics in the fact.

### dim_product
- Primary sources:
  - `silver.product`
  - `silver.productcategory`
  - `silver.productmodel`
  - `silver.productmodelproductdescription` filtered to `culture = 'en'`
  - `silver.productdescription`
- User-required composition:
  - combine Product
  - include ProductCategory parent-child flattened to category + subcategory
  - include ProductDescription `description`
  - include ProductModel `name` as `model_name`
  - use ProductModelProductDescription only for English (`culture = 'en'`) link/filtering
- Category logic based on actual parent-child schema:
  - self-join `productcategory` to flatten:
    - parent category name as `category_name`
    - child category name as `subcategory_name`
  - if product category has no parent, place name in `category_name` and null in `subcategory_name`
- Description logic:
  - preferred path per requested tables: `product.product_model_id` -> `productmodelproductdescription.product_model_id` (culture = 'en') -> `productdescription.product_description_id`
- Grain: one row per product
- Key: `product_key` or business key `product_id`
- Relevant fields:
  - product_id, product name, product_number
  - color, size, weight
  - standard_cost, list_price
  - model_name
  - description
  - category_name, subcategory_name
  - sell_start_date, sell_end_date, discontinued_date
  - flags such as `is_discontinued`, `is_currently_sellable`
- Hierarchies:
  - Category > Subcategory > Product
  - ModelName > Product
- Exclude binary image payloads from Gold.

### fact_sales_order
- Source: `silver.salesorderdetail` joined to `silver.salesorderheader`, then linked to Gold dimensions
- Grain: one row per sales order detail line
- Business key: `(sales_order_id, sales_order_detail_id)`
- Foreign keys:
  - `order_key`
  - `product_key`
  - `customer_key`
  - `salesperson_key`
  - `order_date_key`
  - `ship_date_key`
- Retained degenerate/order identifiers if useful:
  - `sales_order_id`
  - `sales_order_detail_id`
- Measures / analytics columns:
  - `order_qty`
  - `unit_price`
  - `unit_price_discount`
  - `line_total`
  - header-level `sub_total`, `tax_amt`, `freight`, `total_due`
- Allocation note:
  - Because `sub_total`, `tax_amt`, `freight`, and `total_due` are header-grain amounts, either:
    - keep them only for reference and avoid summing at line grain, or
    - allocate them proportionally to line amount into `allocated_tax_amt`, `allocated_freight`, `allocated_total_due`.
  - Recommended: include allocated variants for additive reporting and keep raw header values clearly marked as non-additive.
- Derived fact metrics:
  - `gross_line_amount = unit_price * order_qty`
  - `discount_amount = unit_price_discount * order_qty` if discount is per-unit in source interpretation
  - `discount_pct_line`
- Regional performance comes from the customer geography attached via `dim_customer`.
- Average and maximum sales metrics will be implemented primarily as semantic measures rather than persisted columns.

## Test
Write all test outcomes to `Tables/test/test_results` with schema:
`run_id, test_name, layer, table_name, status, actual, expected, details, checked_at`

Standard tests to run after Gold writes:
- `row_count_bronze_vs_silver_<table>`
  - Layer: bronze/silver
  - For each base transactional/master table, compare row counts Bronze vs Silver.
  - Expectation: Silver row count within ~1% of Bronze for non-dedup-heavy tables; where dedup removes more, log explicit explanation in details.
- `gold_dim_pk_not_null_<dim>`
  - Layer: gold
  - For each dimension (`dim_order_date`, `dim_ship_date`, `dim_customer`, `dim_salesperson`, `dim_order`, `dim_product`), assert primary/business key used in the semantic model is never null.
- `gold_dim_pk_unique_<dim>`
  - Layer: gold
  - For each dimension, assert key uniqueness.
- `gold_fact_fk_integrity_<fact>_<dim>`
  - Layer: gold
  - Assert every non-null fact foreign key resolves to its parent dimension:
    - fact_sales_order -> dim_order_date
    - fact_sales_order -> dim_ship_date (excluding unknown/null handling rows)
    - fact_sales_order -> dim_customer
    - fact_sales_order -> dim_salesperson
    - fact_sales_order -> dim_order
    - fact_sales_order -> dim_product
- `business_rule_sales_amounts_non_negative`
  - Layer: gold
  - Assert `order_qty > 0`, `unit_price >= 0`, `line_total >= 0`, and `discount_pct_line` is within a sensible range (for example 0 to 1.0 unless source proves otherwise).
- `business_rule_regional_coverage_present`
  - Layer: gold
  - Assert at least one non-null `country_region` and one non-null `state_province` or `city` exists in `dim_customer` to support requested map visuals.
- `business_rule_salesperson_discount_presence`
  - Layer: gold
  - Assert there is at least one discounted line (`unit_price_discount > 0`) and at least one non-null salesperson assignment so the “largest discounts by salesperson” report page is meaningful.
- `business_rule_shipdate_unknown_row_exists`
  - Layer: gold
  - Assert `dim_ship_date` contains unknown row for null ship dates if nulls are present in header source.

Test execution behavior:
- Each test appends exactly one result row to `test_results`.
- Status values:
  - PASS when rule is satisfied
  - FAIL when assertion evaluated and failed
  - ERROR when test execution itself crashed
- Include concise JSON or scalar text in `actual`, `expected`, and `details`.
- Raise after tests if any FAIL/ERROR occurred so pipeline surfaces quality issues.

## Semantic model
Create a Direct Lake semantic model over Gold with star schema relationships centered on `fact_sales_order`.

Tables to expose:
- `fact_sales_order`
- `dim_order_date`
- `dim_ship_date`
- `dim_customer`
- `dim_salesperson`
- `dim_order`
- `dim_product`

Relationships:
- `fact_sales_order[order_date_key]` -> `dim_order_date[order_date_key]` (many-to-one, single direction)
- `fact_sales_order[ship_date_key]` -> `dim_ship_date[ship_date_key]`
- `fact_sales_order[customer_key]` -> `dim_customer[customer_key]`
- `fact_sales_order[salesperson_key]` -> `dim_salesperson[salesperson_key]`
- `fact_sales_order[order_key]` -> `dim_order[order_key]`
- `fact_sales_order[product_key]` -> `dim_product[product_key]`

Hierarchies to create:
- Order Date: Year > Quarter > Month > Date
- Ship Date: Year > Quarter > Month > Date
- Customer Geography: CountryRegion > StateProvince > City > PostalCode
- Product: CategoryName > SubcategoryName > Product Name
- Salesperson: Domain > Salesperson Display Name where domain exists
- Optional Order hierarchy: OnlineOrderFlag > Status > ShipMethod

Explicit measures:
- `Sales Amount = SUM(fact_sales_order[line_total])`
- `Average Sales Amount = AVERAGE(fact_sales_order[line_total])`
- `Max Sales Amount = MAX(fact_sales_order[line_total])`
- `Order Count = DISTINCTCOUNT(fact_sales_order[sales_order_id])`
- `Units Sold = SUM(fact_sales_order[order_qty])`
- `Discount Amount = SUM(fact_sales_order[discount_amount])`
- `Average Discount Amount = AVERAGE(fact_sales_order[discount_amount])`
- `Max Discount Amount = MAX(fact_sales_order[discount_amount])`
- `Discount % = DIVIDE([Discount Amount], SUMX(fact_sales_order, fact_sales_order[gross_line_amount]))`
- `Average Discount % = AVERAGEX(fact_sales_order, fact_sales_order[discount_pct_line])`
- `Max Discount % = MAX(fact_sales_order[discount_pct_line])`
- `Average Order Value = DIVIDE([Sales Amount], [Order Count])`
- `Monthly Sales Trend = [Sales Amount]` for time-intelligence visuals using `dim_order_date`
- Optional additive measure if allocated fields are built:
  - `Allocated Total Due = SUM(fact_sales_order[allocated_total_due])`

Modeling notes:
- Hide surrogate keys and raw audit columns from report authors.
- Mark both date dimensions as date tables if Fabric/Power BI supports dual active roles cleanly; otherwise keep one active relationship and expose the second explicitly for ship-date visuals.
- Format currency and percentage measures appropriately.
- Add descriptions to tables, columns, and measures emphasizing regional performance, discount analytics, and order trends.

## Report
Create a Power BI report tailored to the requested sales reporting solution. Always include a Data quality page.

### Page 1 — Regional Performance
- Filled map or bubble map using:
  - Location: `dim_customer[country_region]`, `state_province]`, and/or `city`
  - Size/Color: `Sales Amount`
- Complementary visual: bar chart of Sales Amount by Country/State
- KPI cards:
  - `Sales Amount`
  - `Average Sales Amount`
  - `Max Sales Amount`
  - `Order Count`
- Matrix:
  - Geography hierarchy rows
  - Product hierarchy columns optional
  - Values: Sales Amount, Average Sales Amount, Max Sales Amount
- Slicers:
  - Order Date hierarchy
  - Ship Date hierarchy
  - Salesperson
  - Product Category

### Page 2 — Orders & Monthly Trends
- Line chart:
  - Axis: `dim_order_date[Year-Month]`
  - Values: `Sales Amount`, optional `Order Count`
- Clustered column chart:
  - Orders by `dim_order[status]` or `ship_method`
- Table or matrix:
  - Sales order number, customer, ship method, status, Sales Amount, Units Sold
- KPI cards:
  - `Average Order Value`
  - `Units Sold`
  - `Average Sales Amount`
  - `Max Sales Amount`
- Slicers:
  - Online order flag
  - Ship method
  - Category/Subcategory

### Page 3 — Discounts & Salesperson Performance
- Bar chart:
  - Axis: `dim_salesperson[salesperson_display_name]`
  - Values: `Discount Amount`
  - Sort descending to identify largest discounts
- Scatter chart:
  - X: `Average Discount %`
  - Y: `Sales Amount`
  - Size: `Order Count`
  - Details: Salesperson
- Matrix:
  - Salesperson hierarchy rows
  - Product hierarchy columns
  - Values: Discount Amount, Discount %, Average Discount %, Max Discount %
- KPI cards:
  - `Discount Amount`
  - `Discount %`
  - `Average Discount %`
  - `Max Discount %`

### Page 4 — Product Performance
- Treemap:
  - Group: Category > Subcategory
  - Values: Sales Amount
- Bar chart:
  - Top products by Sales Amount
- Table:
  - Product, Model Name, Category, Subcategory, Sales Amount, Units Sold, Average Sales Amount, Max Sales Amount

### Page 5 — Data Quality
- Table visual bound to `test_results`
- Cards:
  - total tests
  - passed tests
  - failed tests
  - errored tests
- Bar or donut chart by status
- Table filtered to latest `run_id`
- Optional detail slicers:
  - layer
  - table_name
  - status

Report authoring notes:
- Use consistent currency/percent formatting.
- Ensure map visuals degrade gracefully if only country/state-level geography is populated.
- Include drill-down on all configured hierarchies.
- Add tooltip pages for geography and salesperson to show average/max sales and discount metrics.

## Data Agent
Create an AI Skill grounded on the semantic model for business users exploring sales, regional performance, orders, products, and discounts.

Role:
- You are a Sales Performance Analytics Agent for the SalesLT reporting model. You help users understand regional sales performance, monthly trends, orders, customer geography, product performance, and salesperson discount behavior using the certified semantic model only.

Domain hints:
- The core business process is sales order line analysis.
- Regional analysis comes from customer-linked geography.
- Salesperson analysis comes from the cleaned salesperson dimension derived from customer assignments.
- Date analysis can use both order date and ship date.
- Product analysis supports category, subcategory, model, and product detail.
- Discount analysis should use the semantic measures, especially Discount Amount and Discount %.

Starter questions:
- Which regions have the highest and lowest sales?
- Show monthly sales trends for the last 12 months.
- What are the average and maximum sales by country and state?
- Which salespeople are offering the largest discounts?
- What is the average discount percentage by salesperson?
- Which product categories drive the most revenue?
- How many orders were placed by month and by ship method?
- Which customers or regions have the highest average order value?
- Compare sales by order date versus ship date.
- Which subcategories have high discounts but low sales?

Guardrails:
- Answer only from the grounded semantic model; do not invent data or external business context.
- Prefer certified measures over raw column aggregation when both exist.
- Clarify whether the user means order date or ship date when time context is ambiguous.
- Clarify whether “region” means country, state/province, or city when needed.
- Do not expose hidden technical columns, audit fields, password-related fields, or salts.
- Treat header-grain totals carefully; prefer line-grain sales measures and allocated totals when available.
- If a requested metric is not in the model, say so and suggest the closest existing measure.
- Keep responses concise, numeric, and comparison-oriented where possible.

Agent instructions:
- Interpret business terms flexibly:
  - “sales” usually means `Sales Amount`
  - “average sales” means `Average Sales Amount`
  - “maximum sales” means `Max Sales Amount`
  - “discount rate” means `Discount %`
- When users ask for “best” or “worst” regions, rank by `Sales Amount` unless they specify another metric.
- When users ask about discounting behavior, include both absolute `Discount Amount` and relative `Discount %` where possible.
- Use hierarchies for drill-down suggestions, e.g., Country > State > City or Category > Subcategory > Product.
- Encourage follow-up comparisons such as time period, region, category, and salesperson.
- If map-style regional insight is requested, return geography-grain summaries suitable for map visuals.
- Always mention filters applied in the answer.
- For ambiguous questions, ask one clarifying question instead of guessing.
- Maintain a professional, direct, insight-oriented tone: like an excellent analytics copilot that helps users find performance drivers fast.