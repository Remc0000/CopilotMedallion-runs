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

### Iteration 3 — 2026-07-30 12:04:53Z — failed layer: silver (run: 20260730-115433-6a13fc)
- **Root cause (1-line summary)**: A Silver Spark statement failed and Fabric terminally cancelled the session; the deferred-continue error pattern then provided only `System_Cancelled_Session_Statements_Failed` instead of the first failing table, action, and exception.
- **Cross-table audit**:
  - `customeraddress`: yes — its read, deduplication, write, or read-back action can be the first statement failure, especially because it uses a composite key and optional audit columns.
  - `salesorderdetail`: yes — its validation, deduplication, write, or count action can cancel the shared session.
  - `productdescription`: yes — its read, transformation, write, or verification action runs in the same cancellable session.
  - `customer`: yes — its sensitive-column projection, salesperson normalization, write, or verification action can fail.
  - `productcategory`: yes — its key validation, transformation, write, or verification action can fail.
  - `productmodel`: yes — its key validation, transformation, write, or verification action can fail.
  - `salesorderheader`: yes — its date/decimal validation, partition-related input shape, write, or verification action can fail.
  - `productmodelproductdescription`: yes — its composite-key deduplication and optional-audit handling can cause the same terminal session failure.
  - `product`: yes — its derived-column chain, required-key validation, write, or verification action can fail.
  - `address`: yes — its normalization, key validation, write, or verification action can fail.
- **Fix approach**: GENERALIZE — the traceback is a session-level wrapper with no table-specific root exception, and every source table executes Spark statements in the same Silver session; therefore all ten tables need the same named-action diagnostics, pre-write materialization checkpoint, and terminal fail-fast behavior.
- **What was changed**:
  - Tightened `## Generic guidance` so a Spark statement exception is treated as terminal for that session: record the original table/action/traceback without additional Spark work and immediately re-raise.
  - Tightened `## Silver` to require structured `START`/`SUCCESS`/`FAIL` diagnostics around each Spark action and a guarded pre-write materialization checkpoint for every table.
  - Replaced deferred continuation after Spark execution errors with fail-fast behavior; deferred aggregation remains permitted only for non-terminal validation findings while an explicit Spark health probe succeeds.

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
- Do not log secrets or retain `password_hash`, `password_salt`, or `credit_card_approval_code` beyond the source-preserving Bronze layer.
- Every generated notebook code cell must begin with a Python `# ---` divider and one to three human-readable `# ` comment lines explaining what the cell does and why. Never emit an uncommented code cell.
- Treat any exception raised by a Spark action or Spark SQL statement as potentially terminal in Microsoft Fabric. After such an exception, do not issue another Spark read, write, count, collect, SQL command, catalog query, or Spark-backed error-log write in that session.
- Wrap every Spark action in a named action boundary that prints a structured `START` record before execution and a `SUCCESS` or `FAIL` record afterward. A `FAIL` record must include `run_id`, `layer`, `table`, `action`, exception class, exception message, and `traceback.format_exc()`.
- On a Spark action exception, preserve and immediately re-raise the original exception. Call `_save_error` only when it is implemented without Spark and cannot mask the original failure; otherwise print the structured failure record and skip `_save_error`.
- Deferred error aggregation is allowed only for validation findings that do not indicate a failed Spark statement and only while a guarded health probe such as `spark.range(1).count()` succeeds. Never continue a table loop after `AnalysisException`, `Py4JJavaError`, `Py4JNetworkError`, a cancelled-job/session error, or any failed Spark SQL/DataFrame action.

### Global Spark column-reference rules (apply to ALL layers: Bronze, Silver, Gold)
These rules exist to prevent recurring `UNRESOLVED_COLUMN` / `AnalysisException` analyzer errors. They are layer-agnostic — apply them anywhere a Spark DataFrame is transformed.

Rule A — No dotted alias strings.
- Never pass dotted strings like "c.customer_id", "ca.address_type", "h.sales_person", or "pc_child.name" to F.col(...), withColumn(...), Window.partitionBy(...), Window.orderBy(...), or select(...). Spark treats "c.customer_id" as a single column literally named c.customer_id, which does not resolve once any projection or rename has been applied.
- Alias scope (`.alias("c")`, `.alias("ca")`, etc.) is only valid inside the SAME select/join expression that introduces it. Once you produce a new DataFrame via `select(...)` or `withColumn(...)`, the dotted alias form is gone and you must reference plain column names.

Rule B — Materialize helper columns before they are needed downstream.
- For any column that will later be referenced by a Window, a `withColumn`, or a downstream join after a projection, first materialize it as a flat, unambiguous helper column such as `rank_customer_id`, `rank_address_type`, or `sales_person_source` in the same select that introduces the join aliases.

Rule C — Do not drop a column before its last consumer has run.
- Before adding a `withColumn`, verify every `F.col(...)` referenced by that expression still exists on the DataFrame at that step. If a previous `select(...)` projection removed it, either:
  - Move the `withColumn` before the projection that drops the source column.
  - Keep the source column in the projection.
  - Re-derive the value from a column that is still present, often a flag computed earlier from the same source.
- Avoid dropping `discontinued_date` and then referencing it in a later expression. When `is_discontinued` has already been derived, use that flag rather than reaching back to the dropped raw column.

Rule D — Order of derived-column computations matters.
- When one derived column depends on another, add them in dependency order with sequential `withColumn` calls and reference the already-derived column rather than a raw source column that may have been dropped.

Rule E — Validate schema between non-trivial transformation steps.
- After any `select(...)`, `drop(...)`, or substantial `withColumn` chain, and before the next named-column consumer, assert that every required column exists. Fail with a message naming the missing column, table, and DataFrame stage.

Rule F — Self-check pattern for every withColumn / Window.
- For every `withColumn(name, expr)` and Window definition, confirm that every referenced column exists on the DataFrame at that exact point. Correct the transformation order before executing any action if it does not.

Rule G — Optional-column helpers must return typed Column nulls, not Python None.
- A helper such as `_maybe(df, name)` must never return Python `None` when its result can be passed into a Spark function.
- Use a typed Spark null:
  ```
  def _maybe(df, name, dtype="timestamp"):
      return F.col(name) if name in df.columns else F.lit(None).cast(dtype)
  ```
- Choose a type compatible with the surrounding expression. Alternatively, remove Python `None` values from a candidate list before invoking a Spark function.
- Never pass Python `None` directly to `F.coalesce`, `F.greatest`, `F.least`, `F.concat`, `F.when(...).otherwise(...)`, or another Spark expression.

Rule H — Isolate tables, but treat a failed Spark statement as terminal.
- Process source tables sequentially in `for tbl in source_tables:` and keep each iteration self-contained: read → transform → schema validation → materialization checkpoint → write → read-back verification → result.
- Do not build one multi-table Spark SQL statement or one chained DataFrame plan that touches all source tables. Do not share temp views between table iterations.
- Per-table isolation prevents unrelated lazy plans from being combined, but it cannot restore a Fabric Spark session after a statement failure. If a Spark action fails, print the original table/action traceback and immediately re-raise without attempting the remaining tables.
- Continue to another table only after the current table's actions succeeded, or after a non-Spark validation finding for which an explicit Spark health probe succeeds.
- Cross-table joins belong in a separate downstream phase after all required source-aligned tables have been written and verified.

Rule I — Optional audit columns on junction / bridge / view tables.
- Entity tables may contain `ModifiedDate` or `rowguid`, while junction, bridge, or view-shaped tables may not.
- Do not assume `modified_date` or another audit column exists on every table. Guard with membership in `df.columns`.
- For deduplication without audit columns, use only the declared natural/composite key as the deterministic ordering input.
- For `source_dt` or `_silver_ts`, use a typed null timestamp or `F.current_timestamp()`.
- Junction tables must deduplicate on their composite foreign-key grain rather than a nonexistent surrogate key.
- Views must project only columns actually returned by the view.

Rule J — Validate column existence before the expensive transform.
- Before every join, `withColumn`, `groupBy`, aggregation, filter, Window, or projection that names a column, assert that column exists:
  ```
  for required in ("customer_id", "order_date"):
      if required not in df.columns:
          raise RuntimeError(
              f"[{layer}] {tbl}: required column '{required}' missing; available={df.columns}"
          )
  ```
- Repeat validation after a `select`, `drop`, or rename step and before the next consumer.

Rule K — Resilience to partial output: every layer must write discoverable Delta tables.
- Create the target schema and write schema-qualified tables with `saveAsTable`.
- Bronze writes `bronze.<table>`, Silver writes `silver.<table>`, and Gold writes `gold.<table>` plus `test.test_results`.
- Never use path-only ABFSS writes as a substitute for catalog registration.
- Print a final machine-readable summary listing every successfully written table.
- Raise if zero required tables were written. Missing tables must never be treated as successful completion.

Rule L — Disambiguate shared columns in join projections.
- When joined inputs share a column name, never select that column as a bare string.
- In the immediate join projection, qualify the desired side and rename it explicitly, for example `F.col("p.product_model_id").alias("product_model_id")`.
- Enumerate overlapping names before creating the projection. In a multi-table join, alias-qualify every selected column when practical.
- Dotted alias references are valid only in the immediate join expression and projection. Use the resulting flat names afterward.

Rule M — Named Spark-action diagnostics and pre-write checkpoints.
- Give every Spark action an explicit name, including at least `read_input`, `materialize_transform`, `write_delta`, `catalog_exists`, `read_back`, `describe_detail`, `count_read_back`, and `final_catalog_scan`.
- Print the `START` diagnostic before the action so the last emitted record identifies the action even if Fabric cancels the session before Python receives a detailed exception.
- Before overwriting a target table, execute a guarded materialization checkpoint such as `silver_df.limit(1).collect()` after required-column validation. This forces lazy analysis and a representative transformation before the destructive write begins.
- If the materialization checkpoint fails, do not execute the write.
- Never place a Spark action inside a `finally` block. Cleanup and summary code after a Spark failure must be Python-only.

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
  1. Parameters, imports, run metadata, expected table list, result collections, structured diagnostic helpers, and Python-only failure-record helpers.
  2. `silver` schema creation, a Spark health probe, and validation that all expected `bronze.<table>` inputs are catalog-readable.
  3. The sequential per-table read → snake_case transform → schema validation → deduplication → materialization checkpoint → write → read-back verification loop.
  4. Final `SHOW TABLES IN silver` comparison, machine-readable summary, and a raise for validation failures or missing tables while the Spark session remains healthy.
- Every cell must begin with the required Python `# ---` divider and explanatory comments. Additional cells may be used, but these responsibilities must remain executable and cannot be replaced by narrative.
- If response size or implementation complexity is constrained, prioritize the required cells and catalog writes. Omit or defer `OPTIMIZE` before omitting any required transformation, checkpoint, write, diagnostic, or verification.
- Create the schema with `spark.sql("CREATE SCHEMA IF NOT EXISTS silver")` before processing tables.
- Use this exact expected table set: `customeraddress`, `salesorderdetail`, `productdescription`, `customer`, `productcategory`, `productmodel`, `salesorderheader`, `productmodelproductdescription`, `product`, and `address`.
- At notebook start, run a named `session_health_probe` action using `spark.range(1).count()` and require the result to be `1`.
- Before each Spark action, print one compact JSON `START` record containing `run_id`, `layer="silver"`, `table`, and `action`. After success, print the corresponding `SUCCESS` record.
- Required named action boundaries per table are:
  - `read_bronze`
  - `materialize_transform`
  - `write_silver`
  - `catalog_exists`
  - `read_back`
  - `describe_detail`
  - `count_read_back`
- The final layer actions must be named `show_tables_silver` and `final_health_probe`.
- Process tables sequentially. For each `tbl`, read `bronze.<tbl>`, create the source-aligned snake_case DataFrame, validate its required columns, deduplicate it, and execute `silver_df.limit(1).collect()` as `materialize_transform` before the write.
- The materialization checkpoint is mandatory even for an empty table. It must analyze the full projection and transformation plan before `saveAsTable`; a zero-row result is valid.
- If `materialize_transform` fails, print the original exception class, message, and full Python traceback with `table=tbl` and `action="materialize_transform"`, then immediately re-raise. Do not attempt the write, another table, a catalog scan, `OPTIMIZE`, or any other Spark operation.
- Write only as a managed/catalog Delta table:
  ```
  silver_df.write.format("delta").mode("overwrite").option(
      "overwriteSchema", "true"
  ).saveAsTable(f"silver.{tbl}")
  ```
- Do not use `.save(path)`, ABFSS output paths, unqualified `saveAsTable(tbl)`, temporary views, global temporary views, or in-memory DataFrames as substitutes for `silver.<tbl>`.
- Immediately after each successful write, verify:
  - `spark.catalog.tableExists(f"silver.{tbl}")` is true.
  - `spark.table(f"silver.{tbl}")` is readable.
  - `DESCRIBE DETAIL silver.<tbl>` reports `format = 'delta'`.
  - The read-back schema contains the required key columns.
  - The read-back row count and fully qualified target name are recorded in `silver_results`.
- A valid empty source may produce a zero-row Silver table, but the catalog table must still be created and verified.
- For any exception from a Spark read, checkpoint, write, SQL statement, count, collect, or catalog operation:
  - Emit a Python `FAIL` JSON record and `traceback.format_exc()` immediately.
  - Preserve the table and named action that failed.
  - Do not run another Spark statement, including Spark-backed `_save_error`, result-table writes, health probes, catalog scans, or cleanup.
  - Re-raise the original exception immediately so the traceback preceding Fabric's cancellation wrapper remains visible.
- `_save_error('silver', e, table=tbl)` may run after a Spark exception only if it is proven to be Python/REST-only and does not access Spark. Wrap it in a Python-only best-effort block and never allow it to replace the original exception.
- Continue to the next table only after all actions for the current table succeeded. Deferred all-table error accumulation applies only to non-terminal business-validation findings while `spark.range(1).count()` confirms the session remains usable; it does not apply to failed Spark statements.
- Create source-aligned snake_case tables. Trim strings, normalize blank strings to null where appropriate, preserve valid decimals/timestamps, and add `_silver_run_id`, `_silver_ts`, and `_source_modified_at`.
- Deduplicate by these keys, retaining the greatest `modified_date`, then `rowguid`, when those columns exist:
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
- For `customeraddress` and `productmodelproductdescription`, do not require `modified_date` or `rowguid`. If absent, deduplicate deterministically using only the listed composite key.
- For every table, assert its listed key columns after snake_case projection, immediately before the materialization checkpoint, and in the read-back table.
- Validate foreign keys without dropping unresolved rows: order header to customer and bill/ship address; order detail to header and product; product to category/model; product-description bridge to model/description; customer-address bridge to customer/address.
- Normalize `customer.sales_person` into `sales_person_username`: trim, replace `/` with `\` if encountered, take the text after the final backslash, and lowercase. For example, `adventure-works\jillian0` becomes `jillian0`; do not remove trailing digits.
- Exclude password hash/salt and credit-card approval code from Silver.
- Filter invalid negative `order_qty`, `unit_price`, `unit_price_discount`, or header monetary amounts into an error/quarantine result rather than silently correcting them. Permit null optional dates and descriptive attributes.
- Run `OPTIMIZE` only after all required tables have been successfully written, read back, and included in `silver_results`. Give every `OPTIMIZE` statement its own named action boundary. On any `OPTIMIZE` Spark exception, emit the original traceback and fail immediately without another Spark statement.
- After all successful per-table attempts, run the named `final_health_probe`. Only if it succeeds, execute `SHOW TABLES IN silver` and compare discovered names with the exact expected set.
- Treat final table-name comparison as case-insensitive, but require every expected table. Report both `missing_tables` and `unexpected_tables`.
- Print a machine-readable summary containing `silver_results`, expected tables, discovered tables, and `missing_tables`.
- If a non-terminal validation finding remains, or an expected Silver table is missing, record it and raise `RuntimeError` only after final catalog validation. Never allow successful completion with zero or partial discoverable Silver tables.
- Do not attempt to “finish the loop” after a Spark execution exception. In Fabric, preserving the first table/action traceback and stopping immediately takes precedence over deferred attempts because the cancelled session cannot safely execute the remaining statements.

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