# Run Spec 20260517-144043-a594f6

## Updated specs

### Iteration 1 — 2026-05-17 14:52:28Z — failed layer: gold (run: 20260517-144627-02c2b7)
- **Root cause (1-line summary)**: Gold build likely failed due to under-specified joins/column names in the requested custom Gold tables (`dim_customer`, `dim_product`, `dim_sales`, `fact_sales`), especially around self-join/category hierarchy and flattened customer-address logic.
- **What was changed**:
  - Tightened `## Gold` to define the exact required Gold tables, their grains, deterministic flattening rules, join paths, aliases, and output column names.
  - Added explicit rules for the product category self-join and english-only product description selection, plus cleaned salesperson derivation logic.
  - Tightened `## Semantic model` to align strictly to the four requested Gold tables and their exact join keys/field names.

### Iteration 2 — 2026-05-17 15:03:15Z — failed layer: gold (run: 20260517-144627-02c2b7)
- **Root cause (1-line summary)**: Gold layer remains vulnerable to unresolved/incorrect column references because several required Gold fields were specified optimistically rather than constrained to validated SalesLT Silver columns.
- **What was changed**:
  - Tightened `## Generic guidance` with a new rule to validate actual source/Silver schemas before coding Gold logic and to avoid assuming optional SalesLT columns exist.
  - Tightened `## Gold` with explicit fallback/derivation rules for salesperson fields and stricter column contracts for `sales_order_header`, `customer`, and `product_model`.
  - Added mandatory implementation guidance to build `dim_sales` and `fact_sales` from a shared cleaned-salesperson expression and to use only explicitly listed output columns.

### Iteration 3 — 2026-05-17 15:13:41Z — failed layer: gold (run: 20260517-144627-02c2b7)
- **Root cause (1-line summary)**: UNRESOLVED_COLUMN on `c.customer_id` in `gold.dim_customer` windowing because the row-number ranking was applied after alias-qualified columns had already been projected/renamed away.
- **What was changed**:
  - Tightened `## Generic guidance` to forbid referencing source aliases after a projection that removes those aliases, especially in window specs, filters, and later transforms.
  - Tightened `## Gold` / `gold.dim_customer` with a mandatory two-step build pattern: rank on alias-qualified join output first, then project final renamed columns after filtering to rank 1.
  - Added an explicit window-column contract for `dim_customer` using only `c.customer_id`, `ca.address_type`, and `ca.address_id` before the final select.

### Iteration 1 — 2026-05-17 15:16:19Z — failed layer: gold (run: 20260517-144627-02c2b7)
- **Root cause (1-line summary)**: Gold run still failed at session level because Gold transformations remain under-constrained for execution safety; the build must stage and validate each Gold table independently before continuing.
- **What was changed**:
  - Tightened `## Generic guidance` to require fail-fast, table-by-table Gold validation and to stop immediately on the first Gold table error instead of continuing until session cancellation.
  - Tightened `## Gold` with a mandatory build order, pre-write validation checks, and explicit row-level uniqueness/null-key assertions for each of the four required Gold tables.
  - Added an explicit implementation rule that each Gold table must be materialized from its own final DataFrame and validated before starting the next table.

### Iteration 2 — 2026-05-17 15:22:53Z — failed layer: gold (run: 20260517-144627-02c2b7)
- **Root cause (1-line summary)**: Gold execution still reached session cancellation because the spec did not force per-table schema assertions and immediate fail-fast checkpoints before and after each Gold transformation step.
- **What was changed**:
  - Tightened `## Generic guidance` to require explicit schema/row-count checkpoints after each intermediate Gold step and immediate notebook termination on the first failed assertion.
  - Tightened `## Gold` with mandatory per-table preflight checks, intermediate step names, and write verification for `dim_customer`, `dim_product`, `dim_sales`, and `fact_sales`.
  - Added a stricter rule that no later Gold table may start until the prior table has passed source-column validation, final-column validation, key validation, and post-write readability verification.

### Iteration 1 — 2026-05-17 15:42:02Z — failed layer: silver (run: 20260517-144627-02c2b7)
- **Root cause (1-line summary)**: Silver layer failed at session level because Silver transformations were under-constrained and not required to execute with per-table fail-fast schema validation and blocking checkpoints.
- **What was changed**:
  - Tightened `## Generic guidance` with a new cross-cutting rule to build Silver table-by-table with immediate termination on the first Silver validation/write failure.
  - Tightened `## Silver` to define an exact Silver build order, required per-table checkpoints, and mandatory source/final schema assertions before and after each write.
  - Added explicit Silver table column contracts and derived-column dependency rules so transforms only reference validated `snake_case` columns that still exist at each step.

### Iteration 2 — 2026-05-17 15:46:49Z — failed layer: silver (run: 20260517-144627-02c2b7)
- **Root cause (1-line summary)**: Silver still failed at session level because the spec did not force lightweight execution checkpoints after each intermediate Silver step, allowing lazy analyzer/runtime failures to accumulate until Fabric cancelled the session.
- **What was changed**:
  - Tightened `## Generic guidance` to require a lightweight Spark action after every non-trivial Silver intermediate DataFrame and immediate termination on the first failing Silver table.
  - Tightened `## Silver` to mandate named intermediate DataFrames, exact checkpoint actions, and per-table preflight/post-write validation sequencing.
  - Added an explicit no-batching rule: do not build multiple Silver tables or defer multiple writes/actions within the same unresolved execution chain.

### Iteration 3 — 2026-05-17 15:49:13Z — failed layer: silver (run: 20260517-144627-02c2b7)
- **Root cause (1-line summary)**: Silver failed again at session level because view-based Silver tables were under-specified; the notebook must not infer renamed view columns or defer schema discovery for `v_*` sources.
- **What was changed**:
  - Tightened `## Generic guidance` to require immediate schema introspection and persisted-column freezing for any source view before dedup/final projection logic is generated.
  - Tightened `## Silver` with an explicit build pattern for `silver.v_get_all_categories`, `silver.v_product_and_description`, and `silver.v_product_model_catalog_description` using actual renamed columns only, no assumed business-key columns before validation.
  - Added a hard rule that view-based Silver tables must derive dedup keys from validated renamed columns only; if the expected key column is absent after rename, fail immediately with the actual column list.

### Iteration 4 — 2026-05-17 15:50:57Z — failed layer: bronze (run: 20260517-144627-02c2b7)
- **Root cause (1-line summary)**: Bronze build reached session cancellation because Bronze ingestion was under-constrained and did not require source-by-source fail-fast validation, especially for source views and partition helper columns.
- **What was changed**:
  - Tightened `## Generic guidance` with Bronze-specific blocking execution, lightweight checkpoints, and immediate termination on first Bronze failure.
  - Tightened `## Bronze` to define exact Bronze build order, mandatory per-object preflight/post-write validations, and explicit source schema handling for base tables vs `v_*` views.
  - Added an explicit rule for `bronze.sales_order_header` to derive and retain `order_year` only after validating `OrderDate`, and to never assume view key columns during Bronze landing.

## Inputs
- Workspace: `d97c0ec7-74ac-4f12-9e70-124cb0d5a4ad`
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
- Target Lakehouse: **CopilotMedallion_20260517_144043**

## Generic guidance
Apply these reference skills/agents at all times:
- FabricDataEngineer agent: https://github.com/microsoft/skills-for-fabric/blob/main/agents/FabricDataEngineer.agent.md
- e2e-medallion-architecture skill: https://github.com/microsoft/skills-for-fabric/tree/main/skills/e2e-medallion-architecture
- spark-authoring-cli skill: https://github.com/microsoft/skills-for-fabric/tree/main/skills/spark-authoring-cli
- powerbi-authoring-cli skill: https://github.com/microsoft/skills-for-fabric/tree/main/skills/powerbi-authoring-cli
- powerbi-consumption-cli skill: https://github.com/microsoft/skills-for-fabric/tree/main/skills/powerbi-consumption-cli
- powerbi-semantic-model-authoring: https://github.com/RuiRomano/powerbi-agentic-plugins/tree/main/plugins/powerbi/skills/powerbi-semantic-model-authoring
- powerbi-report-authoring: https://github.com/RuiRomano/powerbi-agentic-plugins/tree/main/plugins/powerbi-report-authoring

Cross-cutting code rules:
- Use defensive column references everywhere; assert required columns exist before select, join, dedup, rename, filter, aggregate, and write.
- After every join, immediately project to an explicit alias-prefixed/select-renamed column list to prevent ambiguous names from propagating, unless a downstream window/filter in the same logical build step still needs alias-qualified columns; in that case, keep the needed alias-qualified columns through the ranking/filter step and only then perform the final renamed projection.
- Before every `groupBy`/`agg`, assert all grouping and aggregation input columns exist.
- For REST/API responses, use defensive handling and raise early: `if x is None: raise ...` before any `.get()` access.
- Never use `saveAsTable`; write using idempotent Delta overwrite/merge patterns against paths or managed table APIs supported by Fabric notebooks.
- Put notebook parameters in dedicated parameter cells at the top of each notebook.
- Use idempotent overwrite patterns for full loads and deterministic merge/upsert patterns where incremental logic is introduced.
- All `try/except` blocks must fail loudly: call `_save_error(layer, e)` and then `raise`.
- Add audit/control columns consistently across layers, including load timestamp, run id, source system/table, and record hash where useful.
- Optimize physical layout after writes using partition-aware maintenance only where justified by the actual data shape; avoid unnecessary partitioning on small dimensions.
- Treat source views (`vGetAllCategories`, `vProductAndDescription`, `vProductModelCatalogDescription`) as read-only source entities; do not assume they are canonical over base tables unless explicitly chosen in Gold.
- In Gold, do not infer column names from business descriptions. Use only validated Silver column names after `snake_case` conversion and fail early if an expected column is absent.
- In Gold self-joins and bridge-based enrichments, every input DataFrame must be aliased and every selected output column must be explicitly renamed to its final Gold name in the same projection step.
- Before implementing any Gold table, inspect the actual Silver schema and reconcile it to this spec. If a field in the business request is not a validated Silver column, derive it only from the explicit fallback rules in this spec; otherwise fail early with a clear missing-column error instead of guessing an alternative column name.
- Never reference an alias-qualified column name after a projection/select step that has removed or renamed that alias-qualified column. This applies especially to `Window.partitionBy`, `Window.orderBy`, `filter`, `withColumn`, and `dropDuplicates`. If a later step needs `c.customer_id`, `ca.address_type`, or similar alias-qualified fields, perform that later step before the final renaming/projection.
- In Gold, build and validate one table at a time in this exact order: `gold.dim_customer`, then `gold.dim_product`, then `gold.dim_sales`, then `gold.fact_sales`. Do not start the next Gold table until the current one has passed schema, null-key, and uniqueness checks and has been written successfully.
- For each Gold table, create a dedicated final DataFrame variable containing exactly the persisted output columns for that table. Validate that final DataFrame before write, write it, then clear/release intermediate DataFrames before building the next table.
- If any Gold table fails validation or write, stop the notebook immediately after `_save_error('gold', e)` and `raise`; do not continue attempting later Gold tables, because that can lead to session-wide statement failure/cancellation.
- In Gold, add explicit fail-fast checkpoints after each intermediate build step: source read, join result, ranking/dedup result, final projection, pre-write validation, and post-write readback. Each checkpoint must assert expected columns exist; for ranking/dedup/final steps it must also assert the DataFrame is instantiable and countable without error.
- Gold notebook control flow must be strictly linear and blocking: build one table, validate it, write it, read it back once to verify schema/readability, then and only then proceed to the next table. Never queue or lazily defer multiple Gold actions before validating the current table.
- In Silver, build and validate one table at a time in the exact order defined in `## Silver`. Do not start the next Silver table until the current one has passed source-column validation, transform-step validation, final-schema validation, key validation, write success, and post-write readback.
- If any Silver table fails validation or write, stop the notebook immediately after `_save_error('silver', e)` and `raise`; do not continue attempting later Silver tables.
- In Silver, every intermediate transformation step that changes schema shape (`select`, `drop`, `withColumn`, dedup/ranking, aggregation, join) must be followed by an explicit schema assertion naming the DataFrame variable and the required columns for the next step.
- In Silver, after every non-trivial intermediate DataFrame is created, immediately force a lightweight Spark action before moving on. Use one of these checkpoint actions only: `limit(1).collect()`, `take(1)`, or `count()` when row-count validation is explicitly required. This applies to post-rename DataFrames, post-derived-column DataFrames, ranked/deduped DataFrames, final projection DataFrames, and post-write readback DataFrames.
- Do not batch unresolved Silver work: never create multiple Silver table DataFrames and then trigger actions later. Finish all validations, checkpoint actions, and the write/readback for the current Silver table before reading or transforming the next Silver source table.
- Silver notebook control flow must be strictly linear and blocking: read one Bronze table, build one Silver table, validate it, checkpoint it with a lightweight action, write it, read it back once to verify schema/readability, then and only then proceed to the next Silver table.
- For Bronze or Silver source views, always inspect the actual source schema first, then perform `snake_case` rename, then persist exactly the validated renamed column list. Do not assume key column names for a view until after that inspection and rename have completed.
- For any `v_*` table in Silver, freeze the renamed column list into a named Python variable immediately after the rename checkpoint and use only that validated list in subsequent dedup/final projection logic. Never hand-type additional inferred column names later in the transform.
- In Bronze, build and validate one source object at a time in the exact order defined in `## Bronze`. Do not start the next Bronze object until the current one has passed source existence validation, source-column validation, pre-write checkpoint, write success, and post-write readback.
- If any Bronze table or view fails validation or write, stop the notebook immediately after `_save_error('bronze', e)` and `raise`; do not continue attempting later Bronze objects.
- In Bronze, after each source read and after each non-trivial transformation step, immediately force a lightweight Spark action before moving on. Use only `limit(1).collect()`, `take(1)`, or `count()` when row-count validation is explicitly required.
- Bronze notebook control flow must be strictly linear and blocking: read one source object, validate schema, add only the allowed Bronze metadata/helper columns for that object, checkpoint it, write it, read it back once to verify schema/readability, then and only then proceed to the next source object.
- In Bronze, do not infer or require business keys for `v_*` source objects during landing. Land the source view columns exactly as provided by the source plus Bronze metadata columns only.

Required notebook cell commenting standard:
- EACH code cell must start with a short markdown-style Python comment block.
- Use a `# ---` divider followed by 1-3 human-readable `# ` comment lines describing what the cell is doing and why.
- Never emit a code cell without this leading comment block.
- Example:
  - `# ---`
  - `# Read raw bronze table and apply schema enforcement.`
  - `# Drops rows where any required key is null.`

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

## Bronze
Land each selected source object into the `bronze` schema as 1:1 raw Delta tables with minimal transformation and consistent metadata.

### Bronze write pattern
- Load type: full snapshot ingestion for all listed tables/views unless the user later edits to introduce incremental logic based on `ModifiedDate`.
- Write mode: idempotent overwrite per table for each run.
- Table naming: `bronze.address`, `bronze.customer`, `bronze.customer_address`, `bronze.product`, `bronze.product_category`, `bronze.product_description`, `bronze.product_model`, `bronze.product_model_product_description`, `bronze.sales_order_detail`, `bronze.sales_order_header`, `bronze.v_get_all_categories`, `bronze.v_product_and_description`, `bronze.v_product_model_catalog_description`.
- Bronze build order must be exactly:
  1. `bronze.address`
  2. `bronze.customer`
  3. `bronze.customer_address`
  4. `bronze.product`
  5. `bronze.product_category`
  6. `bronze.product_description`
  7. `bronze.product_model`
  8. `bronze.product_model_product_description`
  9. `bronze.sales_order_detail`
  10. `bronze.sales_order_header`
  11. `bronze.v_get_all_categories`
  12. `bronze.v_product_and_description`
  13. `bronze.v_product_model_catalog_description`
- Metadata columns to append to every bronze table:
  - `ingested_at_utc`
  - `run_id`
  - `source_workspace_id`
  - `source_lakehouse_id`
  - `source_lakehouse_name`
  - `source_object_name`
  - `source_record_hash` from all business columns
- Partitioning:
  - `bronze.sales_order_header`: partition by persisted helper column `order_year` derived from validated source column `OrderDate`
  - `bronze.sales_order_detail`: no partition initially unless volume proves large; detail lacks a natural date without joining header
  - All other tables/views: no partitioning due to likely small/master/reference size
- Schema drift handling:
  - Enforce expected schema from provided column lists for base tables
  - For `v_*` source views, do not enforce guessed business-key columns; only validate that the source object exists and has a non-empty schema before landing
  - Log and fail on missing required source columns
  - Optionally allow additive columns only if user edits the spec to support schema evolution

### Bronze execution contract
- For each Bronze object, execution must be blocking and fail-fast with these checkpoints in order:
  1. validate source object exists in the source lakehouse
  2. read the source object into a named `*_source_df`
  3. immediately validate the source schema:
     - for base tables, assert the required source columns listed below exist before any further transform
     - for `v_*` views, capture the exact source column list into a named variable and assert it is non-empty
  4. immediately run a lightweight checkpoint action on `*_source_df`
  5. add only the allowed Bronze metadata columns and any explicitly allowed helper column for that object into a named `*_bronze_df`
  6. validate the final Bronze schema before write
  7. immediately run a lightweight checkpoint action on `*_bronze_df`
  8. write the Bronze table
  9. read the Bronze table back and verify schema/readability with a lightweight action
- For every Bronze object, create a dedicated `*_bronze_df` variable that contains exactly the persisted source business columns in their original source names/order, plus the Bronze metadata columns, plus only explicitly allowed helper columns for that object.
- No Bronze transform may rename source business columns, deduplicate rows, or infer/introduce analytical attributes beyond the explicit helper column `order_year` on `bronze.sales_order_header`.
- If any Bronze object fails any checkpoint above, stop immediately and do not continue to later Bronze objects.

### Table-specific landing notes
- `SalesLT/Address`
  - Required source columns: `AddressID`, `AddressLine1`, `AddressLine2`, `City`, `StateProvince`, `CountryRegion`, `PostalCode`, `rowguid`, `ModifiedDate`
  - Raw primary key candidate: `AddressID`
  - Preserve address text exactly; no trimming/standardization in Bronze

- `SalesLT/Customer`
  - Required source columns: `CustomerID`, `NameStyle`, `Title`, `FirstName`, `MiddleName`, `LastName`, `Suffix`, `CompanyName`, `SalesPerson`, `EmailAddress`, `Phone`, `PasswordHash`, `PasswordSalt`, `rowguid`, `ModifiedDate`
  - Raw primary key candidate: `CustomerID`
  - Keep sensitive columns `PasswordHash` and `PasswordSalt` in Bronze only

- `SalesLT/CustomerAddress`
  - Required source columns: `CustomerID`, `AddressID`, `AddressType`, `rowguid`, `ModifiedDate`
  - Junction table between customer and address
  - Composite key candidate: `CustomerID`, `AddressID`, `AddressType`

- `SalesLT/Product`
  - Required source columns: `ProductID`, `Name`, `ProductNumber`, `Color`, `StandardCost`, `ListPrice`, `Size`, `Weight`, `ProductCategoryID`, `ProductModelID`, `SellStartDate`, `SellEndDate`, `DiscontinuedDate`, `ThumbNailPhoto`, `ThumbNailPhotoFileName`, `rowguid`, `ModifiedDate`
  - Raw primary key candidate: `ProductID`
  - Preserve binary `ThumbNailPhoto` in Bronze; likely exclude from Silver/Gold analytics

- `SalesLT/ProductCategory`
  - Required source columns: `ProductCategoryID`, `ParentProductCategoryID`, `Name`, `rowguid`, `ModifiedDate`
  - Raw primary key candidate: `ProductCategoryID`
  - Contains parent-child hierarchy via `ParentProductCategoryID`

- `SalesLT/ProductDescription`
  - Required source columns: `ProductDescriptionID`, `Description`, `rowguid`, `ModifiedDate`
  - Raw primary key candidate: `ProductDescriptionID`

- `SalesLT/ProductModel`
  - Required source columns: `ProductModelID`, `Name`, `CatalogDescription`, `rowguid`, `ModifiedDate`
  - Raw primary key candidate: `ProductModelID`

- `SalesLT/ProductModelProductDescription`
  - Required source columns: `ProductModelID`, `ProductDescriptionID`, `Culture`, `rowguid`, `ModifiedDate`
  - Bridge table among product model, description, and culture
  - Composite key candidate: `ProductModelID`, `ProductDescriptionID`, `Culture`

- `SalesLT/SalesOrderDetail`
  - Required source columns: `SalesOrderID`, `SalesOrderDetailID`, `OrderQty`, `ProductID`, `UnitPrice`, `UnitPriceDiscount`, `LineTotal`, `rowguid`, `ModifiedDate`
  - Transaction line fact candidate
  - Composite key candidate: `SalesOrderID`, `SalesOrderDetailID`

- `SalesLT/SalesOrderHeader`
  - Required source columns: `SalesOrderID`, `SalesOrderNumber`, `PurchaseOrderNumber`, `AccountNumber`, `CustomerID`, `BillToAddressID`, `ShipToAddressID`, `ShipMethod`, `OrderDate`, `DueDate`, `ShipDate`, `Status`, `OnlineOrderFlag`, `SubTotal`, `TaxAmt`, `Freight`, `TotalDue`, `SalesPerson`, `rowguid`, `ModifiedDate`
  - Order header fact candidate
  - Raw primary key candidate: `SalesOrderID`
  - Mandatory helper-column rule:
    - derive `order_year` only from validated source column `OrderDate`
    - add `order_year` in the named `sales_order_header_bronze_df` before write
    - do not attempt to derive `order_year` if `OrderDate` is missing; fail immediately instead
  - Final persisted columns for this Bronze table must be exactly:
    - all original source columns in original source order
    - `order_year`
    - all Bronze metadata columns

- `SalesLT/vGetAllCategories`
  - Required process only:
    - validate the source view exists
    - inspect and capture the exact source column list
    - assert the source column list is non-empty
    - land exactly those source columns plus Bronze metadata columns
  - Reference view already flattening category hierarchy
  - Do not require or infer `ProductCategoryID` during Bronze landing

- `SalesLT/vProductAndDescription`
  - Required process only:
    - validate the source view exists
    - inspect and capture the exact source column list
    - assert the source column list is non-empty
    - land exactly those source columns plus Bronze metadata columns
  - Enriched product text view by source-provided columns only
  - Do not require or infer `ProductID` or `Culture` during Bronze landing

- `SalesLT/vProductModelCatalogDescription`
  - Required process only:
    - validate the source view exists
    - inspect and capture the exact source column list
    - assert the source column list is non-empty
    - land exactly those source columns plus Bronze metadata columns
  - Wide descriptive view by source-provided columns only
  - Do not require or infer `ProductModelID` during Bronze landing

## Silver
Standardize names, types, deduplicate, remove clearly non-analytic sensitive/binary fields where appropriate, and produce conformed clean tables in the `silver` schema.

### Silver common rules
- Rename all columns to `snake_case`.
- Add audit columns:
  - `silver_loaded_at_utc`
  - `run_id`
  - `record_hash`
- Standardize string blanks to null where appropriate for descriptive columns.
- Preserve source keys as-is; do not generate surrogate keys in Silver.
- Deduplicate using the stated business keys, keeping the latest by `modified_date` where available.
- Apply `OPTIMIZE` after writes for large tables:
  - `silver.sales_order_header`
  - `silver.sales_order_detail`
- Exclude highly sensitive/non-analytic fields from downstream conformed analytics:
  - Drop `password_hash`, `password_salt` from `silver.customer`
  - Drop `thumbnail_photo` from `silver.product` unless the user wants image reporting scenarios
- Silver build order must be exactly:
  1. `silver.address`
  2. `silver.customer`
  3. `silver.customer_address`
  4. `silver.product`
  5. `silver.product_category`
  6. `silver.product_description`
  7. `silver.product_model`
  8. `silver.product_model_product_description`
  9. `silver.sales_order_header`
  10. `silver.sales_order_detail`
  11. `silver.v_get_all_categories`
  12. `silver.v_product_and_description`
  13. `silver.v_product_model_catalog_description`
- For each Silver table, execution must be blocking and fail-fast with these checkpoints in order:
  1. validate Bronze source table exists
  2. validate required Bronze source columns before rename
  3. perform `snake_case` renaming into a named `*_renamed_df`
  4. validate required post-rename column names before any further transformation
  5. immediately run a lightweight checkpoint action on `*_renamed_df` (`limit(1).collect()` or `take(1)`)
  6. apply derived columns in dependency order into a named `*_derived_df` when derivations are required
  7. validate the derived-step schema and immediately run a lightweight checkpoint action on `*_derived_df`
  8. apply dedup/ranking using only columns that exist on that DataFrame at that step into a named `*_dedup_df`
  9. validate the dedup/ranking-step schema and immediately run a lightweight checkpoint action on `*_dedup_df`
  10. project exactly the intended persisted columns plus required audit columns into a named `*_final_df`
  11. validate final schema exactly
  12. validate required key columns are non-null
  13. validate uniqueness at declared grain
  14. immediately run a lightweight checkpoint action on `*_final_df`
  15. write the table
  16. read the table back and verify schema/readability with a lightweight action
- If any Silver table fails any checkpoint above, stop immediately and do not continue to later Silver tables.
- For every Silver table, create a dedicated `*_final_df` variable that contains exactly the persisted columns for that table before write.
- No Silver transform may reference original Bronze/PascalCase names after the rename-to-`snake_case` step. After rename, use only `snake_case` names.
- Do not assume optional columns exist in Silver source data. If a transform depends on a non-required column, first validate it exists; otherwise either emit the output as null only where the spec explicitly allows, or fail clearly.
- Do not combine multiple Silver tables in a shared transform/write loop that delays Spark actions. Each Silver table must complete all schema assertions, checkpoint actions, write, and readback before the next table starts.
- For Silver tables with no derived columns, still create the named intermediate DataFrames `*_renamed_df`, `*_dedup_df`, and `*_final_df` and run the required schema assertions and lightweight checkpoint actions at each stage.
- For each `v_*` Silver table, the notebook must first inspect the actual Bronze/view schema, then rename to `snake_case`, then capture `actual_view_columns = sorted(df.columns)` from the renamed DataFrame, and only then define dedup logic and final projection from that validated list.
- For each `v_*` Silver table, the final persisted business columns must be exactly the validated renamed source columns in their existing order, plus audit columns; do not invent, drop, or reorder business columns unless this spec explicitly says so.
- For each `v_*` Silver table, if the expected dedup key column(s) are absent after rename, fail immediately with an error that includes the table name and the full renamed column list. Do not substitute a guessed alternative key.

### Silver table specifications
- `silver.address`
  - Source: `bronze.address`
  - Required Bronze columns before rename: `AddressID`, `AddressLine1`, `AddressLine2`, `City`, `StateProvince`, `CountryRegion`, `PostalCode`, `rowguid`, `ModifiedDate`
  - Required columns after rename: `address_id`, `address_line1`, `address_line2`, `city`, `state_province`, `country_region`, `postal_code`, `rowguid`, `modified_date`
  - Key: `address_id`
  - Dedup key: `address_id`
  - Intermediate DataFrame contract:
    - `address_renamed_df` must contain the required post-rename columns
    - `address_dedup_df` must contain the same business columns before final audit-column projection
    - `address_final_df` must contain exactly the final persisted columns plus audit columns
  - Final business columns must be exactly:
    - `address_id`
    - `address_line1`
    - `address_line2`
    - `city`
    - `state_province`
    - `country_region`
    - `postal_code`
    - `rowguid`
    - `modified_date`
  - Persist final business columns plus audit columns only

- `silver.customer`
  - Source: `bronze.customer`
  - Required Bronze columns before rename: `CustomerID`, `NameStyle`, `Title`, `FirstName`, `MiddleName`, `LastName`, `Suffix`, `CompanyName`, `SalesPerson`, `EmailAddress`, `Phone`, `rowguid`, `ModifiedDate`
  - Required columns after rename: `customer_id`, `name_style`, `title`, `first_name`, `middle_name`, `last_name`, `suffix`, `company_name`, `sales_person`, `email_address`, `phone`, `rowguid`, `modified_date`
  - Key: `customer_id`
  - Dedup key: `customer_id`
  - Intermediate DataFrame contract:
    - `customer_renamed_df` must contain all required post-rename columns
    - `customer_derived_df` must add only `full_name` and `customer_name_preferred`
    - `customer_dedup_df` must preserve `full_name` and `customer_name_preferred`
    - `customer_final_df` must contain exactly the final persisted columns plus audit columns
  - Derived-column order is mandatory:
    1. derive `full_name` from `title`, `first_name`, `middle_name`, `last_name`, `suffix`
    2. derive `customer_name_preferred` using `company_name` when present, else `full_name`
  - Final business columns must be exactly:
    - `customer_id`
    - `name_style`
    - `title`
    - `first_name`
    - `middle_name`
    - `last_name`
    - `suffix`
    - `company_name`
    - `sales_person`
    - `email_address`
    - `phone`
    - `rowguid`
    - `modified_date`
    - `full_name`
    - `customer_name_preferred`
  - Persist final business columns plus audit columns only

- `silver.customer_address`
  - Source: `bronze.customer_address`
  - Required Bronze columns before rename: `CustomerID`, `AddressID`, `AddressType`, `rowguid`, `ModifiedDate`
  - Required columns after rename: `customer_id`, `address_id`, `address_type`, `rowguid`, `modified_date`
  - Dedup key: `customer_id`, `address_id`, `address_type`
  - Intermediate DataFrame contract:
    - `customer_address_renamed_df` must contain the required post-rename columns
    - `customer_address_dedup_df` must preserve exactly the business columns needed for final output
    - `customer_address_final_df` must contain exactly the final persisted columns plus audit columns
  - Final business columns must be exactly:
    - `customer_id`
    - `address_id`
    - `address_type`
    - `rowguid`
    - `modified_date`
  - Retain junction structure for later billing/shipping role-play logic
  - Persist final business columns plus audit columns only

- `silver.product`
  - Source: `bronze.product`
  - Required Bronze columns before rename: `ProductID`, `Name`, `ProductNumber`, `Color`, `StandardCost`, `ListPrice`, `Size`, `Weight`, `ProductCategoryID`, `ProductModelID`, `SellStartDate`, `SellEndDate`, `DiscontinuedDate`, `ThumbNailPhotoFileName`, `rowguid`, `ModifiedDate`
  - Required columns after rename: `product_id`, `name`, `product_number`, `color`, `standard_cost`, `list_price`, `size`, `weight`, `product_category_id`, `product_model_id`, `sell_start_date`, `sell_end_date`, `discontinued_date`, `thumbnail_photo_file_name`, `rowguid`, `modified_date`
  - Key: `product_id`
  - Dedup key: `product_id`
  - Intermediate DataFrame contract:
    - `product_renamed_df` must contain all required post-rename columns
    - `product_derived_df` must add only `is_discontinued` and `is_sellable_currently`
    - `product_dedup_df` must preserve both derived flags and all final business columns
    - `product_final_df` must contain exactly the final persisted columns plus audit columns
  - Derived-column order is mandatory:
    1. derive `is_discontinued` from `discontinued_date.isNotNull()`
    2. derive `is_sellable_currently` using existing columns on the DataFrame at that step only, with this exact intent:
       - true when `sell_start_date` is not null
       - and (`sell_end_date` is null or `sell_end_date` is in the future/current date logic as implemented)
       - and `is_discontinued = false`
    3. only after both flags are derived may you project final columns
  - Hard prohibition:
    - do NOT drop `discontinued_date`, `sell_start_date`, or `sell_end_date` before `is_sellable_currently` has been computed
    - do NOT compute `is_sellable_currently` by referencing `discontinued_date` after that column has been projected away; reuse `is_discontinued`
  - Final business columns must be exactly:
    - `product_id`
    - `name`
    - `product_number`
    - `color`
    - `standard_cost`
    - `list_price`
    - `size`
    - `weight`
    - `product_category_id`
    - `product_model_id`
    - `sell_start_date`
    - `sell_end_date`
    - `discontinued_date`
    - `thumbnail_photo_file_name`
    - `rowguid`
    - `modified_date`
    - `is_discontinued`
    - `is_sellable_currently`
  - Persist final business columns plus audit columns only

- `silver.product_category`
  - Source: `bronze.product_category`
  - Required Bronze columns before rename: `ProductCategoryID`, `ParentProductCategoryID`, `Name`, `rowguid`, `ModifiedDate`
  - Required columns after rename: `product_category_id`, `parent_product_category_id`, `name`, `rowguid`, `modified_date`
  - Key: `product_category_id`
  - Dedup key: `product_category_id`
  - Intermediate DataFrame contract:
    - `product_category_renamed_df` must contain the required post-rename columns
    - `product_category_dedup_df` must contain the same business columns before final audit-column projection
    - `product_category_final_df` must contain exactly the final persisted columns plus audit columns
  - Final business columns must be exactly:
    - `product_category_id`
    - `parent_product_category_id`
    - `name`
    - `rowguid`
    - `modified_date`
  - Persist final business columns plus audit columns only

- `silver.product_description`
  - Source: `bronze.product_description`
  - Required Bronze columns before rename: `ProductDescriptionID`, `Description`, `rowguid`, `ModifiedDate`
  - Required columns after rename: `product_description_id`, `description`, `rowguid`, `modified_date`
  - Key: `product_description_id`
  - Dedup key: `product_description_id`
  - Intermediate DataFrame contract:
    - `product_description_renamed_df` must contain the required post-rename columns
    - `product_description_dedup_df` must contain the same business columns before final audit-column projection
    - `product_description_final_df` must contain exactly the final persisted columns plus audit columns
  - Final business columns must be exactly:
    - `product_description_id`
    - `description`
    - `rowguid`
    - `modified_date`
  - Persist final business columns plus audit columns only

- `silver.product_model`
  - Source: `bronze.product_model`
  - Required Bronze columns before rename: `ProductModelID`, `Name`, `rowguid`, `ModifiedDate`
  - Required columns after rename: `product_model_id`, `name`, `rowguid`, `modified_date`
  - Key: `product_model_id`
  - Dedup key: `product_model_id`
  - Intermediate DataFrame contract:
    - `product_model_renamed_df` must contain the required post-rename columns
    - `product_model_dedup_df` must contain the same business columns before final audit-column projection
    - `product_model_final_df` must contain exactly the final persisted columns plus audit columns
  - Final business columns must be exactly:
    - `product_model_id`
    - `name`
    - `rowguid`
    - `modified_date`
  - Persist final business columns plus audit columns only

- `silver.product_model_product_description`
  - Source: `bronze.product_model_product_description`
  - Required Bronze columns before rename: `ProductModelID`, `ProductDescriptionID`, `Culture`, `rowguid`, `ModifiedDate`
  - Required columns after rename: `product_model_id`, `product_description_id`, `culture`, `rowguid`, `modified_date`
  - Dedup key: `product_model_id`, `product_description_id`, `culture`
  - Intermediate DataFrame contract:
    - `product_model_product_description_renamed_df` must contain the required post-rename columns
    - `product_model_product_description_dedup_df` must preserve exactly the business columns needed for final output
    - `product_model_product_description_final_df` must contain exactly the final persisted columns plus audit columns
  - Final business columns must be exactly:
    - `product_model_id`
    - `product_description_id`
    - `culture`
    - `rowguid`
    - `modified_date`
  - Persist final business columns plus audit columns only

- `silver.sales_order_header`
  - Source: `bronze.sales_order_header`
  - Required Bronze columns before rename: `SalesOrderID`, `SalesOrderNumber`, `PurchaseOrderNumber`, `AccountNumber`, `CustomerID`, `BillToAddressID`, `ShipToAddressID`, `ShipMethod`, `OrderDate`, `DueDate`, `ShipDate`, `Status`, `OnlineOrderFlag`, `SubTotal`, `TaxAmt`, `Freight`, `TotalDue`, `SalesPerson`, `rowguid`, `ModifiedDate`
  - Required columns after rename: `sales_order_id`, `sales_order_number`, `purchase_order_number`, `account_number`, `customer_id`, `bill_to_address_id`, `ship_to_address_id`, `ship_method`, `order_date`, `due_date`, `ship_date`, `status`, `online_order_flag`, `sub_total`, `tax_amt`, `freight`, `total_due`, `sales_person`, `rowguid`, `modified_date`
  - Key: `sales_order_id`
  - Dedup key: `sales_order_id`
  - Intermediate DataFrame contract:
    - `sales_order_header_renamed_df` must contain all required post-rename columns
    - `sales_order_header_derived_df` must add only `order_date_key`, `due_date_key`, `ship_date_key`, `order_year`, `order_month`, `order_status_desc`
    - `sales_order_header_dedup_df` must preserve all six derived columns and all final business columns
    - `sales_order_header_final_df` must contain exactly the final persisted columns plus audit columns
  - Derived-column order is mandatory:
    1. `order_date_key` from `order_date`
    2. `due_date_key` from `due_date`
    3. `ship_date_key` from `ship_date` when not null
    4. `order_year` from `order_date`
    5. `order_month` from `order_date`
    6. `order_status_desc` from `status`
  - Final business columns must be exactly:
    - `sales_order_id`
    - `sales_order_number`
    - `purchase_order_number`
    - `account_number`
    - `customer_id`
    - `bill_to_address_id`
    - `ship_to_address_id`
    - `ship_method`
    - `order_date`
    - `due_date`
    - `ship_date`
    - `status`
    - `online_order_flag`
    - `sub_total`
    - `tax_amt`
    - `freight`
    - `total_due`
    - `sales_person`
    - `rowguid`
    - `modified_date`
    - `order_date_key`
    - `due_date_key`
    - `ship_date_key`
    - `order_year`
    - `order_month`
    - `order_status_desc`
  - Persist final business columns plus audit columns only

- `silver.sales_order_detail`
  - Source: `bronze.sales_order_detail`
  - Required Bronze columns before rename: `SalesOrderID`, `SalesOrderDetailID`, `OrderQty`, `ProductID`, `UnitPrice`, `UnitPriceDiscount`, `LineTotal`, `rowguid`, `ModifiedDate`
  - Required columns after rename: `sales_order_id`, `sales_order_detail_id`, `order_qty`, `product_id`, `unit_price`, `unit_price_discount`, `line_total`, `rowguid`, `modified_date`
  - Key: `sales_order_id`, `sales_order_detail_id`
  - Dedup key: `sales_order_id`, `sales_order_detail_id`
  - Intermediate DataFrame contract:
    - `sales_order_detail_renamed_df` must contain all required post-rename columns
    - `sales_order_detail_derived_df` must add only `gross_line_amount`, `discount_amount`, `net_line_amount`
    - `sales_order_detail_dedup_df` must preserve all three derived columns and all final business columns
    - `sales_order_detail_final_df` must contain exactly the final persisted columns plus audit columns
  - Derived-column order is mandatory:
    1. `gross_line_amount = order_qty * unit_price`
    2. `discount_amount = order_qty * unit_price * unit_price_discount`
    3. `net_line_amount = line_total`
  - Final business columns must be exactly:
    - `sales_order_id`
    - `sales_order_detail_id`
    - `order_qty`
    - `product_id`
    - `unit_price`
    - `unit_price_discount`
    - `line_total`
    - `rowguid`
    - `modified_date`
    - `gross_line_amount`
    - `discount_amount`
    - `net_line_amount`
  - Persist final business columns plus audit columns only

- `silver.v_get_all_categories`
  - Source: `bronze.v_get_all_categories`
  - Intermediate DataFrame contract:
    - `v_get_all_categories_renamed_df` must be created from the full renamed view schema
    - immediately after rename, capture the exact list of renamed business columns in a variable and assert it is non-empty
    - `v_get_all_categories_dedup_df` must preserve exactly that validated renamed column list
    - `v_get_all_categories_final_df` must persist exactly that validated renamed column list plus audit columns only
  - Required process:
    1. inspect actual Bronze/view columns
    2. rename all columns to `snake_case`
    3. validate/read checkpoint the renamed DataFrame
    4. only then check whether `product_category_id` exists as a renamed column
    5. if `product_category_id` exists, dedup by `product_category_id`
    6. otherwise fail immediately and print the full renamed column list; do not guess another key
  - Persist the validated renamed view columns plus audit columns only
  - Use as optional hierarchy helper in Gold

- `silver.v_product_and_description`
  - Source: `bronze.v_product_and_description`
  - Intermediate DataFrame contract:
    - `v_product_and_description_renamed_df` must be created from the full renamed view schema
    - immediately after rename, capture the exact list of renamed business columns in a variable and assert it is non-empty
    - `v_product_and_description_dedup_df` must preserve exactly that validated renamed column list
    - `v_product_and_description_final_df` must persist exactly that validated renamed column list plus audit columns only
  - Required process:
    1. inspect actual Bronze/view columns
    2. rename all columns to `snake_case`
    3. validate/read checkpoint the renamed DataFrame
    4. only then check whether BOTH `product_id` and `culture` exist as renamed columns
    5. if both exist, dedup by `product_id`, `culture`
    6. otherwise fail immediately and print the full renamed column list; do not guess substitute key columns
  - Persist the validated renamed view columns plus audit columns only
  - Optional multilingual product description helper

- `silver.v_product_model_catalog_description`
  - Source: `bronze.v_product_model_catalog_description`
  - Intermediate DataFrame contract:
    - `v_product_model_catalog_description_renamed_df` must be created from the full renamed view schema
    - immediately after rename, capture the exact list of renamed business columns in a variable and assert it is non-empty
    - `v_product_model_catalog_description_dedup_df` must preserve exactly that validated renamed column list
    - `v_product_model_catalog_description_final_df` must persist exactly that validated renamed column list plus audit columns only
  - Required process:
    1. inspect actual Bronze/view columns
    2. rename all columns to `snake_case`
    3. validate/read checkpoint the renamed DataFrame
    4. only then check whether `product_model_id` exists as a renamed column
    5. if it exists, dedup by `product_model_id`
    6. otherwise fail immediately and print the full renamed column list; do not guess another key
  - Persist the validated renamed view columns plus audit columns only
  - Optional wide model attribute helper

### Silver modeling note
Two valid enrichment routes exist for product/category text:
- Route A: build Gold mainly from base tables plus bridges, using source views only as helper enrichments.
- Route B: use `vGetAllCategories` and `vProductAndDescription` as preferred descriptive sources because they already flatten logic.
- Proposed default: Route A for lineage clarity; user may edit to Route B for simpler Gold logic.

## Gold
Model a sales analytics star schema in `gold`, using actual transaction, master, and bridge tables from the provided schema.
I want to end up with the following tables:
Dim_Customer -> join customer, customer address, address and leave all relevant/meaningfull fields
Dim_Product -> join product, productcategory, producdescription, productmodel, productmodelproductdescription. Please notice that in productcategory there is a self join. So in the end I have the category (which is the parent category) and a subcategory.
Dim_Sales --> Use the salesperson field from salesorderheader, but remove the adventureworks/ in front of the name and remove the last number at the end off the name
Fact_Sales --> join salesorderdetail and salesorderheader and keep all relevant fields
Use below for more insights, but what I just described is your end goal in gold

### Gold implementation rules for this run
Build only these four Gold tables for the final curated layer and semantic model:
- `gold.dim_customer`
- `gold.dim_product`
- `gold.dim_sales`
- `gold.fact_sales`

Do not materialize alternative Gold tables such as `gold.dim_address`, `gold.dim_date`, `gold.dim_ship_method`, `gold.bridge_customer_address`, `gold.fact_sales_order_line