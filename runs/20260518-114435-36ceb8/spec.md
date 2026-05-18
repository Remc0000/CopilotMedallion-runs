# Run Spec 20260518-114225-f7c474

## Updated specs

### Iteration 1 — 2026-05-18 11:46:37Z — failed layer: bronze (run: 20260518-114435-36ceb8)
- **Root cause (1-line summary)**: Bronze completed without producing discoverable Delta outputs under `Tables/bronze/`, so Silver refused to start.
- **Cross-table audit**: Address: yes — Bronze discoverability failure is table-agnostic and can affect any source if path naming/writes are wrong; Customer: yes — same write-path/discoverability risk; CustomerAddress: yes — same risk; Product: yes — same risk; ProductCategory: yes — same risk; ProductDescription: yes — same risk; ProductModel: yes — same risk; ProductModelProductDescription: yes — same risk; SalesOrderDetail: yes — same risk; SalesOrderHeader: yes — same risk; vGetAllCategories: yes — same risk; vProductAndDescription: yes — same risk; vProductModelCatalogDescription: yes — same risk.
- **Fix approach**: GENERALIZE — the failure is systemic across all Bronze source objects because discoverability depends on a uniform write-path and successful per-table Delta save, not on one table’s business schema.
- **What was changed**:
  - Tightened **Generic guidance** with an explicit discoverability contract: Bronze must write to physical lakehouse paths under `Tables/bronze/<flat_table_name>` and verify `_delta_log` exists after each save.
  - Tightened **Bronze** with exact flat-name mapping, exact target path pattern, required derivation of `_ingested_date`, and mandatory post-write verification plus final non-zero output assertion.
  - Clarified that Bronze must not rely on metastore/table registration for downstream discovery; path-based Delta outputs are the source of truth.

### Iteration 2 — 2026-05-18 11:49:10Z — failed layer: bronze (run: 20260518-114435-36ceb8)
- **Root cause (1-line summary)**: Bronze notebook allowed a per-table statement failure to cancel the Spark session before other tables could be attempted or before actionable table-level diagnostics were persisted.
- **Cross-table audit**: Address: yes — any read/write/assert failure in its iteration can cancel the shared session; Customer: yes — same loop/session risk; CustomerAddress: yes — same; Product: yes — same; ProductCategory: yes — same; ProductDescription: yes — same; ProductModel: yes — same; ProductModelProductDescription: yes — same; SalesOrderDetail: yes — same; SalesOrderHeader: yes — same; vGetAllCategories: yes — same; vProductAndDescription: yes — same; vProductModelCatalogDescription: yes — same.
- **Fix approach**: GENERALIZE — the failure is systemic across every Bronze source object because session cancellation is caused by notebook control-flow and action placement, not by one table’s business schema.
- **What was changed**:
  - Tightened **Generic guidance** with a mandatory Bronze two-phase execution pattern: preflight validation for all source objects first, then isolated per-table read/write/verify processing, with no single statement spanning multiple tables.
  - Tightened **Bronze** to require table-level `try/except`, immediate error persistence, deferred notebook failure until after the loop, and explicit prohibition on pre-write actions (`count`, full scans, verification reads) before a successful save.
  - Added mandatory final summary semantics: attempted/succeeded/failed tables must be recorded, and the notebook must raise once at the end if any required table failed.

### Iteration 3 — 2026-05-18 11:54:27Z — failed layer: bronze (run: 20260518-114435-36ceb8)
- **Root cause (1-line summary)**: Silver again found zero discoverable Bronze outputs, which means the Bronze notebook must enforce a stricter physical target-base-path contract instead of only validating relative `Tables/bronze/...` strings.
- **Cross-table audit**: Address: yes — if the resolved lakehouse base path is wrong, address will also be written outside discoverable Bronze storage; Customer: yes — same base-path risk; CustomerAddress: yes — same; Product: yes — same; ProductCategory: yes — same; ProductDescription: yes — same; ProductModel: yes — same; ProductModelProductDescription: yes — same; SalesOrderDetail: yes — same; SalesOrderHeader: yes — same; vGetAllCategories: yes — same; vProductAndDescription: yes — same; vProductModelCatalogDescription: yes — same.
- **Fix approach**: GENERALIZE — this is a systemic path-resolution/discoverability issue affecting every Bronze object uniformly, so one stricter rule for all source tables is safer than table-specific fixes.
- **What was changed**:
  - Tightened **Generic guidance** with a mandatory resolved-path rule: every layer write must use the target lakehouse ABFSS/base path plus `Tables/<layer>/<flat_table_name>`, and the notebook must fail preflight if the resolved base path is missing, blank, or does not belong to the target lakehouse.
  - Tightened **Bronze** to require exact computation and logging of `bronze_root_path` and per-table `target_path`, explicit prohibition on relative-only saves like `Tables/bronze/...`, and strict post-write verification against the same fully resolved path.
  - Added a required notebook-start self-check that the target lakehouse root is writable/discoverable before processing any source table.

## Inputs
- Workspace: `e47657b4-6180-42ac-8c4c-779c0e480cc3`
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
- Target Lakehouse: **demolake**

## Generic guidance
Apply these reference skills/agents at all times:
- FabricDataEngineer agent: https://github.com/microsoft/skills-for-fabric/blob/main/agents/FabricDataEngineer.agent.md
- e2e-medallion-architecture skill: https://github.com/microsoft/skills-for-fabric/tree/main/skills/e2e-medallion-architecture
- spark-authoring-cli skill: https://github.com/microsoft/skills-for-fabric/tree/main/skills/spark-authoring-cli
- powerbi-authoring-cli skill: https://github.com/microsoft/skills-for-fabric/tree/main/skills/powerbi-authoring-cli
- powerbi-consumption-cli skill: https://github.com/microsoft/skills-for-fabric/tree/main/skills/powerbi-consumption-cli
- powerbi-semantic-model-authoring: https://github.com/RuiRomano/powerbi-agentic-plugins/tree/main/plugins/powerbi/skills/powerbi-semantic-model-authoring
- powerbi-report-authoring: https://github.com/RuiRomano/powerbi-agentic-plugins/tree/main/plugins/powerbi/skills/powerbi-report-authoring

Cross-cutting code rules for every generated notebook:
- Use defensive column references everywhere; never assume a column still exists after a projection, rename, drop, or join.
- After every join, immediately project/select alias-prefixed columns into flat output names; do not carry ambiguous duplicate names forward.
- Before every groupBy/agg, assert the existence of every grouping and aggregation input column.
- For Fabric/REST calls, use defensive handling: `if x is None: raise ...` BEFORE any `.get()` access.
- Do not use `saveAsTable`; write Delta outputs by path under `Tables/<layer>/...`.
- Use parameter cells at the top of each notebook for workspace, lakehouse, paths, run_id, and configurable behavior.
- Make writes idempotent with overwrite patterns and `option('overwriteSchema','true')` where appropriate.
- Use error-loud `try/except`; call `_save_error(layer, e)` (or `_save_error(layer, e, table=tbl)` inside loops) and re-raise.
- Process tables per-table in isolated loops so one table failure does not silently mask others.
- Always emit discoverable Delta tables to the target lakehouse layer path and raise if zero outputs were written.
- Every code cell must start with a short markdown-style comment block using Python comments (`# ---` plus 1-3 explanatory lines).

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
- ALWAYS process source tables in a `for tbl in source_tables:` loop where each iteration is a SELF-CONTAINED unit: read → transform → write → record-result → recover.
- For Bronze specifically, do NOT re-raise inside the per-table `except` block. Persist the failure with `_save_error('bronze', e, table=tbl)`, append the failure details to a results dict/list, continue to the next table, and raise ONCE after the loop finishes if any required tables failed.
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

Rule L — Discoverability requires physical Delta path verification, not just successful write calls.
- A successful `.save(...)` call is not sufficient evidence that downstream layer discovery will work. After each Bronze write, immediately verify the target directory exists under the target lakehouse path and contains a `_delta_log` child.
- Record a table as "written" in the summary dict ONLY after both conditions are true:
  - the target path is exactly `.../Tables/bronze/<flat_table_name>`
  - `_delta_log` exists beneath that path
- Do not rely on temp views, catalog registration, display(), or row-count-only logging as proof of Bronze success. Downstream discovery is path-based.

Rule M — Bronze execution ordering must minimize session-wide cancellation risk.
- Bronze must use a TWO-PHASE pattern:
  - Phase 1: lightweight preflight validation for ALL configured source objects and ALL target path strings, with no writes and no expensive actions.
  - Phase 2: per-table isolated processing for only the validated tables.
- In preflight, validate only:
  - source object name/path is non-empty
  - target path is non-empty and contains `/Tables/bronze/`
  - source object exists in the source lakehouse namespace
- Do NOT run `count()`, wide `display()`, schema-inference side computations, verification re-reads, or any other expensive action during preflight.
- Inside each table iteration, the allowed sequence is:
  - read source table/view
  - append metadata columns
  - save to Delta target path
  - verify path and `_delta_log`
  - optionally re-read persisted Delta path for row count
  - record success
- Do NOT perform row counts on the source DataFrame before the write; if a source object is problematic, that pre-write action can fail before any output is landed and can cancel the session without partial success.

Rule N — Bronze final error contract must be auditable.
- Maintain a results structure with, at minimum, `attempted`, `succeeded`, `failed`, `target_path`, and `error_message` by table.
- At notebook end, print a single JSON summary that includes:
  - all attempted source tables
  - all successful writes with exact target paths
  - all failures with the failing table name and concise error text
- If any required Bronze table failed, raise exactly one final exception AFTER printing the summary, and include the failed table names in the exception message.
- This preserves partial-success diagnostics while still failing the run correctly.

Rule O — Resolved lakehouse base path must be explicit and validated before any layer write.
- Every write path must be built from a resolved target lakehouse base path variable such as `target_lakehouse_base_path` or `tgt_base` plus the relative Delta location. The notebook must never call `.save("Tables/bronze/...")`, `.save("/Tables/bronze/...")`, or any other relative-only path because those can land outside the discoverable lakehouse storage context.
- Before any Bronze processing begins, resolve and validate:
  - workspace id / lakehouse id / lakehouse name inputs are non-empty
  - the resolved base path string is non-empty
  - the resolved base path corresponds to the target lakehouse `demolake`
  - the computed Bronze root is exactly `<resolved_base>/Tables/bronze`
- Log the resolved base path and Bronze root once at notebook start so a failed run shows the exact physical target used.
- If the notebook cannot resolve a trustworthy target lakehouse base path, fail in preflight before touching any source table.

Rule P — Discoverability self-check must verify the target lakehouse root, not only per-table outputs.
- At notebook start, perform a lightweight writeability/discoverability self-check against the resolved target lakehouse root context before entering the per-table loop. The purpose is to catch bad path resolution early.
- Acceptable pattern: verify the resolved Bronze root parent exists or can be listed in the target lakehouse filesystem namespace; if this cannot be confirmed, fail preflight.
- Do not substitute a temp/local filesystem path for the target lakehouse path. Only the target lakehouse physical path counts as discoverable output.

ALSO REQUIRE for every generated notebook: EACH code cell must start with a short markdown comment block (Python `# ---` divider + 1-3 lines of `# ` comments) describing what the cell is doing and why — never emit a cell with no leading comment. Example:
```
# ---
# Read raw bronze table and apply schema enforcement.
# Drops rows where any required key is null.
df = spark.read.table(...)
```
Keep the comments human-readable, not the code repeated in prose. The goal: someone scrolling through the notebook in Fabric can understand each block at a glance.

## Bronze
- Ingest every listed source object into `Tables/bronze/<flat_table_name>` using lowercased final path segments and this exact mapping:
  - `SalesLT/Address` -> `Tables/bronze/address`
  - `SalesLT/Customer` -> `Tables/bronze/customer`
  - `SalesLT/CustomerAddress` -> `Tables/bronze/customeraddress`
  - `SalesLT/Product` -> `Tables/bronze/product`
  - `SalesLT/ProductCategory` -> `Tables/bronze/productcategory`
  - `SalesLT/ProductDescription` -> `Tables/bronze/productdescription`
  - `SalesLT/ProductModel` -> `Tables/bronze/productmodel`
  - `SalesLT/ProductModelProductDescription` -> `Tables/bronze/productmodelproductdescription`
  - `SalesLT/SalesOrderDetail` -> `Tables/bronze/salesorderdetail`
  - `SalesLT/SalesOrderHeader` -> `Tables/bronze/salesorderheader`
  - `SalesLT/vGetAllCategories` -> `Tables/bronze/vgetallcategories`
  - `SalesLT/vProductAndDescription` -> `Tables/bronze/vproductanddescription`
  - `SalesLT/vProductModelCatalogDescription` -> `Tables/bronze/vproductmodelcatalogdescription`
- Bronze MUST execute in two phases:
  - Phase 1: preflight validation over all configured source objects and their computed target paths
  - Phase 2: isolated per-table processing only after preflight has built the list of valid work items
- Bronze MUST resolve a single explicit target lakehouse base path at notebook start and use it for every table write.
  - Required variables:
    - `target_lakehouse_base_path`: fully resolved physical base path for lakehouse `demolake`
    - `bronze_root_path = f"{target_lakehouse_base_path}/Tables/bronze"`
  - Every per-table target must be computed as:
    - `target_path = f"{bronze_root_path}/{flat_table_name}"`
  - The notebook MUST log `target_lakehouse_base_path`, `bronze_root_path`, and each `target_path`.
  - Do NOT call `.save()` with only `Tables/bronze/<flat_table_name>` or any other relative-only string.
- Read each source table independently in a per-table loop; do not combine source reads in Bronze.
- Preserve source schema as-is in Bronze, but add standard metadata columns:
  - `_run_id`
  - `_ingested_at`
  - `_source_table`
  - `_bronze_loaded_at`
  - `_ingested_date` derived from `_ingested_at` as a plain date column; this column must be physically present before any partitioned write
- For source objects that already include `ModifiedDate`, keep it untouched; do not overwrite business audit columns.
- Partitioning intent:
  - For large transactional tables `salesorderheader` and `salesorderdetail`, partition Bronze by `_ingested_date`.
  - For small/master/view tables, write unpartitioned unless the notebook generator enforces `_ingested_date` partitioning uniformly across all Bronze tables.
- Write mode:
  - Full overwrite per run, `format('delta').mode('overwrite').option('overwriteSchema','true')`.
- Bronze path contract:
  - Build the write target from the resolved target lakehouse base path plus the exact relative path `Tables/bronze/<flat_table_name>`.
  - Do not write Bronze outputs anywhere else (not `Files/`, not `Tables/Bronze/`, not mixed-case names, not nested source-system folders).
  - Do not write to relative notebook filesystem paths.
  - Do not rely on metastore registration or `saveAsTable`; downstream discovery must succeed by scanning the physical `Tables/bronze/` folder under the target lakehouse path.
- Bronze per-table control flow is REQUIRED:
  - Wrap each table iteration in its own `try/except`.
  - On failure, call `_save_error('bronze', e, table=<source_table>)`, record the failure in the summary structure, and CONTINUE to the next table.
  - Do not `raise` from inside the per-table loop.
  - After the loop ends, raise once if any required source table failed.
- Bronze preflight validation is REQUIRED before any write:
  - Assert the source object exists before read.
  - Assert the resolved `target_lakehouse_base_path` string is non-empty.
  - Assert `bronze_root_path` is non-empty and contains `/Tables/bronze`.
  - Assert every computed per-table `target_path` starts with the resolved `bronze_root_path` and ends with the expected flat table name.
  - Build the complete list of work items first so target-path bugs are caught before writes begin.
  - Fail preflight immediately if the target lakehouse base path cannot be resolved to `demolake`.
- Bronze target-root self-check is REQUIRED before the ingestion loop:
  - Verify the target lakehouse filesystem namespace for `bronze_root_path` (or its parent under the resolved lakehouse path) is listable/reachable.
  - If that check fails, stop the notebook before reading any source table.
- Bronze action-ordering constraints:
  - Do not call `count()` on the source DataFrame before saving it.
  - Do not call `display()` or other eager actions on the source DataFrame inside the ingestion loop.
  - The first expensive action in a table iteration should be the write to the Delta target path.
  - If row count is needed for logging, obtain it only by re-reading the persisted Delta target path after `_delta_log` verification.
- Bronze post-write verification is REQUIRED for every table:
  - After `save(target_path)`, verify the directory for that exact fully resolved `target_path` exists.
  - Verify `<target_path>/_delta_log` exists.
  - Re-read the just-written Delta path from that same exact fully resolved `target_path` and capture row count from the persisted path, not from a pre-write DataFrame only.
  - Add the table to the success summary dict only after those checks pass.
  - If any check fails, treat the table as failed and call `_save_error('bronze', e, table=<source_table>)`.
- Bronze validation:
  - Record for every attempted table: source name, flat table name, exact fully resolved target path, status, and concise error text if failed.
  - Print a final JSON summary containing attempted, succeeded, and failed Bronze tables and exact paths for successes.
  - Raise if zero Bronze tables were successfully verified as discoverable.
  - Raise if any required source table for downstream Gold modeling (`customer`, `customeraddress`, `address`, `product`, `productcategory`, `productmodel`, `productdescription`, `productmodelproductdescription`, `salesorderheader`, `salesorderdetail`) failed to land.
  - The final raised exception must name the failed required tables explicitly.
- Notes for views:
  - `vgetallcategories`, `vproductanddescription`, and `vproductmodelcatalogdescription` are still landed to Bronze for traceability, but Gold should prefer base tables where the user explicitly requested those joins.
  - Because view schemas can omit audit fields, Bronze must not try to synthesize source-side business audit logic for them.

## Silver
- Standardize all Bronze tables to snake_case column names and write to `Tables/silver/<table>`.
- Add Silver audit fields to every table:
  - `_silver_loaded_at`
  - `_source_dt` using `modified_date` where available, otherwise typed null or current timestamp per the global rules
  - `_is_current` = true for the deduped retained row
- Drop duplicate records with deterministic keys based on actual schema:
  - `address`: dedupe on `address_id`, keep latest `modified_date`
  - `customer`: dedupe on `customer_id`, keep latest `modified_date`
  - `customeraddress`: dedupe on composite `(customer_id, address_id, address_type)`, keep latest `modified_date`
  - `product`: dedupe on `product_id`, keep latest `modified_date`
  - `productcategory`: dedupe on `product_category_id`, keep latest `modified_date`
  - `productdescription`: dedupe on `product_description_id`, keep latest `modified_date`
  - `productmodel`: dedupe on `product_model_id`, keep latest `modified_date`
  - `productmodelproductdescription`: dedupe on composite `(product_model_id, product_description_id, culture)`, keep latest `modified_date`
  - `salesorderdetail`: dedupe on `sales_order_detail_id`; also validate `sales_order_id` exists because Gold fact joins require it
  - `salesorderheader`: dedupe on `sales_order_id`, keep latest `modified_date`
  - `vgetallcategories`: dedupe on `product_category_id`
  - `vproductanddescription`: dedupe on composite `(product_id, culture, description)`
  - `vproductmodelcatalogdescription`: dedupe on `product_model_id`, keep latest `modified_date`
- Silver cleaning rules by table intent:
  - `customer`:
    - trim string fields
    - preserve `sales_person`, `email_address`, `company_name`, name parts, phone
    - do not expose `password_hash` and `password_salt` to Gold; they may remain in raw Silver only if governance requires traceability, but preferred Silver output for business modeling excludes them from curated downstream datasets
  - `address`:
    - trim address strings
    - standardize blank strings to null for `address_line2`
  - `product`:
    - derive helper flags before dropping anything:
      - `is_discontinued` from `discontinued_date`
      - `is_sellable_currently` based on `sell_start_date`, `sell_end_date`, and `is_discontinued`
    - retain pricing and category/model references
    - keep binary thumbnail in Silver only if required; exclude from Gold dimensions to avoid model bloat
  - `productcategory`:
    - preserve both `product_category_id` and `parent_product_category_id` for the required self-join in Gold
  - `salesorderheader`:
    - standardize order dates
    - preserve `customer_id`, `ship_to_address_id`, `bill_to_address_id`, `ship_method`, `status`, `online_order_flag`, monetary fields, and `comment`
    - NOTE: the user requested salesperson from `salesorderheader`, but the actual `SalesOrderHeader` schema does not contain a `SalesPerson` column. The available `sales_person` field exists in `customer`. Gold will therefore source salesperson from `customer.sales_person` via the `salesorderheader.customer_id -> customer.customer_id` join unless the user edits the spec.
  - `salesorderdetail`:
    - validate non-null `sales_order_id`, `sales_order_detail_id`, and `product_id` before Gold joins
    - preserve quantity, unit price, discount, and line total
- Silver output intent:
  - Keep one curated Silver table per source object.
  - OPTIMIZE / V-Order intent after writes for larger tables: `salesorderheader`, `salesorderdetail`, `product`, `customer`.
- Sensitive-data handling:
  - Exclude `password_hash` and `password_salt` from Gold and from the semantic model.
  - If Silver retains them in a raw-curated table, clearly mark them as non-business and do not propagate downstream.

## Gold
- Gold star schema must satisfy the user’s requested entities:
  - `dim_customer`
  - `dim_product`
  - `dim_sales`
  - `fact_sales`

- `dim_customer`
  - Base grain: one row per customer-address relationship, because the user explicitly requested joining customer, customer address, and address while keeping meaningful fields.
  - Join path:
    - `customer.customer_id = customeraddress.customer_id`
    - `customeraddress.address_id = address.address_id`
  - Include meaningful descriptive fields from actual schema:
    - customer identifiers and customer master data
    - full customer name assembled from title/first/middle/last/suffix where present
    - company_name, email_address, phone, name_style
    - customer sales_person from `customer.sales_person`
    - address_type
    - address_line1, address_line2, city, state_province, country_region, postal_code
  - Recommended Gold keying:
    - surrogate/business key candidate `customer_address_key` built deterministically from `customer_id`, `address_id`, and `address_type`
    - also keep `customer_id` and `address_id` as business keys for drill-through and relationship support
  - Modeling note:
    - This design can create multiple rows per customer if a customer has multiple addresses. That matches the user’s requested join. If the user later wants one row per customer only, split address out into a separate `dim_address` and keep `dim_customer` at pure customer grain.

- `dim_product`
  - Base grain: one row per product.
  - Required join path from user request:
    - `product.product_model_id = productmodel.product_model_id`
    - `product.product_category_id = productcategory.product_category_id`
    - self-join `productcategory.parent_product_category_id -> parent.product_category_id`
    - `productmodel.product_model_id = productmodelproductdescription.product_model_id`
    - `productmodelproductdescription.product_description_id = productdescription.product_description_id`
  - Include meaningful fields from actual schema:
    - product_id, product name, product_number, color, size, weight
    - standard_cost, list_price
    - sell_start_date, sell_end_date, discontinued_date
    - derived flags such as `is_discontinued`, `is_sellable_currently`
    - product_model attributes: product model name, catalog_description
    - product description text and culture from the bridge
    - category and subcategory:
      - `subcategory_name` from the child `productcategory.name`
      - `category_name` from the parent `productcategory.name`
  - Self-join requirement:
    - Implement the product category self-join so the final dimension exposes both parent category and subcategory exactly as requested.
  - Grain caution:
    - `productmodelproductdescription` may create multiple rows per product when multiple `culture` values exist. Preferred default is:
      - retain one row per `(product_id, culture)` in `dim_product`, because culture is an actual attribute in the source
    - Alternative the user may edit to later:
      - filter Gold to `culture = 'en'` if a single-language product dimension is preferred.
  - Exclude large binary fields such as `thumbnail_photo` from Gold.

- `dim_sales`
  - Purpose: descriptive salesperson dimension for slicing sales.
  - NOTE: requested source mismatch exists. `SalesOrderHeader` does not contain `SalesPerson`; the only actual source column is `customer.sales_person`.
  - Build logic:
    - derive salesperson values by joining `salesorderheader.customer_id` to `customer.customer_id`
    - select distinct cleaned salesperson strings
  - Cleaning rule required by user:
    - remove `adventure-works\` or `adventureworks\` style prefix if present
    - remove trailing numeric suffix at the end of the name if present
    - trim whitespace
  - Suggested output columns:
    - `sales_key` surrogate key
    - `sales_person_raw`
    - `sales_person_clean`
    - optional parsed helper fields if derivable safely from the cleaned text
  - If cleaned salesperson is null/blank, keep an `Unknown` member for referential integrity.

- `fact_sales`
  - Base grain: one row per sales order detail line.
  - Required join path from user request:
    - `salesorderdetail.sales_order_id = salesorderheader.sales_order_id`
  - Retain relevant factual and degenerate fields from actual schema:
    - line grain keys: `sales_order_id`, `sales_order_detail_id`
    - foreign keys: `product_id`, `customer_id`, `ship_to_address_id`, `bill_to_address_id`
    - salesperson foreign key via cleaned salesperson join to `dim_sales`
    - dates: `order_date`, `due_date`, `ship_date`
    - order descriptors: `sales_order_number`, `purchase_order_number`, `account_number`, `ship_method`, `status`, `online_order_flag`
    - additive measures: `order_qty`, `unit_price`, `unit_price_discount`, `line_total`
    - header-level monetary fields retained for analysis with care: `sub_total`, `tax_amt`, `freight`, `total_due`
  - Recommended dimensional relationships:
    - `fact_sales.product_id` (or product surrogate) -> `dim_product`
    - `fact_sales.customer_address_key` -> `dim_customer` if customer-address grain is adopted
    - `fact_sales.sales_key` -> `dim_sales`
  - Important grain note:
    - Because `dim_customer` is customer-address grain while `fact_sales` has both bill-to and ship-to address IDs, choose one conformed relationship strategy:
      - Preferred default: create `fact_sales.customer_address_key_ship` from `(customer_id, ship_to_address_id)` and use that as the active relationship to `dim_customer`; keep bill-to fields as descriptive columns.
      - Alternative user-editable option: create separate `dim_customer_shipto` and `dim_customer_billto` role-playing dimensions if both address roles must be sliced independently.
  - Header money caution:
    - `sub_total`, `tax_amt`, `freight`, and `total_due` are header-level amounts repeated across detail lines after the join. For semantic measures, aggregate them carefully using distinct-order logic, not simple sum across lines.

## Test
- Write all test outcomes to `Tables/test/test_results` with schema:
  - `run_id, test_name, layer, table_name, status, actual, expected, details, checked_at`
- Each test appends exactly one result row per table/test combination.

- Standard test 1: row counts per layer
  - Compare Bronze vs Silver row counts for each landed source table.
  - Expected: Silver row count should be approximately Bronze row count within 1% for stable master tables; if dedup occurs, allow Silver to be less than Bronze with details logged.
  - For large transactional tables, also compare joined Gold `fact_sales` row count to Silver `salesorderdetail`; expected equality or near equality depending on rejected null-key rows.

- Standard test 2: no-null PKs in gold dims
  - `dim_customer`: no null `customer_address_key`; also no null `customer_id`
  - `dim_product`: no null `product_id` (or product surrogate if created)
  - `dim_sales`: no null `sales_key` and no null `sales_person_clean`

- Standard test 3: unique PKs in gold dims
  - `dim_customer`: uniqueness of `customer_address_key`
  - `dim_product`: uniqueness of chosen grain key:
    - if one row per `(product_id, culture)`, test uniqueness on that composite business grain or a deterministic surrogate
  - `dim_sales`: uniqueness of `sales_key` and uniqueness of `sales_person_clean` after standardization

- Standard test 4: referential integrity
  - Every `fact_sales.product_id` must exist in `dim_product`
  - Every populated `fact_sales.sales_key` must exist in `dim_sales`
  - Every active `fact_sales.customer_address_key_ship` must exist in `dim_customer`
  - Log separate RI results for each FK relationship

- Standard test 5: tailored business-rule sanity checks
  - `fact_sales.line_total >= 0`
  - `fact_sales.order_qty > 0`
  - `fact_sales.unit_price >= 0`
  - `salesorderheader.total_due >= salesorderheader.sub_total` at order level when both exist
  - `dim_product.category_name` should be populated whenever `subcategory_name` is populated, validating the category self-join
  - `dim_sales.sales_person_clean` should not start with `adventureworks` or end in trailing digits after cleaning

- Additional useful checks
  - For `dim_customer`, ensure at least one of `company_name` or customer full name is populated
  - For `fact_sales`, validate `order_date <= due_date` when both are non-null
  - For shipped orders, validate `ship_date >= order_date` when both are non-null

## Semantic model
- Mode: Direct Lake over Gold tables in `demolake`.
- Primary star schema:
  - `fact_sales`
  - `dim_product`
  - `dim_customer`
  - `dim_sales`
- Relationships:
  - `fact_sales` many-to-one -> `dim_product`
  - `fact_sales` many-to-one -> `dim_customer` using the chosen ship-to customer-address key as active relationship
  - `fact_sales` many-to-one -> `dim_sales`
- If implemented, expose `order_date`, `due_date`, and `ship_date` as date columns in fact; a dedicated date dimension can be added later, but it is not required by the user request.
- Hide technical columns from report authors:
  - raw rowguid values
  - load timestamps
  - password-related fields
  - binary/photo columns
  - raw helper dedup columns
- Explicit measures tailored to actual data:
  - `Sales Amount = SUM(fact_sales[line_total])`
  - `Order Quantity = SUM(fact_sales[order_qty])`
  - `Average Unit Price = AVERAGE(fact_sales[unit_price])`
  - `Discount Amount = SUMX(fact_sales, fact_sales[unit_price] * fact_sales[order_qty] * fact_sales[unit_price_discount])`
  - `Order Count = DISTINCTCOUNT(fact_sales[sales_order_id])`
  - `Customer Count = DISTINCTCOUNT(fact_sales[customer_id])`
  - `Avg Sales per Order = DIVIDE([Sales Amount], [Order Count])`
  - `Header Total Due = SUMX(VALUES(fact_sales[sales_order_id]), MAX(fact_sales[total_due]))`
- Useful dimension attributes for slicing:
  - `dim_product`: category_name, subcategory_name, product name, color, size, culture, is_sellable_currently
  - `dim_customer`: country_region, state_province, city, address_type, company_name, customer full name
  - `dim_sales`: sales_person_clean
  - `fact_sales`: online_order_flag, ship_method, status
- Semantic notes:
  - Because header amounts repeat at detail grain, report authors should use `Header Total Due` rather than summing `total_due`.
  - If `dim_product` is kept at `(product_id, culture)` grain, ensure the relationship key in fact uses the corresponding chosen product grain strategy; otherwise filter to a single culture in Gold.

## Report
- Build a Power BI report on top of the Direct Lake semantic model.
- Page 1: Sales Overview
  - KPI cards: `Sales Amount`, `Order Count`, `Order Quantity`, `Customer Count`
  - Line chart: `Sales Amount` by `order_date`
  - Clustered column chart: `Sales Amount` by `category_name`
  - Bar chart: `Sales Amount` by `sales_person_clean`
  - Matrix: category_name > subcategory_name > product name with `Sales Amount` and `Order Quantity`

- Page 2: Customer & Geography
  - Map or filled map: `Sales Amount` by `country_region` / `state_province`
  - Bar chart: top customers by `Sales Amount` using company_name or customer full name
  - Stacked bar: `Sales Amount` by `address_type`
  - Table: customer, city, state_province, sales_person_clean, order count, sales amount

- Page 3: Product Performance
  - Tree map: `Sales Amount` by category/subcategory
  - Scatter chart: `Average Unit Price` vs `Order Quantity` by product
  - Bar chart: top products by `Sales Amount`
  - Slicer set: category_name, subcategory_name, color, size, is_sellable_currently, culture

- Page 4: Data quality
  - Table visual over `test_results` with status highlighting
  - Cards: count of PASS / FAIL / ERROR
  - Matrix: tests by layer and table_name
  - Detail table showing `actual`, `expected`, `details`, `checked_at`
- Report behavior:
  - Include slicers for date, salesperson, category, country, and online_order_flag.
  - Use tooltips to show order count and average unit price on sales visuals.
  - Default filters should exclude blank/unknown helper members only where that does not hide data quality issues; the Data quality page must show all results.

## Data Agent
- Agent type: AISkill grounded on the semantic model.
- Role:
  - "You are a sales analytics assistant for the SalesLT Direct Lake model. Help business users understand sales performance by product, customer geography, and salesperson, using only the curated Gold model and published measures."

- Domain hints:
  - `fact_sales` is line-level sales data joined from sales order header and detail.
  - `dim_product` contains product, model, category, subcategory, and description attributes.
  - `dim_customer` combines customer master data with customer-address attributes.
  - `dim_sales` contains cleaned salesperson names derived from customer sales_person values because SalesOrderHeader does not expose a salesperson column in the actual schema.
  - Header totals must use the dedicated measure, not a raw sum of repeated line values.

- Starter questions:
  - Which product categories generate the highest sales amount?
  - Who are the top salespeople by sales amount and order count?
  - Which customers contribute the most revenue?
  - How do sales break down by country, state, and city?
  - What are the top subcategories by order quantity?
  - Are online orders different from offline orders in sales amount or order count?
  - Which products have high sales but low average unit price?
  - Are there any products marked sellable currently that have no recent sales?
  - What data quality tests failed in the latest run?
  - How does header total due compare with summed line sales by order?

- Guardrails:
  - Use only the published semantic model tables, relationships, and measures.
  - Prefer explicit measures (`Sales Amount`, `Order Count`, `Header Total Due`, etc.) over raw column aggregation.
  - Do not reference Bronze or Silver tables.
  - Do not expose or reason over password fields, salts, hashes, raw rowguid values, or binary image data.
  - When discussing salesperson, explain that the cleaned salesperson comes from `customer.sales_person` because `salesorderheader` lacks that field in the actual schema.
  - If a question requires unsupported logic such as profitability beyond available cost/sales assumptions, state the limitation clearly.
  - If ambiguity exists because of the customer-address grain or product culture grain, state which grain is being used in the answer.