# Run Spec 20260518-071602-ff99c8

## Updated specs

### Iteration 1 — 2026-05-18 07:26:07Z — failed layer: bronze (run: 20260518-072303-b0e64b)
- **Root cause (1-line summary)**: Bronze completed without creating discoverable Delta tables under `Tables/bronze/`, so Silver could not detect any upstream outputs and refused to continue.
- **Cross-table audit**: `SalesLT/Address`: yes — same landing contract must create `bronze.address`; `SalesLT/Customer`: yes — same; `SalesLT/CustomerAddress`: yes — same; `SalesLT/Product`: yes — same; `SalesLT/ProductCategory`: yes — same; `SalesLT/ProductDescription`: yes — same; `SalesLT/ProductModel`: yes — same; `SalesLT/ProductModelProductDescription`: yes — same; `SalesLT/SalesOrderDetail`: yes — same; `SalesLT/SalesOrderHeader`: yes — same; `SalesLT/vGetAllCategories`: yes — same; `SalesLT/vProductAndDescription`: yes — same; `SalesLT/vProductModelCatalogDescription`: yes — same. The failure is systemic because discoverability depends on how every Bronze table is written and registered, not on one table’s business schema.
- **Fix approach**: **GENERALIZE** — the root cause is a uniform Bronze publication/discoverability issue that can hit every listed source table, so one explicit landing-and-validation rule should apply to all Bronze outputs.
- **What was changed**:
  - Tightened **Generic guidance** with a mandatory publish/verify contract: every produced layer table must be both written as Delta and discoverable through the Lakehouse `Tables/<schema>/` namespace before downstream work proceeds.
  - Tightened **Bronze** to require exact target table names, mandatory registration in schema `bronze`, and a post-write verification step that asserts all expected Bronze tables are discoverable.
  - Added explicit Bronze failure behavior: if any expected Bronze table is missing after write/registration, fail Bronze loudly instead of allowing Silver to start.

### Iteration 2 — 2026-05-18 07:31:07Z — failed layer: bronze (run: 20260518-072303-b0e64b)
- **Root cause (1-line summary)**: Silver again found zero discoverable tables because Bronze must not merely write data files; it must create catalog-discoverable managed tables in schema `bronze` under `Tables/bronze/` and verify them one-by-one before declaring success.
- **Cross-table audit**: `SalesLT/Address`: yes — must publish as `bronze.address`; `SalesLT/Customer`: yes — must publish as `bronze.customer`; `SalesLT/CustomerAddress`: yes — must publish as `bronze.customer_address`; `SalesLT/Product`: yes — must publish as `bronze.product`; `SalesLT/ProductCategory`: yes — must publish as `bronze.product_category`; `SalesLT/ProductDescription`: yes — must publish as `bronze.product_description`; `SalesLT/ProductModel`: yes — must publish as `bronze.product_model`; `SalesLT/ProductModelProductDescription`: yes — must publish as `bronze.product_model_product_description`; `SalesLT/SalesOrderDetail`: yes — must publish as `bronze.sales_order_detail`; `SalesLT/SalesOrderHeader`: yes — must publish as `bronze.sales_order_header`; `SalesLT/vGetAllCategories`: yes — must publish as `bronze.v_get_all_categories`; `SalesLT/vProductAndDescription`: yes — must publish as `bronze.v_product_and_description`; `SalesLT/vProductModelCatalogDescription`: yes — must publish as `bronze.v_product_model_catalog_description`. All listed sources are affected because the root cause is publication/registration behavior, not business schema.
- **Fix approach**: **GENERALIZE** — the same explicit managed-table publication contract and verification loop is required for every Bronze source, so one uniform rule is safer than per-table customizations.
- **What was changed**:
  - Tightened **Generic guidance** to forbid treating raw Delta files, temp views, or path-only writes as successful medallion outputs; success now requires schema-qualified read-back and catalog visibility.
  - Tightened **Bronze** with a required per-table publish sequence: overwrite target managed Delta table, verify `spark.read.table('bronze.<name>')` succeeds, and verify the table name appears in schema `bronze` listing before moving to the next table.
  - Added an explicit final Bronze gate listing the exact 13 required table names; Bronze must raise an error naming any missing tables and stop the run.

### Iteration 3 — 2026-05-18 07:34:34Z — failed layer: bronze (run: 20260518-072303-b0e64b)
- **Root cause (1-line summary)**: Bronze likely hit one or more statement-level failures that cancelled the Spark session; the current spec is too vague on per-table isolation, source existence/read validation, and fail-fast diagnostics to identify which table caused the cancellation.
- **Cross-table audit**: `SalesLT/Address`: yes — source read/publish/verify can fail and cancel the session; `SalesLT/Customer`: yes — same; `SalesLT/CustomerAddress`: yes — same; `SalesLT/Product`: yes — same; `SalesLT/ProductCategory`: yes — same; `SalesLT/ProductDescription`: yes — same; `SalesLT/ProductModel`: yes — same; `SalesLT/ProductModelProductDescription`: yes — same; `SalesLT/SalesOrderDetail`: yes — same; `SalesLT/SalesOrderHeader`: yes — same; `SalesLT/vGetAllCategories`: yes — same; `SalesLT/vProductAndDescription`: yes — same; `SalesLT/vProductModelCatalogDescription`: yes — same. The root cause is plausibly systemic because any Bronze source read, metadata-column addition, partition write, or publish verification failure can abort the session before downstream detection.
- **Fix approach**: **GENERALIZE** — every Bronze source needs the same defensive read → validate → write → verify sequence and per-table diagnostics, because the traceback does not isolate a single business table/column and the same session-cancellation pattern could occur for any listed source.
- **What was changed**:
  - Tightened **Generic guidance** to require a per-table execution log/checkpoint pattern so the failing source table is explicitly identified before Spark session cancellation bubbles up.
  - Tightened **Bronze** with an ordered per-table contract: assert source readability first, normalize metadata-column creation safely, write exactly one table at a time, and verify each published table before starting the next.
  - Added explicit Bronze guards for optional partitioning and source/object existence so non-partitionable or unreadable sources fail with table-specific diagnostics instead of causing opaque session-level cancellation.

### Iteration 4 — 2026-05-18 07:46:52Z — failed layer: bronze (run: 20260518-072303-b0e64b)
- **Root cause (1-line summary)**: Bronze publication verification expected the wrong schema shape and metadata column names (`ingestion_timestamp`, `source_path`, `batch_id`, `ingestion_date`), causing read-back validation to fail with an empty-schema result for `bronze.address`.
- **Cross-table audit**: `SalesLT/Address`: yes — failed directly during read-back column assertion; `SalesLT/Customer`: yes — same Bronze verifier would expect the wrong metadata names; `SalesLT/CustomerAddress`: yes — same; `SalesLT/Product`: yes — same; `SalesLT/ProductCategory`: yes — same; `SalesLT/ProductDescription`: yes — same; `SalesLT/ProductModel`: yes — same; `SalesLT/ProductModelProductDescription`: yes — same; `SalesLT/SalesOrderDetail`: yes — same; `SalesLT/SalesOrderHeader`: yes — same; `SalesLT/vGetAllCategories`: yes — same; `SalesLT/vProductAndDescription`: yes — same; `SalesLT/vProductModelCatalogDescription`: yes — same. The same verifier/schema-mismatch root cause can hit every Bronze table because the verification logic is shared and the metadata contract must be uniform across all tables.
- **Fix approach**: **GENERALIZE** — the failure is a systemic Bronze schema-contract mismatch across all source tables, so one explicit canonical metadata contract and verification rule should apply uniformly to every Bronze publish/read-back.
- **What was changed**:
  - Tightened **Generic guidance** to enforce one canonical Bronze metadata column set only, and to forbid any alternate legacy metadata names in verification logic.
  - Tightened **Bronze** so read-back verification must validate against the exact source business columns plus only the four required metadata columns: `ingested_at_utc`, `run_id`, `source_lakehouse`, `source_table`, with no renaming of business columns.
  - Added an explicit Bronze rule that verification must inspect the schema-qualified table object `bronze.<table_name>` only and fail with a table-specific message if the read-back schema is empty or missing the original source columns.

### Iteration 5 — 2026-05-18 07:53:35Z — failed layer: bronze (run: 20260518-072303-b0e64b)
- **Root cause (1-line summary)**: Silver still found zero discoverable Bronze tables, indicating Bronze must use a Fabric-supported catalog publish path that creates actual lakehouse tables under `Tables/bronze/`, not only Delta files or non-catalog objects.
- **Cross-table audit**: `SalesLT/Address`: yes — must materialize as a real lakehouse table `bronze.address`; `SalesLT/Customer`: yes — same; `SalesLT/CustomerAddress`: yes — same; `SalesLT/Product`: yes — same; `SalesLT/ProductCategory`: yes — same; `SalesLT/ProductDescription`: yes — same; `SalesLT/ProductModel`: yes — same; `SalesLT/ProductModelProductDescription`: yes — same; `SalesLT/SalesOrderDetail`: yes — same; `SalesLT/SalesOrderHeader`: yes — same; `SalesLT/vGetAllCategories`: yes — same; `SalesLT/vProductAndDescription`: yes — same; `SalesLT/vProductModelCatalogDescription`: yes — same. All listed inputs share the same publication risk because discoverability is determined by the Bronze write/register pattern, not by each table’s business columns.
- **Fix approach**: **GENERALIZE** — one explicit Fabric lakehouse publication contract is needed for all Bronze tables because the same non-discoverable-output failure can hit every source uniformly.
- **What was changed**:
  - Tightened **Generic guidance** to require Fabric-supported catalog publication that yields both schema-qualified read-back and physical presence under `Tables/<schema>/<table_name>`.
  - Tightened **Bronze** to forbid path-only writes as final output and to require a post-publish verification against both the catalog and the lakehouse tables filesystem shape for every Bronze target.
  - Added an explicit Bronze implementation constraint to use managed table creation/overwrite semantics that publish the final objects as `bronze.<table_name>` in the target lakehouse before Bronze can report success.

### Iteration 6 — 2026-05-18 08:03:43Z — failed layer: bronze (run: 20260518-072303-b0e64b)
- **Root cause (1-line summary)**: Bronze verification failed on `bronze.address` because the post-write read-back returned `available=[]`; the spec allowed verification before confirming the catalog object had a non-empty schema, and it was too vague about the exact publish/read-back sequence to use.
- **Cross-table audit**: `SalesLT/Address`: yes — failed directly with empty-schema read-back; `SalesLT/Customer`: yes — same publish/read-back race or non-materialized table risk; `SalesLT/CustomerAddress`: yes — same; `SalesLT/Product`: yes — same; `SalesLT/ProductCategory`: yes — same; `SalesLT/ProductDescription`: yes — same; `SalesLT/ProductModel`: yes — same; `SalesLT/ProductModelProductDescription`: yes — same; `SalesLT/SalesOrderDetail`: yes — same; `SalesLT/SalesOrderHeader`: yes — same; `SalesLT/vGetAllCategories`: yes — same; `SalesLT/vProductAndDescription`: yes — same; `SalesLT/vProductModelCatalogDescription`: yes — same. The root cause is systemic because the same publish-then-read-back contract is used for every Bronze table.
- **Fix approach**: **GENERALIZE** — all Bronze tables need the same stricter publish materialization and schema-read-back rule, so one uniform Bronze contract is safer than table-by-table exceptions.
- **What was changed**:
  - Tightened **Generic guidance** with a new rule: schema assertions must only occur after a successful schema-qualified read-back that returns a non-empty column list, and empty-schema results must be treated as publication failures rather than normal missing-column checks.
  - Tightened **Bronze** to require a two-phase verification sequence for every target: first confirm catalog registration and successful `spark.read.table(...)`, then inspect the returned schema and only then compare expected columns.
  - Added an explicit Bronze constraint to avoid immediate column assertions against a table object that has not yet materialized with a readable schema; the error path must report `available=[]` as an empty-schema publication defect.

## Inputs
- Workspace: `16119d4a-a872-4e9f-b622-0b14f8fb05b7`
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
- Target Lakehouse: **e2e**

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
- Use defensive column references everywhere; never assume a column exists after projection/rename without checking.
- After every join, immediately project/select alias-prefixed columns into flat output names; do not carry ambiguous duplicate names forward.
- Before every groupBy/agg, assert the required grouping and measure columns exist on the DataFrame in that step.
- For all REST/API responses, use defensive handling: `if x is None: raise ...` before any `.get()` access.
- Do not use `saveAsTable`; write to managed Delta paths/tables with supported Fabric patterns.
- Include parameter cells at the top of each notebook for workspace, lakehouse, schema, run_id, and table names.
- Use idempotent overwrite patterns for rebuildable layer outputs; if incremental logic is added, make merge keys explicit.
- All try/except blocks must fail loudly, call `_save_error(layer, e)`, and then re-raise.
- Keep transformations deterministic and re-runnable for the same inputs.
- Prefer overwrite of full curated outputs in Silver/Gold for this run because the selected source set is compact and relationally complete.
- Publication/discoverability rule for every layer output: a write is not considered successful unless the resulting Delta table is discoverable from the target lakehouse `Tables/<schema>/` namespace by its final table name. After each table write, validate discoverability via catalog/list-tables logic or direct read-back of the exact target table identifier before proceeding.
- Never rely on path-only outputs for medallion handoff. Downstream layers must be able to enumerate/read the produced tables by schema-qualified table name (for example `bronze.address`), so the producing layer must register/publish each output accordingly.
- If a layer expects a fixed set of outputs from the prior layer, compute that expected set explicitly and fail immediately in the producing layer if any are missing, rather than allowing the next layer to infer from spec-only context.
- Treat temp views, cached DataFrames, notebook-local variables, and raw `/Files` or unregistered `/Tables` paths as non-published artifacts. None of these satisfy the handoff contract to the next medallion layer.
- For every published table, the success check must include BOTH of the following against the final schema-qualified name: (1) a catalog/listing check that the table name appears in the target schema, and (2) a direct `spark.read.table('<schema>.<table_name>')` read-back. If either fails, the layer has not published successfully.
- In addition to catalog/read-back checks, every published table must physically exist in the target lakehouse table namespace at `Tables/<schema>/<table_name>` after publication. If the catalog entry exists but the lakehouse `Tables/<schema>/` location does not contain the named table folder/object, treat publication as failed because downstream enumeration depends on that namespace.
- For Fabric lakehouse outputs, final publication must use a supported managed-table creation/overwrite pattern that results in a real lakehouse table object under `Tables/<schema>/`. Writing Delta files to an arbitrary path and assuming they are discoverable is not acceptable.
- For any notebook that loops over multiple tables, process exactly one table at a time in a deterministic order and emit a table-scoped progress log before and after each major step (`source_check`, `read`, `transform`, `write`, `publish_verify`). If Spark later cancels the session, the notebook output must still reveal the last table and step attempted.
- Before performing a write for any source table, explicitly validate that the source object is readable by its exact schema-qualified source name and fail immediately with that exact name if the read fails. Do not continue to the next table after a source-read failure.
- Any optional behavior that depends on a source column existing (for example, deriving a partition column from `OrderDate`) must first assert the column exists on that source DataFrame; if absent, skip that optional behavior or fail with a table-specific message. Never let an optional optimization be the first point of failure.
- Canonical metadata contract rule: when a layer spec defines required metadata columns, verification logic must assert only those exact names and must not substitute legacy or inferred alternatives. For Bronze, the only allowed standard metadata columns are `ingested_at_utc`, `run_id`, `source_lakehouse`, and `source_table`.
- Do not mix metadata contracts across templates. Names such as `ingestion_timestamp`, `source_path`, `batch_id`, and `ingestion_date` are not valid Bronze-required columns for this run and must not appear in assertions, publish checks, or completion criteria.
- Schema verification must compare against the actual DataFrame read from the schema-qualified table name. If `spark.read.table('<schema>.<table>')` returns a DataFrame with zero columns, treat that as a publication/read-back failure immediately and report the table name plus the empty `available=[]` schema state.
- New publication verification rule: do not perform expected-column comparison until AFTER the notebook has confirmed that `spark.read.table('<schema>.<table>')` returned a DataFrame with a non-empty schema. An empty-schema read-back is a distinct publish/materialization failure, not a normal missing-column case, and must short-circuit the verification flow before any business-column diff logic.

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
Purpose: land exact source tables into `bronze` with minimal transformation, preserving source fidelity and enough metadata for replay/debug.

Landing approach for all selected tables:
- Read each source table/view from the source lakehouse and write to a discoverable managed Delta table in schema `bronze` using the exact target names below.
- For each source, the Bronze notebook must both:
  - write the Delta data for the target table, and
  - publish/register it so it is discoverable as `bronze.<table_name_snake_case>` from the target lakehouse catalog and visible under `Tables/bronze/`.
- Preserve original business columns exactly as ingested; only add standard metadata columns:
  - `ingested_at_utc`
  - `run_id`
  - `source_lakehouse`
  - `source_table`
  - `source_record_hash` over non-binary business columns where practical
- Use overwrite for this run to keep notebooks idempotent.
- Partition large transactional bronze tables by a derived date where present:
  - `bronze.sales_order_header`: partition by `order_date_date` derived from source column `OrderDate` if and only if `OrderDate` exists on the DataFrame being written
  - `bronze.sales_order_detail`: no natural date on detail, so do not partition independently unless joined staging is explicitly created
- Do not partition small/master tables such as address, customer, product, category, model, description, and bridge tables.
- Mandatory per-table execution order for Bronze:
  1. Resolve the exact source object name from the mapping below.
  2. Assert the source object is readable by that exact name before any transform/write logic.
  3. Read the source into a DataFrame and capture the original source column list for later verification.
  4. Add Bronze metadata columns only; do not rename or reshape business columns in Bronze.
  5. Apply table-specific optional partitioning only after asserting the required source column exists.
  6. Publish exactly one target managed Delta lakehouse table using a Fabric-supported catalog creation/overwrite pattern that results in a real object at `bronze.<table_name>` under `Tables/bronze/<table_name>`.
  7. Immediately verify the published target table before starting the next source table.
- Mandatory Bronze publication implementation constraints:
  - The final Bronze artifact must be a catalog-discoverable lakehouse table, not just Delta files written to a folder.
  - Writing directly to a filesystem path is allowed only as an internal staging step; it is not the completion step. The completion step must create/replace the schema-qualified lakehouse table object `bronze.<table_name>`.
  - The publish step must be completed as a single table-materialization action for the final identifier `bronze.<table_name>`; do not verify against an intermediate object, temp view, staged path, or partially registered table.
  - After publication, `bronze.<table_name>` must be readable through `spark.read.table(...)` and also materially present in the target lakehouse `Tables/bronze/` namespace with the same final table name.
  - Do not treat temp views, SQL views, or notebook-local DataFrames as a successful Bronze publish.
- Mandatory Bronze schema contract:
  - Business columns in Bronze must remain exactly as read from the source object, preserving original casing and names such as `AddressID`, `AddressLine1`, `City`, `rowguid`, and `ModifiedDate`.
  - Bronze verification must assert the read-back table contains all captured original source business columns plus the required Bronze metadata columns actually defined in this spec.
  - For this run, the required Bronze verification set is exactly:
    - all original source business columns captured at step 3, plus
    - `ingested_at_utc`
    - `run_id`
    - `source_lakehouse`
    - `source_table`
  - `source_record_hash` is recommended where practical and may be validated when present, but it must not replace or rename any of the four required standard metadata columns above.
  - Do not assert or generate any legacy ingestion columns not defined in this spec, including:
    - `ingestion_timestamp`
    - `source_path`
    - `batch_id`
    - `ingestion_date`
- Mandatory per-table publish contract for Bronze:
  - Create schema `bronze` before any writes if it does not already exist.
  - For each source table, publish exactly one managed Delta table with the exact schema-qualified target name listed below.
  - After writing each table, immediately verify all six conditions in this exact order before moving to the next source:
    - `bronze.<table_name>` appears in the target lakehouse schema listing for `bronze`
    - `spark.read.table('bronze.<table_name>')` succeeds
    - the read-back DataFrame has a non-empty schema (`len(read_back.columns) > 0`); `available=[]` is a hard publication/materialization failure and must stop verification immediately
    - only after the non-empty-schema check passes, assert the read-back DataFrame contains all original source business columns plus the required Bronze metadata columns from this spec
    - the published table exists in the lakehouse tables namespace under `Tables/bronze/<table_name>`
    - the table verified is the exact final object `bronze.<table_name>`, not a path-only or pre-publish DataFrame
  - Verification must be performed against the schema-qualified object `bronze.<table_name>` itself. Do not verify against a path-only DataFrame, temp view, or stale variable from before publication.
  - If any one table fails source validation, write, publication, or verification, stop Bronze immediately; do not continue to remaining tables and do not allow the notebook to report success.
- Mandatory diagnostics for Bronze:
  - Emit a start/end log line for each source table using the exact source name and exact target name.
  - Emit a start/end log line for each sub-step: `source_check`, `read`, `metadata_add`, `write`, `publish_verify`.
  - Emit the exact publish method used for each table and whether the final object was verified in both the catalog and `Tables/bronze/<table_name>`.
  - Any raised error message must include both the source object name (for example `SalesLT/SalesOrderHeader`) and the target table name (for example `bronze.sales_order_header`) where applicable.
  - If read-back verification finds an empty schema or missing columns, the error must list:
    - exact source object name
    - exact target table name
    - whether catalog listing succeeded
    - whether `spark.read.table(...)` succeeded
    - missing column names only if the schema was non-empty
    - available read-back column list
  - If namespace verification fails, the error must list:
    - exact source object name
    - exact target table name
    - expected lakehouse location `Tables/bronze/<table_name>`
    - whether catalog listing succeeded
    - whether `spark.read.table(...)` succeeded
  - If a table-specific step fails, fail Bronze at that table immediately instead of attempting the remaining tables; this is required to avoid opaque session-cancelled outcomes with no actionable culprit table.
- Mandatory source/metadata guards for Bronze:
  - For `source_record_hash`, hash only non-binary business columns actually present on the DataFrame; explicitly exclude binary/image-like columns such as `ThumbNailPhoto` and any other binary-typed columns from the hash input.
  - If a table has zero rows, it must still be published as an empty Bronze table with the correct schema and required metadata columns; zero-row sources are not considered missing tables. Zero rows is valid, but zero columns (`available=[]`) is never valid.
  - Views (`vGetAllCategories`, `vProductAndDescription`, `vProductModelCatalogDescription`) must be treated with the same readability and publication checks as base tables.
- Mandatory post-write verification for Bronze: after all writes complete, assert that the full expected table set below is discoverable in schema `bronze` and physically present in `Tables/bronze/`. If any table is missing, Bronze must fail loudly and must not report success.
- Bronze success must be determined from the published catalog state and lakehouse table namespace only. File existence somewhere else is insufficient.

Required Bronze target table names:
- `SalesLT/Address` -> `bronze.address`
- `SalesLT/Customer` -> `bronze.customer`
- `SalesLT/CustomerAddress` -> `bronze.customer_address`
- `SalesLT/Product` -> `bronze.product`
- `SalesLT/ProductCategory` -> `bronze.product_category`
- `SalesLT/ProductDescription` -> `bronze.product_description`
- `SalesLT/ProductModel` -> `bronze.product_model`
- `SalesLT/ProductModelProductDescription` -> `bronze.product_model_product_description`
- `SalesLT/SalesOrderDetail` -> `bronze.sales_order_detail`
- `SalesLT/SalesOrderHeader` -> `bronze.sales_order_header`
- `SalesLT/vGetAllCategories` -> `bronze.v_get_all_categories`
- `SalesLT/vProductAndDescription` -> `bronze.v_product_and_description`
- `SalesLT/vProductModelCatalogDescription` -> `bronze.v_product_model_catalog_description`

Bronze completion criteria:
- The schema `bronze` exists in the target lakehouse.
- All 13 required Bronze tables above are readable by schema-qualified name.
- A catalog/listing check over schema `bronze` returns at least those 13 names.
- The lakehouse namespace `Tables/bronze/` contains all 13 required table objects/folders by final table name.
- Every required Bronze table read-back contains its original source business columns plus these required metadata columns:
  - `ingested_at_utc`
  - `run_id`
  - `source_lakehouse`
  - `source_table`
- Silver must be able to enumerate Bronze from actual lakehouse tables only; if Bronze outputs exist only as files, temp views, or non-discoverable objects, that is a build failure.
- The final verification step must compare the actual discovered set in schema `bronze` to this exact expected set and fail with the explicit missing table names if there is any mismatch:
  - `address`
  - `customer`
  - `customer_address`
  - `product`
  - `product_category`
  - `product_description`
  - `product_model`
  - `product_model_product_description`
  - `sales_order_detail`
  - `sales_order_header`
  - `v_get_all_categories`
  - `v_product_and_description`
  - `v_product_model_catalog_description`

Table-specific notes:
- `Address`: land as-is; this is a location/master table likely used for regional analytics.
- `Customer`: land as-is even though it contains sensitive columns (`PasswordHash`, `PasswordSalt`). These must not survive into Gold or the semantic model.
- `CustomerAddress`: land for traceability, but per user requirement it is not used in the primary Gold customer dimension design unless the user later changes the spec.
- `Product`, `ProductCategory`, `ProductDescription`, `ProductModel`, `ProductModelProductDescription`: land all base product entities because the requested Gold product dimension is assembled from them.
- `SalesOrderHeader` and `SalesOrderDetail`: core sales transaction sources; preserve all monetary and status fields.
- Views:
  - `vGetAllCategories`, `vProductAndDescription`, `vProductModelCatalogDescription` should also be landed because they may be useful as validation/reference sources.
  - Gold should prefer base tables for lineage unless a view uniquely supplies attributes more cleanly than the normalized tables.

## Silver
Purpose: standardize names/types, remove obvious duplicates, clean high-value attributes, and prepare conformed entities for Gold.

Common Silver standards for every table:
- Rename columns to `snake_case`.
- Add audit columns:
  - `silver_loaded_at_utc`
  - `run_id`
  - `record_source`
- Standardize empty strings to null for descriptive text fields where appropriate.
- Retain `rowguid` as a source lineage field, not a business key.
- Run `OPTIMIZE` after writes for larger tables (`sales_order_header`, `sales_order_detail`, `product` and final conformed outputs).

Suggested deduplication keys and retention logic:
- `silver.address`: dedup on `address_id`, keep latest by `modified_date`.
- `silver.customer`: dedup on `customer_id`, keep latest by `modified_date`.
- `silver.customer_address`: dedup on composite (`customer_id`, `address_id`, `address_type`), keep latest by `modified_date`.
- `silver.product`: dedup on `product_id`, keep latest by `modified_date`.
- `silver.product_category`: dedup on `product_category_id`, keep latest by `modified_date`.
- `silver.product_description`: dedup on `product_description_id`, keep latest by `modified_date`.
- `silver.product_model`: dedup on `product_model_id`, keep latest by `modified_date`.
- `silver.product_model_product_description`: dedup on composite (`product_model_id`, `product_description_id`, `culture`), keep latest by `modified_date`.
- `silver.sales_order_header`: dedup on `sales_order_id`, keep latest by `modified_date`.
- `silver.sales_order_detail`: dedup on composite (`sales_order_id`, `sales_order_detail_id`), keep latest by `modified_date`.
- `silver.v_get_all_categories`: dedup on `product_category_id`.
- `silver.v_product_and_description`: dedup on composite (`product_id`, `culture`).
- `silver.v_product_model_catalog_description`: dedup on `product_model_id`, keep latest by `modified_date`.

Business cleaning/conformance:
- `silver.customer`
  - Build normalized full-name helper fields from title/first/middle/last/suffix.
  - Standardize email to lowercase for reporting convenience, but do not use as key.
  - Parse `sales_person` into two fields:
    - `sales_person_raw`
    - `sales_person_clean`
  - Cleaning rule: when value matches `<domain>\<username>`, keep username only; then proper-case for display, e.g. `adventure-works\jillian0` -> `Jillian0`.
- `silver.address`
  - Standardize city/state/country strings with trimmed casing.
  - Create region-friendly helpers: `city_norm`, `state_province_norm`, `country_region_norm`, `postal_code_norm`.
- `silver.product`
  - Remove binary `thumbnail_photo` from downstream curated joins unless explicitly needed.
  - Create helper flags:
    - `is_active_sell_window` from `sell_start_date`, `sell_end_date`, `discontinued_date`
    - `is_discontinued`
- `silver.product_category`
  - Resolve parent-child hierarchy into helper columns in a conformed category staging table:
    - `category_id`
    - `parent_category_id`
    - `category_name`
    - later Gold will flatten to category/subcategory.
- `silver.product_model_product_description`
  - Filter a conformed helper dataset for `culture = 'en'` to satisfy user requirement.
- `silver.sales_order_header`
  - Standardize dates: `order_date`, `due_date`, `ship_date` and derive `order_date_key`, `ship_date_key`.
  - Create normalized status label from numeric `status`.
  - Keep order-level descriptive attributes (`sales_order_number`, `purchase_order_number`, `account_number`, `ship_method`, `online_order_flag`, `comment`) for the Gold order dimension.
- `silver.sales_order_detail`
  - Validate monetary fields are non-negative where expected.
  - Add `discount_amount = order_qty * unit_price * unit_price_discount`.
  - Add `gross_sales_amount = order_qty * unit_price`.
  - Preserve `line_total` as provided source net line amount.

Important note on requested customer design:
- The user requested: "Customer: Combine Customer and Address. Don’t use CustomerAddress in between."
- With the actual schema provided, `Customer` does not contain `address_id`, and `Address` does not contain `customer_id`. The only relational path between them is `CustomerAddress`.
- Therefore, a direct relational combine of Customer and Address is not technically supported by the actual columns.
- Gold should implement one of these options, with Option 1 recommended unless the user edits the spec:
  - Option 1 (recommended, schema-valid): use `CustomerAddress` only as a technical bridge in Silver/Gold plumbing, but do not surface it as a business entity; return a single customer dimension that contains customer plus a preferred address.
  - Option 2 (strict literal interpretation): build customer dimension from `Customer` only and exclude address fields entirely.
- This spec proceeds with Option 1 because regional reporting requires address attributes and the schema otherwise cannot support them.

## Gold
Purpose: deliver a star schema for sales reporting, matching the user’s requested entities and regional/discount analysis needs.

Star schema tables to create:

### Dimensions

#### `gold.dim_order_date`
- Source: `silver.sales_order_header.order_date`
- Grain: one row per calendar date.
- Key: `order_date_key` as `yyyymmdd` integer.
- Include standard calendar attributes and a hierarchy:
  - year > quarter > month > date
- Mark as the active date dimension for order-based analysis.

#### `gold.dim_ship_date`
- Source: `silver.sales_order_header.ship_date`
- Grain: one row per calendar date, including null-handling strategy for unshipped orders.
- Key: `ship_date_key` as `yyyymmdd` integer, with unknown/default member for null ship dates.
- Include same calendar hierarchy:
  - year > quarter > month > date
- This satisfies the user requirement for a separate ship-date dimension with hierarchy.

#### `gold.dim_customer`
- Intended shape per user: combine customer and address, keep only relevant fields.
- Recommended implementation using actual schema:
  - Join `silver.customer` -> `silver.customer_address` -> `silver.address`
  - Choose one preferred address row per customer using a deterministic rule:
    - prioritize `address_type = 'Main Office'`
    - else `address_type = 'Home'`
    - else alphabetical `address_type`
    - tie-break by latest `modified_date`
- Grain: one row per customer.
- Key: `customer_key` surrogate, with `customer_id` retained as natural key.
- Relevant attributes:
  - customer name/display name
  - company_name
  - email_address
  - phone
  - sales_person_clean
  - city
  - state_province
  - country_region
  - postal_code
  - preferred_address_type
- Explicitly exclude:
  - password_hash
  - password_salt
  - rowguid-heavy technical fields not needed for analytics
- Hierarchy where applicable:
  - country_region > state_province > city > customer
  - company/customer rollup if business-friendly names are stable

#### `gold.dim_sales_person`
- Source: distinct cleaned values from `silver.customer.sales_person_clean`
- Grain: one row per salesperson.
- Key: `sales_person_key` surrogate; retain cleaned name as business key where unique.
- Cleaning rule per user:
  - when stored as `<domain>/username` or `<domain>\username`, extract username
  - format display value in proper case
- Attributes:
  - sales_person_name
  - sales_person_raw
  - sales_person_username
- Note: this dimension is indirectly linked to sales through customer on the header/order path, because salesperson exists on customer rather than on order detail/header columns.

#### `gold.dim_order`
- Source: `silver.sales_order_header`
- Grain: one row per sales order.
- Key: `order_key` surrogate; retain `sales_order_id` as natural key.
- Move suitable dimensional attributes out of the fact:
  - sales_order_number
  - purchase_order_number
  - account_number
  - revision_number
  - status / status_label
  - online_order_flag
  - ship_method
  - credit_card_approval_code
  - comment
- Keep foreign keys to customer/order dates/ship dates in the fact rather than embedding all relationships here.
- Hierarchy where applicable:
  - status groupings are limited; no deep natural hierarchy beyond order number drill if desired.

#### `gold.dim_product`
- Source assembly per user requirement:
  - `silver.product`
  - `silver.product_category`
  - `silver.product_description` (Description only)
  - `silver.product_model` (Name only as `model_name`)
  - `silver.product_model_product_description` filtered to `culture = 'en'`
- Recommended join path using actual schema:
  - `product.product_category_id` -> `product_category.product_category_id`
  - `product.product_model_id` -> `product_model.product_model_id`
  - `product_model.product_model_id` -> `product_model_product_description.product_model_id` filtered `culture = 'en'`
  - `product_model_product_description.product_description_id` -> `product_description.product_description_id`
- Flatten category parent-child into:
  - `category_name`
  - `subcategory_name`
  using self-join on `product_category.parent_product_category_id`
- Grain: one row per product.
- Key: `product_key` surrogate; retain `product_id` as natural key.
- Relevant attributes:
  - product name
  - product number
  - color
  - size
  - weight
  - standard_cost
  - list_price
  - model_name
  - english description
  - category_name
  - subcategory_name
  - sell_start_date / sell_end_date / discontinued_date or derived current flags if useful
- Hierarchies:
  - category_name > subcategory_name > product name
  - model_name > product name if useful for drill

### Fact

#### `gold.fact_sales_order`
- Required by user: combine `SalesOrderHeader` and `SalesOrderDetail`, keeping only relevant measures.
- Grain: one row per sales order line (`sales_order_id`, `sales_order_detail_id`).
- Base join:
  - `silver.sales_order_detail.sales_order_id` -> `silver.sales_order_header.sales_order_id`
- Foreign keys:
  - `order_key`
  - `customer_key`
  - `sales_person_key`
  - `product_key`
  - `order_date_key`
  - `ship_date_key`
- Degenerate/source identifiers retained:
  - `sales_order_id`
  - `sales_order_detail_id`
- Measures to store:
  - `order_qty`
  - `unit_price`
  - `unit_price_discount`
  - `gross_sales_amount`
  - `discount_amount`
  - `line_total` as net sales
  - optional allocated header amounts if needed later:
    - `header_subtotal`
    - `header_tax_amt`
    - `header_freight`
    - `header_total_due`
- Modeling guidance:
  - Prefer `line_total` as the primary net sales measure because it exists at detail grain.
  - Do not sum header totals blindly at line grain without either allocation or distinct-order measures; if included, label them clearly as header-level repeated values.
- Regional analytics path:
  - fact -> dim_customer -> geography attributes for map visuals
- Discount/salesperson analytics path:
  - fact -> dim_sales_person through customer-derived salesperson mapping

## Test
After Gold writes, run tests and append one row per test execution to `test.test_results` with schema:
`run_id, test_name, layer, table_name, status, actual, expected, details, checked_at`

Standard tests to implement:

1. Row count reconciliation: Bronze vs Silver
- For each major base table (`customer`, `address`, `product`, `sales_order_header`, `sales_order_detail`)
- Expectation: silver count is approximately bronze count within 1%, allowing only dedup/cleaning variance.
- Append one result row per table.

2. Row count reconciliation: Silver to Gold fact coverage
- Compare `silver.sales_order_detail` count to `gold.fact_sales_order` count.
- Expectation: exact match unless filtered records are intentionally excluded; if so, details must state why.

3. No-null PKs in Gold dimensions
- `dim_order_date.order_date_key`
- `dim_ship_date.ship_date_key`
- `dim_customer.customer_key`
- `dim_sales_person.sales_person_key`
- `dim_order.order_key`
- `dim_product.product_key`
- Expectation: zero nulls.

4. Unique PKs in Gold dimensions
- Assert uniqueness of each Gold dimension primary key.
- Expectation: total rows equals distinct PK rows.

5. Referential integrity for fact foreign keys
- Every `fact_sales_order.product_key` exists in `dim_product`
- Every `fact_sales_order.customer_key` exists in `dim_customer`
- Every `fact_sales_order.sales_person_key` exists in `dim_sales_person`
- Every `fact_sales_order.order_key` exists in `dim_order`
- Every `fact_sales_order.order_date_key` exists in `dim_order_date`
- Every non-unknown `fact_sales_order.ship_date_key` exists in `dim_ship_date`

6. Business-rule sanity check tailored to this data
- Discount percentage sanity:
  - `discount_pct = discount_amount / gross_sales_amount`
  - Expectation: all non-null discount percentages are between 0 and 1.
- Also validate `line_total <= gross_sales_amount` for non-negative sales lines.
- This directly supports the requested discount analysis.

7. Regional completeness sanity check
- For customers referenced by facts, calculate percent with non-null country/state/city in `dim_customer`.
- Expectation: high completeness target, e.g. at least 95% populated for mappable customers; if lower, mark FAIL with count details.

## Semantic model
Create a Direct Lake semantic model over the Gold star schema.

Model tables:
- `dim_order_date`
- `dim_ship_date`
- `dim_customer`
- `dim_sales_person`
- `dim_order`
- `dim_product`
- `fact_sales_order`

Relationships:
- `fact_sales_order[order_date_key]` -> `dim_order_date[order_date_key]` (active)
- `fact_sales_order[ship_date_key]` -> `dim_ship_date[ship_date_key]` (inactive if needed for alternate date analysis, or active only in ship-date views depending on tooling pattern)
- `fact_sales_order[customer_key]` -> `dim_customer[customer_key]`
- `fact_sales_order[sales_person_key]` -> `dim_sales_person[sales_person_key]`
- `fact_sales_order[order_key]` -> `dim_order[order_key]`
- `fact_sales_order[product_key]` -> `dim_product[product_key]`

Hierarchies to create on all applicable dimensions:
- `dim_order_date`: Year > Quarter > Month > Date
- `dim_ship_date`: Year > Quarter > Month > Date
- `dim_customer`: Country > State/Province > City > Customer
- `dim_product`: Category > Subcategory > Product
- `dim_sales_person`: optional display hierarchy if username/display/raw are retained
- `dim_order`: optional order drill hierarchy by status > order number if useful

Explicit measures:
- `Sales Amount = SUM(fact_sales_order[line_total])`
- `Gross Sales Amount = SUM(fact_sales_order[gross_sales_amount])`
- `Discount Amount = SUM(fact_sales_order[discount_amount])`
- `Discount % = DIVIDE([Discount Amount], [Gross Sales Amount])`
- `Average Sales Amount = AVERAGE(fact_sales_order[line_total])`
- `Max Sales Amount = MAX(fact_sales_order[line_total])`
- `Order Count = DISTINCTCOUNT(fact_sales_order[sales_order_id])`
- `Sales Order Line Count = COUNTROWS(fact_sales_order)`
- `Average Order Quantity = AVERAGE(fact_sales_order[order_qty])`
- `Max Order Line Value = MAX(fact_sales_order[line_total])`

Recommended additional measures for user intent:
- `Monthly Sales Amount = [Sales Amount]` used with order-date month hierarchy
- `Average Discount % = AVERAGE(fact_sales_order[unit_price_discount])` if line-level rate analysis is desired, but prefer weighted `Discount %` for executive reporting
- `Max Discount % = MAX(fact_sales_order[unit_price_discount])`
- `Shipped Order Count = CALCULATE([Order Count], NOT ISBLANK(dim_ship_date[date]))` if date table design supports it

Modeling notes:
- Use `Sales Amount`, `Average Sales Amount`, and `Max Sales Amount` prominently because the user explicitly requested average and maximum sales metrics.
- For salesperson discount analysis, rank `dim_sales_person` by `[Discount Amount]` and `[Discount %]`.
- Hide surrogate keys and technical lineage fields from report view.
- Exclude sensitive customer fields entirely from the semantic model.

## Report
Create a Power BI report tailored to the requested sales reporting solution.

### Page 1 — Regional Performance
Purpose: quickly identify high- and low-performing regions.
Visuals:
- Filled map or bubble map using `dim_customer[country/state/city]` and `[Sales Amount]`
- Choropleth/map alternative by state/province if geography quality supports it
- Clustered bar chart: Sales Amount by `state_province`
- Table or matrix: Country > State > City with:
  - Sales Amount
  - Average Sales Amount
  - Max Sales Amount
  - Order Count
  - Discount %
- KPI cards:
  - Sales Amount
  - Average Sales Amount
  - Max Sales Amount
  - Order Count
Slicers:
- Order Date hierarchy
- Ship Date hierarchy
- Country/State
- SalesPerson

### Page 2 — Orders and Monthly Trends
Purpose: show order activity and time trends.
Visuals:
- Line chart: monthly `Sales Amount` by `dim_order_date` hierarchy
- Line and clustered column chart:
  - columns = `Sales Amount`
  - line = `Order Count`
  by month
- Matrix by `dim_order[status_label]` and `ship_method` with:
  - Sales Amount
  - Average Sales Amount
  - Max Sales Amount
  - Order Count
- Detail table for orders:
  - sales_order_id / order number
  - customer
  - ship method
  - status
  - Sales Amount
  - Discount Amount
  - Discount %
