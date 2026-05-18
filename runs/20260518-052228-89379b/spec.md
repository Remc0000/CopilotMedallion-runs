# Run Spec 20260518-051803-a256b5

## Updated specs

### Iteration 1 — 2026-05-18 05:24:57Z — failed layer: bronze (run: 20260518-052228-89379b)
- **Root cause (1-line summary)**: Bronze completed without producing discoverable Delta tables under the `bronze` schema path/name convention, so Silver could not find any upstream inputs.
- **Cross-table audit**: `SalesLT/Address`: yes — Bronze discovery failure would block it like all other sources because this is a table-registration/path-convention issue, not a source-specific shape issue; `SalesLT/Customer`: yes — same reason; `SalesLT/CustomerAddress`: yes — same reason; `SalesLT/Product`: yes — same reason; `SalesLT/ProductCategory`: yes — same reason; `SalesLT/ProductDescription`: yes — same reason; `SalesLT/ProductModel`: yes — same reason; `SalesLT/ProductModelProductDescription`: yes — same reason; `SalesLT/SalesOrderDetail`: yes — same reason; `SalesLT/SalesOrderHeader`: yes — same reason; `SalesLT/vGetAllCategories`: yes — same reason; `SalesLT/vProductAndDescription`: yes — same reason; `SalesLT/vProductModelCatalogDescription`: yes — same reason.
- **Fix approach**: GENERALIZE — the root cause is systemic across every Bronze source object because the next layer depends on a uniform bronze schema/table discovery pattern.
- **What was changed**:
  - Tightened **Generic guidance** with mandatory Bronze discoverability checks and required catalog/path parity for every written table.
  - Tightened **Bronze** to require creation of the `bronze` schema, exact table names, exact physical layout under `Tables/bronze/<table_name>`, and post-write verification that all expected Bronze tables are discoverable before the layer is considered successful.
  - Added an explicit Bronze failure rule: if any expected Bronze table is missing after write/registration, raise and fail Bronze rather than allowing Silver to start.

### Iteration 2 — 2026-05-18 05:27:12Z — failed layer: bronze (run: 20260518-052228-89379b)
- **Root cause (1-line summary)**: Bronze notebook generation failed because the LLM returned zero notebook cells, so no executable content existed for the layer.
- **Cross-table audit**: `SalesLT/Address`: yes — would be skipped entirely if no cells are produced; `SalesLT/Customer`: yes — same reason; `SalesLT/CustomerAddress`: yes — same reason; `SalesLT/Product`: yes — same reason; `SalesLT/ProductCategory`: yes — same reason; `SalesLT/ProductDescription`: yes — same reason; `SalesLT/ProductModel`: yes — same reason; `SalesLT/ProductModelProductDescription`: yes — same reason; `SalesLT/SalesOrderDetail`: yes — same reason; `SalesLT/SalesOrderHeader`: yes — same reason; `SalesLT/vGetAllCategories`: yes — same reason; `SalesLT/vProductAndDescription`: yes — same reason; `SalesLT/vProductModelCatalogDescription`: yes — same reason.
- **Fix approach**: GENERALIZE — this is a notebook-generation contract failure that can affect every Bronze source object uniformly, so the spec now forces a minimum executable notebook structure for the entire Bronze layer.
- **What was changed**:
  - Tightened **Generic guidance** to require that every generated notebook contain executable code cells and forbids markdown-only or empty notebook output.
  - Tightened **Bronze** with a mandatory minimum cell plan, requiring at least one executable Bronze orchestration notebook that reads all listed inputs, writes all expected outputs, and performs verification.
  - Added an explicit no-empty-notebook rule: if notebook generation would otherwise return zero cells, the generator must still emit the required Bronze scaffold and fail only at runtime with a clear error if a source read/write check fails.

### Iteration 3 — 2026-05-18 05:31:41Z — failed layer: bronze (run: 20260518-052228-89379b)
- **Root cause (1-line summary)**: Bronze still did not create discoverable managed tables under the exact `Tables/bronze/<table_name>` convention, so the next run again had no catalog-visible Bronze inputs.
- **Cross-table audit**: `SalesLT/Address`: yes — same managed-table/discoverability requirement applies; `SalesLT/Customer`: yes — same; `SalesLT/CustomerAddress`: yes — same; `SalesLT/Product`: yes — same; `SalesLT/ProductCategory`: yes — same; `SalesLT/ProductDescription`: yes — same; `SalesLT/ProductModel`: yes — same; `SalesLT/ProductModelProductDescription`: yes — same; `SalesLT/SalesOrderDetail`: yes — same; `SalesLT/SalesOrderHeader`: yes — same; `SalesLT/vGetAllCategories`: yes — same; `SalesLT/vProductAndDescription`: yes — same; `SalesLT/vProductModelCatalogDescription`: yes — same.
- **Fix approach**: GENERALIZE — the failure is systemic across all Bronze outputs, so the spec now mandates one exact managed-table write/verification contract for every Bronze object rather than per-table exceptions.
- **What was changed**:
  - Tightened **Generic guidance** with a stricter managed-table-only rule for Bronze and an explicit ban on path-only/orphan writes.
  - Tightened **Bronze** to require exact source-to-target table names, exact schema-qualified registration as `bronze.<table_name>`, exact physical folder expectation `Tables/bronze/<table_name>`, and explicit verification using catalog/listing plus read-back of each table.
  - Added a mandatory end-of-layer assertion that the discoverable Bronze set count must equal 13 exactly before Bronze is considered successful.

### Iteration 4 — 2026-05-18 05:35:37Z — failed layer: bronze (run: 20260518-052228-89379b)
- **Root cause (1-line summary)**: Bronze session was cancelled after one or more statements failed, with no layer-level guardrail forcing per-table fail-fast diagnostics and isolated validation before continuing the loop.
- **Cross-table audit**: `SalesLT/Address`: yes — any read/write/count failure inside the Bronze loop could cancel the session if not trapped and surfaced immediately; `SalesLT/Customer`: yes — same; `SalesLT/CustomerAddress`: yes — same; `SalesLT/Product`: yes — same; `SalesLT/ProductCategory`: yes — same; `SalesLT/ProductDescription`: yes — same; `SalesLT/ProductModel`: yes — same; `SalesLT/ProductModelProductDescription`: yes — same; `SalesLT/SalesOrderDetail`: yes — same; `SalesLT/SalesOrderHeader`: yes — same; `SalesLT/vGetAllCategories`: yes — same; `SalesLT/vProductAndDescription`: yes — same; `SalesLT/vProductModelCatalogDescription`: yes — same.
- **Fix approach**: GENERALIZE — the failure mode is systemic across all Bronze source objects because any unguarded failed statement in the orchestration loop can abort the whole Spark session regardless of table.
- **What was changed**:
  - Tightened **Generic guidance** to require statement-level fail-fast checks, explicit source read validation, and immediate abort on first Bronze object failure with the exact source object and target table named.
  - Tightened **Bronze** to require a deterministic per-table processing sequence (read, validate, enrich, write, read-back verify) with no continuation after a failed table.
  - Added explicit Bronze diagnostics requirements so the notebook surfaces the failing statement/table clearly before session cancellation, rather than masking the root cause behind a generic cancelled-session error.

### Iteration 5 — 2026-05-18 05:38:52Z — failed layer: bronze (run: 20260518-052228-89379b)
- **Root cause (1-line summary)**: Bronze still hit a generic cancelled-session error, indicating the notebook needs stricter statement minimization and a single canonical source-read/write pattern that avoids unsupported or redundant actions across all source objects.
- **Cross-table audit**: `SalesLT/Address`: yes — can fail if read or write uses a non-canonical object reference or extra validation statements; `SalesLT/Customer`: yes — same; `SalesLT/CustomerAddress`: yes — same; `SalesLT/Product`: yes — same; `SalesLT/ProductCategory`: yes — same; `SalesLT/ProductDescription`: yes — same; `SalesLT/ProductModel`: yes — same; `SalesLT/ProductModelProductDescription`: yes — same; `SalesLT/SalesOrderDetail`: yes — same; `SalesLT/SalesOrderHeader`: yes — same; `SalesLT/vGetAllCategories`: yes — same, especially because it may be a view-like object; `SalesLT/vProductAndDescription`: yes — same, especially because it may be a view-like object; `SalesLT/vProductModelCatalogDescription`: yes — same, especially because it may be a view-like object.
- **Fix approach**: GENERALIZE — the available failure signal is session-wide and non-table-specific, so the safest fix is one uniform Bronze execution contract for all 13 source objects rather than speculative per-table exceptions.
- **What was changed**:
  - Tightened **Generic guidance** to require a single canonical read method for source objects, a single canonical managed-table write method for Bronze, and to ban non-essential Spark actions during Bronze.
  - Tightened **Bronze** with an exact implementation contract: resolve source objects only as `SalesLT.<object_name>`, use one write/replacement pattern per table, and reduce validation to one read-back plus one count per table.
  - Added explicit prohibitions on fallback read strategies, exploratory statements, and multiple write attempts for the same table within one run.

### Iteration 6 — 2026-05-18 05:45:51Z — failed layer: bronze (run: 20260518-052228-89379b)
- **Root cause (1-line summary)**: Downstream generation still could not discover any Bronze tables, so Bronze must now enforce a single exact managed-table creation pattern plus an explicit schema-qualified read-back gate before the layer is considered complete.
- **Cross-table audit**: `SalesLT/Address`: yes — same discoverability risk because downstream expects `bronze.address`; `SalesLT/Customer`: yes — same; `SalesLT/CustomerAddress`: yes — same; `SalesLT/Product`: yes — same; `SalesLT/ProductCategory`: yes — same; `SalesLT/ProductDescription`: yes — same; `SalesLT/ProductModel`: yes — same; `SalesLT/ProductModelProductDescription`: yes — same; `SalesLT/SalesOrderDetail`: yes — same; `SalesLT/SalesOrderHeader`: yes — same; `SalesLT/vGetAllCategories`: yes — same, including if written as a path-only object or temp view; `SalesLT/vProductAndDescription`: yes — same; `SalesLT/vProductModelCatalogDescription`: yes — same.
- **Fix approach**: GENERALIZE — the root cause is systemic across all 13 Bronze objects because the failure is about target table registration/discoverability, not one source table’s schema.
- **What was changed**:
  - Tightened **Generic guidance** with a hard rule that Bronze success requires `spark.table("bronze.<table_name>")` read-back success for every expected output, not just files or listing hints.
  - Tightened **Bronze** to require one exact schema-qualified SQL write contract per table (`CREATE OR REPLACE TABLE bronze.<table_name> AS SELECT ...`) and to forbid path-only writes, temp-view-only outputs, or relying on inferred registration.
  - Added an explicit end-of-layer gate that must validate the full expected set of 13 `bronze.<table_name>` objects by exact name before Silver generation is allowed.

## Inputs
- Workspace: `2127a578-6ec2-447e-b8a9-94868408b064`
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
- Target Lakehouse: **e2egp54**

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
- Use defensive column references everywhere; before selecting, renaming, deriving, joining, filtering, aggregating, or windowing, assert required columns exist on the DataFrame in that exact step.
- After every join, immediately project columns with explicit aliases/prefixes so downstream logic never depends on ambiguous duplicate names.
- Before every `groupBy` / `agg`, assert the grouping and aggregated columns exist; fail fast with a clear error naming the missing column(s) and DataFrame variable.
- For REST/API responses, use defensive handling: `if x is None: raise ...` BEFORE any `.get()` access.
- Do not use `saveAsTable`; write using path/table patterns supported for Lakehouse notebooks and keep writes idempotent.
- Every notebook must start with parameter cells for workspace, source lakehouse, target lakehouse, schema, table names, and run id so the same notebook can be rerun safely.
- Use idempotent overwrite/upsert patterns by layer; Bronze can overwrite full landing per table, Silver should overwrite curated tables deterministically, Gold should overwrite dimensional/fact outputs deterministically.
- Use error-loud `try/except` blocks around major read/transform/write units; in `except`, call `_save_error(layer, e)` and then `raise`.
- Never silently skip missing required inputs; if an expected table or column is absent, raise with a specific message.
- Avoid hard-coding schema assumptions beyond the actual source columns discovered at runtime.
- Treat the three `v*` objects as source tables/views and validate whether they add business value versus duplicating derivations already achievable from base tables.
- Bronze/Silver discoverability rule: a layer is only considered successful if its outputs are both physically written and discoverable by Spark catalog/listing in the target Lakehouse using the expected schema-qualified names. For Bronze specifically, Silver must be able to discover the outputs as `bronze.<table_name>` objects; if discovery fails for any expected table, raise in Bronze and stop the run.
- Catalog/path parity rule: when writing managed Lakehouse tables, ensure the registered table name and the physical folder layout align with the schema/table convention expected by downstream layers. Do not write orphaned Delta paths without a discoverable table object, and do not register a table under a different schema/name than the medallion spec.
- Managed Bronze write rule: for Bronze, the required outcome is a managed, catalog-visible table object in schema `bronze` for every listed source. Path-only writes, temp views, in-memory DataFrames, or files written outside the managed Lakehouse `Tables/bronze/<table_name>` convention do not satisfy the spec even if Delta files exist.
- Post-write verification rule: after each table write, immediately verify the table exists in the target schema and is queryable; after the full Bronze loop, verify the complete expected Bronze set is present before allowing downstream generation.
- Bronze read-back gate rule: for every required Bronze output, the verification must include a successful schema-qualified read using the exact object name `spark.table("bronze.<table_name>")` (or equivalent `SELECT * FROM bronze.<table_name> LIMIT 1`). A file-path existence check, a temporary view, or a schema listing alone is insufficient.
- Bronze managed-table creation rule: use one exact managed-table creation pattern that both writes data and registers the target object under the final schema-qualified name in a single step. Prefer `CREATE OR REPLACE TABLE bronze.<table_name> AS SELECT ...` from a prepared temp view/DataFrame projection. Do not rely on a separate later registration step to make a prior path write discoverable.
- Notebook generation contract rule: every layer notebook MUST contain executable code cells. Do not return an empty notebook, a markdown-only notebook, or a placeholder plan with no Python/Spark cells.
- Minimum executable notebook rule: if the generator is uncertain, it must still emit a runnable scaffold with parameter initialization, helper functions, source table list, a loop over the required objects for the failed layer, write logic, and final validation. Runtime validation may fail loudly, but notebook generation itself must not yield zero cells.
- Cell-count safety rule: every generated notebook must contain multiple cells, including at least one executable setup cell and at least one executable transform/write cell. For Bronze specifically, the notebook must include enough executable cells to read source objects, write target tables, and verify discoverability.
- Statement-failure containment rule: for loop-driven Bronze ingestion, wrap EACH source object in its own `try/except` block and abort the notebook on the first failed object after logging the exact `source_object`, `target_table`, and failed phase (`read`, `metadata_enrichment`, `write`, `read_back_validation`, or `count_validation`). Do not continue to later tables after one object fails.
- Pre-action validation rule: before any expensive write or count action, validate that the DataFrame variable exists, is not `None`, and contains the columns required for the immediate next step; if not, raise a targeted error before issuing the Spark action.
- Bronze diagnostic rule: emit progress logs before and after each major statement in the form `START <phase> <source_object> -> <target_table>` and `SUCCESS <phase> <source_object> -> <target_table>` so a cancelled session can be traced back to the last attempted statement.
- Bronze canonical I/O rule: in Bronze, use exactly one canonical source read form per object and one canonical managed-table write form per target. Do not attempt multiple alternate read syntaxes, fallback path probes, repeated overwrite attempts, or mixed write APIs for the same table within a single run.
- Bronze action minimization rule: in Bronze, avoid any non-essential Spark actions beyond the required per-table write, one read-back queryability check, and one count validation. Do not add `display()`, `show()`, `collect()`, repeated `count()`, schema-profiling scans, or trial queries during the ingestion loop.
- Bronze source reference rule: when reading the listed inputs, resolve each source using the exact object name implied by the input list (`SalesLT/<ObjectName>` -> Spark object `SalesLT.<ObjectName>`). Do not invent alternate schema names, snake_case source names, or path-based fallbacks for the initial source read unless the exact object read has already failed and the notebook is raising that explicit error.

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

ALSO REQUIRE for every generated notebook: EACH code cell must start with a short markdown comment block (Python `# ---` divider + 1-3 lines of `# ` comments) describing what the cell is doing and why — never emit a cell with no leading comment. Example:
```
# ---
# Read raw bronze table and apply schema enforcement.
# Drops rows where any required key is null.
df = spark.read.table(...)
```
Keep the comments human-readable, not the code repeated in prose. The goal: someone scrolling through the notebook in Fabric can understand each block at a glance.

## Bronze
Land each selected source object into the `bronze` schema as a raw-but-queryable Delta table with minimal transformation plus technical metadata.

### Bronze landing pattern
- One bronze table per source object, preserving source granularity and original business meaning.
- Recommended bronze names:
  - `bronze.address`
  - `bronze.customer`
  - `bronze.customer_address`
  - `bronze.product`
  - `bronze.product_category`
  - `bronze.product_description`
  - `bronze.product_model`
  - `bronze.product_model_product_description`
  - `bronze.sales_order_detail`
  - `bronze.sales_order_header`
  - `bronze.v_get_all_categories`
  - `bronze.v_product_and_description`
  - `bronze.v_product_model_catalog_description`
- The Bronze notebook MUST be emitted as an executable notebook, not a narrative plan. It must contain Python/Spark cells that can run end-to-end in Fabric.
- Minimum Bronze notebook structure is REQUIRED and must contain at least these executable units:
  - parameter/setup cell
  - imports + helper functions cell
  - schema/table mapping cell for all 13 source objects
  - source read + bronze write loop cell
  - post-write verification/completeness assertion cell
- The Bronze notebook MUST create/ensure the `bronze` schema exists in the target Lakehouse before writing any table.
- For each source object above, the write outcome MUST be a discoverable schema-qualified table with the exact name listed above; do not write only to an unmanaged/orphan path and assume downstream discovery will infer it.
- Physical organization MUST align with the Lakehouse table convention expected by downstream discovery: each Bronze output must land under the target Lakehouse managed tables area for schema `bronze` and table `<table_name>` so that listing/querying `bronze.<table_name>` succeeds immediately after the write.
- Bronze writes are REQUIRED to use one canonical target mapping only; do not derive alternative folder names or schema names:
  - `SalesLT/Address` -> schema `bronze`, table `address`, expected object `bronze.address`, expected managed folder `Tables/bronze/address`
  - `SalesLT/Customer` -> schema `bronze`, table `customer`, expected object `bronze.customer`, expected managed folder `Tables/bronze/customer`
  - `SalesLT/CustomerAddress` -> schema `bronze`, table `customer_address`, expected object `bronze.customer_address`, expected managed folder `Tables/bronze/customer_address`
  - `SalesLT/Product` -> schema `bronze`, table `product`, expected object `bronze.product`, expected managed folder `Tables/bronze/product`
  - `SalesLT/ProductCategory` -> schema `bronze`, table `product_category`, expected object `bronze.product_category`, expected managed folder `Tables/bronze/product_category`
  - `SalesLT/ProductDescription` -> schema `bronze`, table `product_description`, expected object `bronze.product_description`, expected managed folder `Tables/bronze/product_description`
  - `SalesLT/ProductModel` -> schema `bronze`, table `product_model`, expected object `bronze.product_model`, expected managed folder `Tables/bronze/product_model`
  - `SalesLT/ProductModelProductDescription` -> schema `bronze`, table `product_model_product_description`, expected object `bronze.product_model_product_description`, expected managed folder `Tables/bronze/product_model_product_description`
  - `SalesLT/SalesOrderDetail` -> schema `bronze`, table `sales_order_detail`, expected object `bronze.sales_order_detail`, expected managed folder `Tables/bronze/sales_order_detail`
  - `SalesLT/SalesOrderHeader` -> schema `bronze`, table `sales_order_header`, expected object `bronze.sales_order_header`, expected managed folder `Tables/bronze/sales_order_header`
  - `SalesLT/vGetAllCategories` -> schema `bronze`, table `v_get_all_categories`, expected object `bronze.v_get_all_categories`, expected managed folder `Tables/bronze/v_get_all_categories`
  - `SalesLT/vProductAndDescription` -> schema `bronze`, table `v_product_and_description`, expected object `bronze.v_product_and_description`, expected managed folder `Tables/bronze/v_product_and_description`
  - `SalesLT/vProductModelCatalogDescription` -> schema `bronze`, table `v_product_model_catalog_description`, expected object `bronze.v_product_model_catalog_description`, expected managed folder `Tables/bronze/v_product_model_catalog_description`
- The Bronze code MUST explicitly iterate over the full required object list above; do not omit any object due to uncertainty.
- Source object resolution for the initial read is mandatory and exact:
  - `SalesLT/Address` must be read as `SalesLT.Address`
  - `SalesLT/Customer` must be read as `SalesLT.Customer`
  - `SalesLT/CustomerAddress` must be read as `SalesLT.CustomerAddress`
  - `SalesLT/Product` must be read as `SalesLT.Product`
  - `SalesLT/ProductCategory` must be read as `SalesLT.ProductCategory`
  - `SalesLT/ProductDescription` must be read as `SalesLT.ProductDescription`
  - `SalesLT/ProductModel` must be read as `SalesLT.ProductModel`
  - `SalesLT/ProductModelProductDescription` must be read as `SalesLT.ProductModelProductDescription`
  - `SalesLT/SalesOrderDetail` must be read as `SalesLT.SalesOrderDetail`
  - `SalesLT/SalesOrderHeader` must be read as `SalesLT.SalesOrderHeader`
  - `SalesLT/vGetAllCategories` must be read as `SalesLT.vGetAllCategories`
  - `SalesLT/vProductAndDescription` must be read as `SalesLT.vProductAndDescription`
  - `SalesLT/vProductModelCatalogDescription` must be read as `SalesLT.vProductModelCatalogDescription`
- Do not snake_case or otherwise rewrite the source object names for the read step. Snake_case applies only to the Bronze target table names listed above.
- The Bronze loop MUST use exactly one read attempt per source object via Spark table resolution and exactly one managed-table create/replace attempt per target table. Do not implement fallback reads from ad hoc file paths, `Tables/...` guesses, alternate schemas, or multiple write retries inside the same notebook pass.
- The required Bronze write contract is:
  - read `SalesLT.<ObjectName>` into a DataFrame
  - add the required Bronze technical columns
  - register that prepared DataFrame as a temp view for the current object
  - execute `CREATE OR REPLACE TABLE bronze.<table_name> AS SELECT * FROM <prepared_temp_view>`
- Do not satisfy Bronze with `df.write.format("delta").save(...)`, `insertInto`, temp-view-only output, or any pattern that depends on a later registration step to make the table discoverable.
- The Bronze loop MUST process each source object in this exact per-table order: (1) log START read, (2) read source object, (3) assert DataFrame is non-null and schema is non-empty, (4) add Bronze technical columns, (5) create or replace `bronze.<table_name>` using the exact SQL contract above, (6) read back `bronze.<table_name>` via schema-qualified name, (7) run `SELECT COUNT(1)` validation against `bronze.<table_name>`, (8) confirm schema listing contains the table, (9) log SUCCESS for the table.
- If any one of the nine per-table steps above fails for a source object, Bronze MUST stop immediately and raise an error that includes all of: `source_object_name`, `target_schema`, `target_table`, `phase`, and the underlying Spark exception text. Do not swallow the exception and do not continue with later tables.
- For Bronze, do not issue extra exploratory actions such as repeated `count()`, `display()`, wide `collect()`, `show()`, schema-profiling passes, or schema inference loops on every table beyond the single required post-write validation count; keep actions minimal to reduce statement-failure cascades and session cancellation risk.
- Source-read validation is mandatory before enrichment: each source object must be read explicitly from the source Lakehouse and validated as queryable before any target write begins. If the read fails, raise `Bronze source read failed for <source_object>` immediately rather than attempting downstream statements.
- The Bronze load MUST treat successful completion as dependent on all 13 required tables being discoverable. A partial success with fewer than 13 registered Bronze tables is a failure, not a warning.
- If generation logic would otherwise return no notebook cells, it MUST instead emit the required Bronze scaffold above and implement the full table loop with fail-fast runtime checks. Returning zero cells is not allowed.
- After writing each Bronze table, immediately verify all of the following before moving on:
  - the table is discoverable as `bronze.<table_name>`
  - `spark.table("bronze.<table_name>")` succeeds
  - it is queryable by Spark
  - a `SELECT COUNT(1)` against `bronze.<table_name>` succeeds
  - listing the `bronze` schema includes `<table_name>`
  - it contains at least the source columns plus the required Bronze technical columns
- After the full Bronze load, perform a completeness assertion that the full expected Bronze set exists and is discoverable:
  - `bronze.address`
  - `bronze.customer`
  - `bronze.customer_address`
  - `bronze.product`
  - `bronze.product_category`
  - `bronze.product_description`
  - `bronze.product_model`
  - `bronze.product_model_product_description`
  - `bronze.sales_order_detail`
  - `bronze.sales_order_header`
  - `bronze.v_get_all_categories`
  - `bronze.v_product_and_description`
  - `bronze.v_product_model_catalog_description`
- The final Bronze assertion MUST also verify the discoverable Bronze table count equals exactly `13` for the required set above. If the count is not exactly 13, fail Bronze with an explicit missing/unexpected-table diagnostic and do NOT allow the run to proceed.
- The final Bronze assertion MUST be based on exact schema-qualified object names, not only folder existence. Specifically, verify that each expected `bronze.<table_name>` can be resolved by Spark and that the set of resolved names matches the expected 13-name set exactly.
- If any expected Bronze table is missing from discovery after the write loop, fail Bronze with an explicit missing-table error and do NOT allow the run to proceed to Silver.
- Add standard technical columns to every bronze table:
  - `ingested_at_utc`
  - `run_id`
  - `source_workspace_id`
  - `source_lakehouse_id`
  - `source_object_name`
  - `source_row_hash` for change detection / dedup support
- Keep source column names intact in Bronze; do not snake_case until Silver.
- `source_row_hash` must be derived only from the source business columns present in the source DataFrame for that object, and must explicitly exclude the Bronze technical columns (`ingested_at_utc`, `run_id`, `source_workspace_id`, `source_lakehouse_id`, `source_object_name`) so reruns remain deterministic.
- Write mode: deterministic full overwrite for this run, because these are source snapshots and no CDC columns are provided.
- Partitioning:
  - Do not over-partition the small master tables.
  - For `sales_order_header` and `sales_order_detail`, prefer no partition initially unless runtime row counts justify date partitioning; if partitioning is needed, derive a date partition in Silver from `OrderDate` via header join rather than forcing it in Bronze.
- Binary handling:
  - Keep `ThumbNailPhoto` in `bronze.product`, but plan to drop or isolate it in Silver/Gold if not analytically useful.
- View-like source objects:
  - Land all three `v*` objects in Bronze for traceability, but treat them as optional accelerators in later layers because their content overlaps with base tables.

### Bronze table-specific notes
- `Address`: natural key appears to be `AddressID`; preserve all address text fields as-is.
- `Customer`: likely primary key `CustomerID`; retain sensitive columns in Bronze (`PasswordHash`, `PasswordSalt`) for lineage only, but mark for removal in Silver.
- `CustomerAddress`: likely junction table with composite key (`CustomerID`, `AddressID`, `AddressType`).
- `Product`: likely primary key `ProductID`; retain pricing and lifecycle dates.
- `ProductCategory`: likely self-referencing hierarchy via `ParentProductCategoryID`.
- `ProductDescription`: likely primary key `ProductDescriptionID`.
- `ProductModel`: likely primary key `ProductModelID`.
- `ProductModelProductDescription`: likely composite key (`ProductModelID`, `ProductDescriptionID`, `Culture`).
- `SalesOrderHeader`: header grain appears to be one row per `SalesOrderID`.
- `SalesOrderDetail`: line grain appears to be one row per (`SalesOrderID`, `SalesOrderDetailID`).
- `vGetAllCategories`: flattened category helper keyed by `ProductCategoryID`.
- `vProductAndDescription`: product-description helper, likely one row per (`ProductID`, `Culture`) if multilingual values exist.
- `vProductModelCatalogDescription`: product model enrichment keyed by `ProductModelID`.

## Silver
Curate to conformed, cleaned, snake_case tables in the `silver` schema. Remove sensitive fields, standardize types/names, deduplicate deterministically, and add audit columns.

### Silver standards
- Rename columns to `snake_case`.
- Trim strings and normalize blank strings to null where appropriate on descriptive fields.
- Add:
  - `silver_loaded_at_utc`
  - `record_source`
  - `is_current_row` where dedup logic keeps latest record
- Deduplicate by business key using latest available timestamp, typically `modified_date`, with `rowguid` as a tiebreaker where available.
- Run `OPTIMIZE` on large Silver outputs after writes, especially sales header/detail.
- Remove or quarantine analytically sensitive/non-reporting fields:
  - Drop `password_hash`, `password_salt` from customer in Silver.
  - Drop `thumbnail_photo` from product Silver unless explicitly required for reporting.

### Suggested silver tables and dedup keys
- `silver.address`
  - Dedup key: `address_id`
  - Keep latest by `modified_date`, then `rowguid`
  - Standardize city/state/country casing only if done deterministically
- `silver.customer`
  - Dedup key: `customer_id`
  - Keep latest by `modified_date`, then `rowguid`
  - Add derived `full_name`
  - Retain `company_name`, `sales_person`, `email_address`, `phone`
- `silver.customer_address`
  - Dedup key: (`customer_id`, `address_id`, `address_type`)
  - Keep latest by `modified_date`, then `rowguid`
  - Standardize `address_type`
- `silver.product`
  - Dedup key: `product_id`
  - Keep latest by `modified_date`, then `rowguid`
  - Add lifecycle flags in dependency order:
    - `is_discontinued`
    - `is_sellable_currently`
    - optional `is_priced` based on `list_price`
- `silver.product_category`
  - Dedup key: `product_category_id`
  - Keep latest by `modified_date`, then `rowguid`
  - Preserve parent-child hierarchy columns
- `silver.product_description`
  - Dedup key: `product_description_id`
  - Keep latest by `modified_date`, then `rowguid`
- `silver.product_model`
  - Dedup key: `product_model_id`
  - Keep latest by `modified_date`, then `rowguid`
- `silver.product_model_product_description`
  - Dedup key: (`product_model_id`, `product_description_id`, `culture`)
  - Keep latest by `modified_date`, then `rowguid`
- `silver.sales_order_header`
  - Dedup key: `sales_order_id`
  - Keep latest by `modified_date`, then `sales_order_number`, then `rowguid`
  - Add derived dates:
    - `order_date_key`
    - `due_date_key`
    - `ship_date_key`
  - Add flags:
    - `is_online_order`
    - `is_shipped`
- `silver.sales_order_detail`
  - Dedup key: (`sales_order_id`, `sales_order_detail_id`)
  - Keep latest by `modified_date`, then `rowguid`
  - Add `net_unit_price` = `unit_price - unit_price_discount` only if type-safe and business-approved
- `silver.category_flat`
  - Prefer sourcing from `vGetAllCategories`
  - Dedup key: `product_category_id`
  - Use as optional flattened helper rather than core source of truth
- `silver.product_description_flat`
  - Two plausible routes:
    - Route A: prefer `vProductAndDescription` as prebuilt helper keyed by (`product_id`, `culture`)
    - Route B: derive by joining `product` -> `product_model` -> `product_model_product_description` -> `product_description`
  - Recommend Route A for simpler semantic reporting, but retain Route B as fallback if row counts mismatch
- `silver.product_model_catalog_description`
  - Dedup key: `product_model_id`
  - Keep latest by `modified_date`, then `rowguid`
  - Treat as optional enrichment for narrative/product attributes

### Silver join intent
- Join `customer_address` to `address` only in downstream curated entities; keep base Silver tables normalized.
- Join `sales_order_detail` to `sales_order_header` in Gold, not in core Silver, to preserve line grain.
- Use `product_category_id` from product to category.
- Use `product_model_id` from product to model.
- Use `product_model_id` and `product_description_id` bridge to attach descriptions.
- Prefer base-table derivations over view-based helpers when reconciling discrepancies; if view and derived output differ, log row-count comparison and choose one path explicitly.

Gold should look like this:
Dim_Customer -> join customer, customeraddress (only use the AddressType='Main Office'), address and leave all relevant/meaningfull fields
Dim_Product -> join product, productcategory, producdescription, productmodel, productmodelproductdescription (where culture='en'). Please notice that in productcategory there is a self join. So in the end I have the category (which is the parent category) and a subcategory.
Dim_Sales --> Use the salesperson field from salesorderheader, but remove the adventureworks/ in front of the name and remove the last number at the end off the name
Fact_Sales --> join salesorderdetail and salesorderheader and keep all relevant fields

## Gold
Build business-facing star-schema outputs in the `gold` schema from the conformed Silver layer.

### Gold tables
- `gold.dim_customer`
  - Grain: one row per `customer_id`
  - Join `silver.customer` to `silver.customer_address` on `customer_id`
  - Filter `silver.customer_address.address_type = 'Main Office'`
  - Join to `silver.address` on `address_id`
  - Keep relevant customer and Main Office address attributes
- `gold.dim_product`
  - Grain: one row per `product_id`
  - Join `silver.product` to `silver.product_category` using `product.product_category_id = product_category.product_category_id`
  - Self-join `silver.product_category` to resolve parent category vs subcategory
  - Join `silver.product` to `silver.product_model` using `product_model_id`
  - Join through `silver.product_model_product_description` filtered to `culture = 'en'`
  - Join to `silver.product_description` on `product_description_id`
  - Output both parent category and child subcategory attributes
- `gold.dim_sales`
  - Grain: one row per cleaned salesperson
  - Source from distinct `silver.sales_order_header.sales_person`
  - Clean rule:
    - remove `adventure-works\` or `adventureworks\` prefix if present
    - remove trailing numeric suffix
- `gold.fact_sales`
  - Grain: one row per (`sales_order_id`, `sales_order_detail_id`)
  - Join `silver.sales_order_detail` to `silver.sales_order_header` on `sales_order_id`
  - Keep relevant line and header fields needed for reporting and semantic modeling

## Semantic model
Build a Direct Lake semantic model over the Gold outputs produced from the revised star schema.

### Tables
- `dim_customer`
  - Grain: one row per `customer_id`
  - Sourced from Gold `dim_customer`
  - Expected columns to expose:
    - `customer_id`
    - `full_name`
    - `company_name`
    - `title`
    - `email_address`
    - `phone`
    - `customer_name_style` if present
    - `address_id` for the selected Main Office address
    - `address_type` (should resolve to `Main Office` where populated)
    - `address_line1`
    - `address_line2`
    - `city`
    - `state_province`
    - `country_region`
    - `postal_code`
- `dim_product`
  - Grain: one row per `product_id`
  - Sourced from Gold `dim_product`
  - Expected columns to expose:
    - `product_id`
    - `product_name`
    - `product_number`
    - `color`
    - `size`
    - `weight`
    - `standard_cost`
    - `list_price`
    - `sell_start_date`
    - `sell_end_date`
    - `discontinued_date`
    - `is_discontinued`
    - `is_sellable_currently`
    - `product_model_id`
    - `product_model_name`
    - `product_description`
    - `culture`
    - `category_id` or `product_category_id`
    - `category_name` for the parent category
    - `subcategory_id`
    - `subcategory_name`
- `dim_sales`
  - Grain: one row per cleaned salesperson value
  - Sourced from Gold `dim_sales`
  - Expected columns to expose:
    - `sales_person`
    - optional helper columns if implemented, such as cleaned/raw variants, but hide raw variants from report authors
  - Business rule:
    - built from `sales_order_header.sales_person`
    - remove `adventure-works\` or `adventureworks\` prefix if present
    - remove trailing numeric suffix from the salesperson name/code
- `fact_sales`
  - Grain: one row per sales order detail line after joining detail and header
  - Sourced from Gold `fact_sales`
  - Expected columns to expose:
    - `sales_order_id`
    - `sales_order_detail_id`
    - `sales_order_number`
    - `customer_id`
    - `product_id`
    - `sales_person`
    - `order_date`
    - `due_date`
    - `ship_date`
    - `order_date_key` if materialized in Gold
    - `due_date_key` if materialized in Gold
    - `ship_date_key` if materialized in Gold
    - `ship_to_address_id`
    - `bill_to_address_id`
    - `status`
    - `online_order_flag` or `is_online_order`
    - `order_qty`
    - `unit_price`
    - `unit_price_discount`
    - `line_total`
    - `sub_total`
    - `tax_amt`
    - `freight`
    - `total_due`

### Relationships
- `fact_sales[customer_id]` -> `dim_customer[customer_id]` many-to-one
- `fact_sales[product_id]` -> `dim_product[product_id]` many-to-one
- `fact_sales[sales_person]` -> `dim_sales[sales_person]` many-to-one
- Do not create separate semantic relationships to address tables because customer geography is already denormalized into `dim_customer`.
- If `fact_sales` also carries customer geography attributes directly from header/address joins in Gold, keep those hidden unless required; prefer the conformed dimensions above.

### Recommended explicit measures
- `Sales Amount = SUM(fact_sales[line_total])`
- `Order Count = DISTINCTCOUNT(fact_sales[sales_order_id])`
- `Units Sold = SUM(fact_sales[order_qty])`
- `Average Order Value = DIVIDE(SUM(fact_sales[total_due]), [Order Count])`
- `Average Selling Price = DIVIDE([Sales Amount], [Units Sold])`
- `Discount Amount = SUMX(fact_sales, fact_sales[unit_price_discount] * fact_sales[order_qty])`
- `Freight Amount = SUM(fact_sales[freight])`
- `Tax Amount = SUM(fact_sales[tax_amt])`
- `Online Order Count = CALCULATE([Order Count], fact_sales[online_order_flag] = TRUE())`
  - If Gold uses `is_online_order` instead of `online_order_flag`, bind the measure to that final column name instead.
- `Shipped Order Count = CALCULATE([Order Count], NOT ISBLANK(fact_sales[ship_date]))`

### Semantic modeling notes
- This model is intentionally centered on a single line-grain fact table plus three dimensions: customer, product, and salesperson.
- Because `fact_sales` contains header values repeated at line grain, use:
  - `line_total` and `order_qty` for line/product analysis
  - `total_due`, `freight`, and `tax_amt` only with caution for order-level analysis
- To avoid double counting on order-level financials, prefer either:
  - report visuals filtered to distinct orders, or
  - additional DAX measures that aggregate by distinct `sales_order_id` when needed
- Hide technical fields and duplicate raw variants, especially:
  - raw salesperson text if both raw and cleaned forms exist
  - surrogate/helper keys not intended for authoring
  - culture if it is fixed to `en`
- Expose a product hierarchy:
  - `category_name` > `subcategory_name` > `product_name`
- Expose a customer geography hierarchy:
  - `country_region` > `state_province` > `city` > `postal_code`
- If `dim_product` contains only English descriptions by design, no culture slicer is needed.

## Report
Create a Power BI report tailored to the revised Gold model with customer, product, salesperson, and fact-level sales analysis. Always include a Data quality page.

### Page 1: Sales Overview
- KPI cards:
  - Sales Amount
  - Order Count
  - Units Sold
  - Average Order Value
- Line chart:
  - Sales Amount by `fact_sales[order_date]`
- Clustered column chart:
  - Sales Amount by `dim_product[category_name]`
- Bar chart:
  - Sales Amount by `dim_sales[sales_person]`
- Matrix:
  - `dim_product[category_name]` -> `dim_product[subcategory_name]` -> `dim_product[product_name]`
  - Values:
    - Sales Amount
    - Units Sold
    - Average Selling Price

### Page 2: Customer and Geography
- Bar chart:
  - Top customers by Sales Amount using `dim_customer[full_name]` or `dim_customer[company_name]`
- Map or filled map if geography quality is sufficient:
  - Sales Amount by `dim_customer[country_region]`, `dim_customer[state_province]`, and `dim_customer[city]`
- Table:
  - `dim_customer[full_name]`
  - `dim_customer[company_name]`
  - `dim_customer[email_address]`
  - `dim_customer[phone]`
  - `dim_customer[address_line1]`
  - `dim_customer[city]`
  - `dim_customer[state_province]`
  - Order Count
  - Sales Amount
- Slicer set:
  - `fact_sales[order_date]`
  - `dim_customer[country_region]`
  - `dim_customer[state_province]`
  - `dim_sales[sales_person]`

### Page 3: Product Performance
- Bar chart:
  - Top products by Sales Amount or Units Sold using `dim_product[product_name]`
- Scatter chart:
  - `dim_product[list_price]` vs Sales Amount by `dim_product[product_name]`
- Table:
  - `dim_product[product_name]`
  - `dim_product[product_number]`
  - `dim_product[color]`
  - `dim_product[size]`
  - `dim_product[standard_cost]`
  - `dim_product[list_price]`
  - `dim_product[category_name]`
  - `dim_product[subcategory_name]`
  - Sales Amount
  - Units Sold
- Optional decomposition tree:
  - Sales Amount by `category_name` > `subcategory_name` > `product_name`
- Optional detail card or table:
  - `dim_product[product_model_name]`
  - `dim_product[product_description]`
  - `dim_product[is_sellable_currently]`
  - `dim_product[is_discontinued]`

### Page 4: Salesperson Analysis
- Bar chart:
  - Sales Amount by `dim_sales[sales_person]`
- Table:
  - `dim_sales[sales_person]`
  - Order Count
  - Units Sold
  - Sales Amount
  - Average Order Value
- Trend visual:
  - Sales Amount by `fact_sales[order_date]` split by `dim_sales[sales_person]`
- Slicers:
  - `dim_sales[sales_person]`
  - `dim_product[category_name]`
  - `fact_sales[order_date]`

### Page 5: Data quality
- Cards:
  - Gold `fact_sales` row count
  - Gold `dim_customer` distinct customer count
  - Gold `dim_product` distinct product count
  - Gold `dim_sales` distinct salesperson count
- Validation cards or KPI indicators:
  - customers missing Main Office address
  - products missing English description
  - products missing parent category or subcategory
  - fact rows with missing customer, product, or salesperson key
- Table:
  - failing test name
  - table
  - severity
  - failed row count
  - run id
- Bar chart:
  - failures by Gold table
- Detail grid:
  - sample missing `customer_id`, `product_id`, or cleaned `sales_person`
  - sample products with null `category_name` / `subcategory_name`

### Report design notes
- Prefer `fact_sales` for all core metrics