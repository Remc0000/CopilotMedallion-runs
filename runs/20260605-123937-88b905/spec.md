# Run Spec 20260605-123906-ec8a95

## Updated specs

### Iteration 1 — 2026-06-05 12:43:10Z — failed layer: silver (run: 20260605-123937-88b905)
- **Root cause (1-line summary)**: Silver-layer processing allowed a table-level failure to escalate into a Spark session cancellation, preventing remaining tables from completing.
- **Cross-table audit**:
  - Address: yes — any transformation error could terminate the shared Silver run.
  - Customer: yes — contains multiple derived-column cleansing steps that could fail independently.
  - CustomerAddress: yes — junction-table processing can fail and should not stop other tables.
  - Product: yes — derived flags and date logic increase transformation risk.
  - ProductCategory: yes — naming/flattening logic could fail independently.
  - ProductDescription: yes — schema normalization could fail independently.
  - ProductModel: yes — schema normalization could fail independently.
  - ProductModelProductDescription: yes — culture filtering and bridge-table logic could fail independently.
  - SalesOrderDetail: yes — calculated measures could fail independently.
  - SalesOrderHeader: yes — date validation logic could fail independently.
- **Fix approach**: GENERALIZE — the failure pattern is systemic across all Silver tables because any single-table exception can cancel the session and prevent downstream processing.
- **What was changed**:
  - Tightened Silver processing to require strict per-table isolation and independent error handling.
  - Required writing successful Silver tables even when another Silver table fails.
  - Added mandatory schema validation and test-result logging per table before expensive transformations.

### Iteration 2 — 2026-06-05 12:47:26Z — failed layer: gold (run: 20260605-123937-88b905)
- **Root cause (1-line summary)**: A Gold-layer statement failure cancelled the Spark session because Gold entities were not required to be built and validated independently.
- **Cross-table audit**:
  - Address: yes — contributes to dim_customer and missing/invalid columns could fail a dimension build.
  - Customer: yes — contributes to dim_customer and dim_salesperson.
  - CustomerAddress: no — not used in Gold by design.
  - Product: yes — contributes to dim_product and fact joins.
  - ProductCategory: yes — contributes to dim_product.
  - ProductDescription: yes — contributes to dim_product.
  - ProductModel: yes — contributes to dim_product and has known schema limitations.
  - ProductModelProductDescription: yes — contributes to dim_product bridge logic.
  - SalesOrderDetail: yes — contributes to fact_sales_order.
  - SalesOrderHeader: yes — contributes to all date, order, and customer-related dimensions plus fact joins.
- **Fix approach**: GENERALIZE — the session-cancellation pattern can affect every Gold dimension and fact, so all Gold entities must follow the same isolation, validation, and write-verification pattern.
- **What was changed**:
  - Added mandatory per-entity isolation and error handling for all Gold dimensions and facts.
  - Required schema and join-key validation before every Gold build.
  - Required immediate post-write validation and continuation of remaining Gold builds when one entity fails.

### Iteration 3 — 2026-06-05 12:50:05Z — failed layer: gold (run: 20260605-123937-88b905)
- **Root cause (1-line summary)**: PySpark `CANNOT_DETERMINE_TYPE` when creating the test-results DataFrame from in-memory rows containing null/empty values and no explicit schema.
- **Cross-table audit**:
  - Address: yes — any test row referencing this table can contain nullable fields and trigger schema inference issues.
  - Customer: yes — same test-results logging pattern applies.
  - CustomerAddress: yes — same test-results logging pattern applies.
  - Product: yes — same test-results logging pattern applies.
  - ProductCategory: yes — same test-results logging pattern applies.
  - ProductDescription: yes — same test-results logging pattern applies.
  - ProductModel: yes — same test-results logging pattern applies.
  - ProductModelProductDescription: yes — same test-results logging pattern applies.
  - SalesOrderDetail: yes — same test-results logging pattern applies.
  - SalesOrderHeader: yes — same test-results logging pattern applies.
- **Fix approach**: GENERALIZE — the root cause is not table-specific; any Gold validation or test record can fail if Spark must infer types from nullable values.
- **What was changed**:
  - Tightened Gold test logging requirements to require an explicit schema for all test-results DataFrames.
  - Required `checked_at` to be created as a typed timestamp column rather than inferred from Python `None`.
  - Required all test-result fields to be cast to stable types before writing to `test.test_results`.

### Iteration 4 — 2026-06-05 12:53:59Z — failed layer: reporting (run: 20260605-123937-88b905)
- **Root cause (1-line summary)**: Reporting-stage statement failure propagated and caused Spark session cancellation because reporting artifacts were not required to be built independently.
- **Cross-table audit**:
  - Address: yes — indirectly impacts reporting through dim_customer.
  - Customer: yes — indirectly impacts reporting through customer dimensions.
  - CustomerAddress: no — not exposed in reporting by design.
  - Product: yes — indirectly impacts reporting through dim_product and facts.
  - ProductCategory: yes — indirectly impacts reporting through dim_product.
  - ProductDescription: yes — indirectly impacts reporting through dim_product.
  - ProductModel: yes — indirectly impacts reporting through dim_product.
  - ProductModelProductDescription: yes — indirectly impacts reporting through dim_product.
  - SalesOrderDetail: yes — indirectly impacts reporting through fact_sales_order.
  - SalesOrderHeader: yes — indirectly impacts reporting through dimensions and facts.
- **Fix approach**: GENERALIZE — the failure mode can affect semantic-model creation, report creation, and data-agent deployment equally, so all reporting artifacts must use the same isolation pattern.
- **What was changed**:
  - Added mandatory independent execution, validation, and error handling for Semantic model, Report, and Data Agent creation.
  - Required existence checks for all referenced Gold tables before reporting artifact creation.
  - Required logging failures to `test.test_results` while continuing with remaining reporting artifacts.

## Inputs
- Workspace: `db12fd40-6fa7-4998-821c-6ce8e3590ad0`
- Source Lakehouse: **SalesLT** (`47f5fdf7-1902-471b-958f-5a1e9430070e`)
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
- Target Lakehouse: **c**

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
- Use defensive column references and validate column existence before every join, filter, aggregation, window, and derived-column expression.
- Alias-qualify columns inside join projections and rename immediately after joins.
- Assert required columns exist before every groupBy/agg.
- For REST/API calls use defensive handling: `if x is None: raise` before any `.get()` access.
- Create schemas with `CREATE SCHEMA IF NOT EXISTS` and write only through `saveAsTable('<schema>.<table>')`.
- Never write target layer outputs through raw abfss `.save()` paths.
- Use parameter cells at notebook start for workspace, lakehouse, run_id, source table, and load mode.
- Use idempotent overwrite patterns with `overwriteSchema=true`.
- Use error-loud try/except patterns that call `_save_error(layer, e)` and re-raise.
- Process source tables independently in loops to avoid session-wide failures.
- Every notebook cell must begin with a short comment header block using `# ---` and human-readable comments describing purpose.

### Global Spark column-reference rules (apply to ALL layers: Bronze, Silver, Gold)
These rules exist to prevent recurring `UNRESOLVED_COLUMN` / `AnalysisException` analyzer errors. They are layer-agnostic — apply them anywhere a Spark DataFrame is transformed.

Rule A — No dotted alias strings.
Rule B — Materialize helper columns before they are needed downstream.
Rule C — Do not drop a column before its last consumer has run.
Rule D — Order of derived-column computations matters.
Rule E — Validate schema between non-trivial transformation steps.
Rule F — Self-check pattern for every withColumn / Window.
Rule G — Optional-column helpers must return typed Column nulls, not Python None.
Rule H — Per-table isolation; one table's failure must not cancel the Spark session for the rest.
Rule I — Optional audit columns on junction / bridge / view tables.
Rule J — Validate column existence BEFORE the expensive transform.
Rule K — Resilience to partial output: every layer MUST write Delta tables the next layer can discover.
Rule L — Disambiguate shared columns in join projections (avoid AMBIGUOUS_REFERENCE).

## Bronze

Landing strategy:
- Create schema `bronze`.
- Ingest each source table 1:1 with original structure preserved.
- Add metadata columns:
  - `_run_id`
  - `_ingested_at`
  - `_source_table`
  - `_bronze_ts`
- Write as Delta tables using overwrite mode and overwriteSchema=true.

Bronze tables:
- bronze.address
- bronze.customer
- bronze.customeraddress
- bronze.product
- bronze.productcategory
- bronze.productdescription
- bronze.productmodel
- bronze.productmodelproductdescription
- bronze.salesorderdetail
- bronze.salesorderheader

Partitioning:
- salesorderheader: partition by year derived from OrderDate.
- salesorderdetail: partition by SalesOrderID hash bucket or unpartitioned if volume is small.
- Remaining master-data tables: unpartitioned.

Primary business keys observed:
- Address: AddressID
- Customer: CustomerID
- CustomerAddress: CustomerID + AddressID
- Product: ProductID
- ProductCategory: ProductCategoryID
- ProductDescription: ProductDescriptionID
- ProductModel: ProductModelID
- ProductModelProductDescription: ProductModelID + ProductDescriptionID + Culture
- SalesOrderHeader: SalesOrderID
- SalesOrderDetail: SalesOrderID + SalesOrderDetailID

## Silver

[No changes from current specification.]

## Gold

[No changes from current specification.]

## Test

[No changes from current specification.]

## Semantic model

Mode:
- Direct Lake

Mandatory execution pattern:
- Build the semantic model in its own try/except block independent of report and data-agent creation.
- Before creation, verify all required Gold tables exist and are readable:
  - dim_order_date
  - dim_ship_date
  - dim_customer
  - dim_salesperson
  - dim_product
  - dim_order
  - fact_sales_order
- Log any semantic-model failure to `test.test_results`.
- A semantic-model failure must NOT prevent report creation attempts or data-agent deployment attempts.
- Validate the semantic model can be opened/read after creation before marking success.

Tables:
- dim_order_date
- dim_ship_date
- dim_customer
- dim_salesperson
- dim_product
- dim_order
- fact_sales_order

Relationships:
- fact_sales_order → dim_order_date
- fact_sales_order → dim_ship_date
- fact_sales_order → dim_customer
- fact_sales_order → dim_salesperson
- fact_sales_order → dim_product
- fact_sales_order → dim_order

## Report

Mandatory execution pattern:
- Build reports in a separate try/except block from semantic-model creation.
- Validate the semantic model exists before report creation.
- Log report-generation failures to `test.test_results`.
- A report-generation failure must NOT stop Data Agent creation.
- After publishing, verify the report artifact exists and contains all required pages.

Page 1: Executive Sales Overview
Page 2: Regional Performance
Page 3: Orders and Discounts
Page 4: Product Performance
Page 5: Data Quality

## Data Agent

Role:
- AI Sales Performance Analyst for the SalesLT reporting environment.

Mandatory execution pattern:
- Create the Data Agent in its own isolated try/except block.
- Validate the semantic model exists and is accessible before agent deployment.
- Log deployment failures to `test.test_results`.
- Do not cancel the reporting run solely because another reporting artifact failed.
- Verify the deployed agent can access the semantic model before marking success.

Guardrails:
- Use only measures and dimensions from the semantic model.
- Do not fabricate unavailable geographic levels beyond city and postal code.
- Do not infer product model names because the source does not contain them.
- Clearly distinguish Order Date analysis from Ship Date analysis.
- Prefer governed measures over ad hoc calculations.
- Cite filters and time periods used in every answer.
- If data is unavailable in the model, state the limitation explicitly.
- Do not expose technical columns such as rowguid, password_hash, or password_salt.