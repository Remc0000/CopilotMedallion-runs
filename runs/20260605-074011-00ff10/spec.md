# Run Spec 20260605-073923-154f5e

## Updated specs

### Iteration 1 — 2026-06-05 07:42:33Z — failed layer: bronze (run: 20260605-074011-00ff10)
- **Root cause (1-line summary)**: Bronze completed without producing discoverable schema-backed Delta tables, causing Silver to fail with "prior layer produced no discoverable tables".
- **Cross-table audit**:
  - Address: yes — subject to the same write/discovery mechanism as all Bronze tables.
  - Customer: yes — subject to the same write/discovery mechanism as all Bronze tables.
  - CustomerAddress: yes — subject to the same write/discovery mechanism as all Bronze tables.
  - Product: yes — subject to the same write/discovery mechanism as all Bronze tables.
  - ProductCategory: yes — subject to the same write/discovery mechanism as all Bronze tables.
  - ProductDescription: yes — subject to the same write/discovery mechanism as all Bronze tables.
  - ProductModel: yes — subject to the same write/discovery mechanism as all Bronze tables.
  - ProductModelProductDescription: yes — subject to the same write/discovery mechanism as all Bronze tables.
  - SalesOrderDetail: yes — subject to the same write/discovery mechanism as all Bronze tables.
  - SalesOrderHeader: yes — subject to the same write/discovery mechanism as all Bronze tables.
- **Fix approach**: GENERALIZE — the failure is systemic and affects discoverability for every Bronze table, not a single table-specific schema issue.
- **What was changed**:
  - Tightened Bronze write requirements to mandate one successful `saveAsTable('bronze.<table>')` per source table.
  - Added post-write validation that each expected Bronze table is discoverable via Spark catalog metadata before notebook completion.
  - Added a hard-fail condition if any expected Bronze table is missing or if fewer than 10 Bronze tables are discoverable.

### Iteration 2 — 2026-06-05 07:46:18Z — failed layer: bronze (run: 20260605-074011-00ff10)
- **Root cause (1-line summary)**: Bronze still did not produce discoverable tables; table registration/naming did not match the required `bronze.<table>` catalog objects inspected by the pipeline.
- **Cross-table audit**:
  - Address: yes — must register as `bronze.address`.
  - Customer: yes — must register as `bronze.customer`.
  - CustomerAddress: yes — must register as `bronze.customeraddress`.
  - Product: yes — must register as `bronze.product`.
  - ProductCategory: yes — must register as `bronze.productcategory`.
  - ProductDescription: yes — must register as `bronze.productdescription`.
  - ProductModel: yes — must register as `bronze.productmodel`.
  - ProductModelProductDescription: yes — must register as `bronze.productmodelproductdescription`.
  - SalesOrderDetail: yes — must register as `bronze.salesorderdetail`.
  - SalesOrderHeader: yes — must register as `bronze.salesorderheader`.
- **Fix approach**: GENERALIZE — every source table is subject to the same registration/discovery requirement.
- **What was changed**:
  - Added a mandatory source-to-target mapping for all 10 Bronze tables.
  - Required catalog verification using both `spark.catalog.tableExists()` and `SHOW TABLES IN bronze`.
  - Added a final hard-fail check requiring exactly the 10 expected Bronze table names before notebook success.

### Iteration 1 — 2026-06-05 07:57:06Z — failed layer: bronze (run: 20260605-074011-00ff10)
- **Root cause (1-line summary)**: Silver could not find any physical Bronze tables under `Tables/bronze/`, indicating Bronze outputs were not materialized in the target lakehouse schema location expected by downstream discovery.
- **Cross-table audit**:
  - Address: yes — must physically exist as a managed table in `Tables/bronze/address`.
  - Customer: yes — same managed-table location requirement.
  - CustomerAddress: yes — same managed-table location requirement.
  - Product: yes — same managed-table location requirement.
  - ProductCategory: yes — same managed-table location requirement.
  - ProductDescription: yes — same managed-table location requirement.
  - ProductModel: yes — same managed-table location requirement.
  - ProductModelProductDescription: yes — same managed-table location requirement.
  - SalesOrderDetail: yes — same managed-table location requirement.
  - SalesOrderHeader: yes — same managed-table location requirement.
- **Fix approach**: GENERALIZE — the failure is a lakehouse table materialization/discovery issue affecting every Bronze table uniformly.
- **What was changed**:
  - Tightened Bronze requirements to require managed lakehouse tables backed by the target lakehouse Tables area, not temporary views or external-only registrations.
  - Added mandatory verification of both catalog registration and physical discoverability for all 10 Bronze tables.
  - Added a hard-fail condition if the Bronze schema contains fewer than 10 readable managed Delta tables.

### Iteration 2 — 2026-06-05 08:00:39Z — failed layer: bronze (run: 20260605-074011-00ff10)
- **Root cause (1-line summary)**: Silver explicitly reported no discoverable tables in `Tables/bronze/`; catalog registration alone was insufficient and Bronze outputs were not validated against the lakehouse Tables area used by downstream discovery.
- **Cross-table audit**:
  - Address: yes — downstream discovery requires a readable managed table under the bronze schema.
  - Customer: yes — same requirement.
  - CustomerAddress: yes — same requirement.
  - Product: yes — same requirement.
  - ProductCategory: yes — same requirement.
  - ProductDescription: yes — same requirement.
  - ProductModel: yes — same requirement.
  - ProductModelProductDescription: yes — same requirement.
  - SalesOrderDetail: yes — same requirement.
  - SalesOrderHeader: yes — same requirement.
- **Fix approach**: GENERALIZE — every source table is affected by the same lakehouse attachment/materialization/discovery mechanism.
- **What was changed**:
  - Tightened Bronze requirements to require verification that the active lakehouse is the target lakehouse `o` before any read or write.
  - Added mandatory post-write validation using catalog metadata and table detail metadata for all 10 Bronze tables.
  - Added a hard-fail requirement if any Bronze table is not a managed Delta table in schema `bronze` or if fewer than 10 tables are returned by `SHOW TABLES IN bronze`.

### Iteration 1 — 2026-06-05 08:03:54Z — failed layer: bronze (run: 20260605-074011-00ff10)
- **Root cause (1-line summary)**: Spark session was cancelled due to one or more failed Bronze statements; the spec lacked mandatory statement-level validation and fail-fast checks around schema creation, source reads, and table writes.
- **Cross-table audit**:
  - Address: yes — source read, row-count validation, and write operations can fail.
  - Customer: yes — source read, row-count validation, and write operations can fail.
  - CustomerAddress: yes — source read, row-count validation, and write operations can fail.
  - Product: yes — source read, row-count validation, and write operations can fail.
  - ProductCategory: yes — source read, row-count validation, and write operations can fail.
  - ProductDescription: yes — source read, row-count validation, and write operations can fail.
  - ProductModel: yes — source read, row-count validation, and write operations can fail.
  - ProductModelProductDescription: yes — source read, row-count validation, and write operations can fail.
  - SalesOrderDetail: yes — source read, row-count validation, and write operations can fail.
  - SalesOrderHeader: yes — source read, row-count validation, and write operations can fail.
- **Fix approach**: GENERALIZE — the failure pattern is operational and can affect every Bronze table uniformly.
- **What was changed**:
  - Added mandatory pre-flight validation for lakehouse attachment, schema existence, and source-table accessibility before processing any table.
  - Required per-table read, count, write, and read-back validation with explicit RuntimeError generation on any failure.
  - Added a Bronze execution manifest requiring all 10 tables to be successfully processed and recorded before notebook success.

### Iteration 2 — 2026-06-05 08:04:59Z — failed layer: bronze (run: 20260605-074011-00ff10)
- **Root cause (1-line summary)**: Bronze Spark statements failed before completion, most likely due to invalid source table resolution; the spec did not require exact source object names and schema validation before ingestion.
- **Cross-table audit**:
  - Address: yes — source must resolve exactly as `SalesLT.Address`.
  - Customer: yes — source must resolve exactly as `SalesLT.Customer`.
  - CustomerAddress: yes — source must resolve exactly as `SalesLT.CustomerAddress`.
  - Product: yes — source must resolve exactly as `SalesLT.Product`.
  - ProductCategory: yes — source must resolve exactly as `SalesLT.ProductCategory`.
  - ProductDescription: yes — source must resolve exactly as `SalesLT.ProductDescription`.
  - ProductModel: yes — source must resolve exactly as `SalesLT.ProductModel`.
  - ProductModelProductDescription: yes — source must resolve exactly as `SalesLT.ProductModelProductDescription`.
  - SalesOrderDetail: yes — source must resolve exactly as `SalesLT.SalesOrderDetail`.
  - SalesOrderHeader: yes — source must resolve exactly as `SalesLT.SalesOrderHeader`.
- **Fix approach**: GENERALIZE — every Bronze ingestion depends on the same source-table resolution and schema validation process.
- **What was changed**:
  - Added mandatory exact source-table name mapping and source schema verification before any write.
  - Required successful catalog existence checks and schema introspection for all 10 source tables.
  - Added a hard-fail condition if any expected source table cannot be resolved exactly as specified.

### Iteration 3 — 2026-06-05 08:09:23Z — failed layer: bronze (run: 20260605-074011-00ff10)
- **Root cause (1-line summary)**: Silver still found no discoverable Bronze tables, indicating Bronze tables were not being created in the attached target lakehouse catalog/schema that downstream discovery scans.
- **Cross-table audit**:
  - Address: yes — must exist as `bronze.address` in the target lakehouse catalog.
  - Customer: yes — same catalog/schema discoverability requirement.
  - CustomerAddress: yes — same catalog/schema discoverability requirement.
  - Product: yes — same catalog/schema discoverability requirement.
  - ProductCategory: yes — same catalog/schema discoverability requirement.
  - ProductDescription: yes — same catalog/schema discoverability requirement.
  - ProductModel: yes — same catalog/schema discoverability requirement.
  - ProductModelProductDescription: yes — same catalog/schema discoverability requirement.
  - SalesOrderDetail: yes — same catalog/schema discoverability requirement.
  - SalesOrderHeader: yes — same catalog/schema discoverability requirement.
- **Fix approach**: GENERALIZE — the failure affects all Bronze tables equally because downstream discovery is performed at the schema/catalog level, not by individual table logic.
- **What was changed**:
  - Tightened Bronze requirements to forbid notebook success unless all 10 tables are visible from `SHOW TABLES IN bronze` in the active target lakehouse session.
  - Added mandatory verification of current catalog/database before and after every write.
  - Added a final hard-fail condition if `SHOW TABLES IN bronze` returns anything other than the 10 expected table names.

## Inputs
- Workspace: `373889eb-1531-49df-9b0c-474976350c90`
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
- Target Lakehouse: **o**

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
- Use defensive column references and assert required columns exist before use.
- Execute `CREATE SCHEMA IF NOT EXISTS bronze`, `silver`, `gold`, and `test` before writes.
- Write all outputs with schema-qualified `saveAsTable('<schema>.<table>')`.
- Never write target lakehouse outputs via raw abfss `.save()`.
- Parameterize workspace, lakehouse, schema, run_id, and table names in notebook parameter cells.
- Use idempotent overwrite patterns with Delta and `overwriteSchema=true`.
- Use error-loud try/except blocks that call `_save_error(layer, e)` and re-raise.
- Process tables in isolated loops with independent read-transform-write logic.
- Every notebook code cell must begin with a short explanatory comment block.
- After layer completion, validate discoverability through catalog metadata, not path existence.
- Do not use temporary views, global temp views, or in-memory objects as layer outputs; every layer output must be a persisted managed Delta table in the target lakehouse.
- Before the first write in any layer, explicitly verify that the attached/default lakehouse is the target lakehouse `o`; fail immediately if another lakehouse is active.

## Bronze

- Land each source table 1:1 into the `bronze` schema using the exact mapping below:
  - `SalesLT.Address` → `bronze.address`
  - `SalesLT.Customer` → `bronze.customer`
  - `SalesLT.CustomerAddress` → `bronze.customeraddress`
  - `SalesLT.Product` → `bronze.product`
  - `SalesLT.ProductCategory` → `bronze.productcategory`
  - `SalesLT.ProductDescription` → `bronze.productdescription`
  - `SalesLT.ProductModel` → `bronze.productmodel`
  - `SalesLT.ProductModelProductDescription` → `bronze.productmodelproductdescription`
  - `SalesLT.SalesOrderDetail` → `bronze.salesorderdetail`
  - `SalesLT.SalesOrderHeader` → `bronze.salesorderheader`
- Preserve source schema and datatypes.
- Add ingestion metadata:
  - `_run_id`
  - `_ingested_at`
  - `_source_table`
  - `_bronze_ts`
- Partitioning:
  - salesorderheader: partition by year(orderdate)
  - salesorderdetail: partition by salesorderid hash/bucket strategy if supported
  - remaining tables: no partitioning due to small dimension size
- Write mode:
  - Delta overwrite with schema evolution enabled.

Mandatory pre-flight validation:
- Run `CREATE SCHEMA IF NOT EXISTS bronze` and immediately verify `SHOW TABLES IN bronze` executes successfully before processing any source table.
- Attach and use the target lakehouse **o** before creating tables.
- Verify the active/default lakehouse is **o** immediately before the first Bronze write; abort the notebook if verification fails.
- Capture and log the active catalog, current database, and lakehouse context before processing the first table.
- Before processing each source table:
  - verify the exact source object exists using the fully qualified name shown in the mapping above.
  - verify the source table is readable with `spark.read.table(...)`.
  - execute a row count on the source dataframe.
  - execute schema introspection and record column names.
  - raise a RuntimeError immediately if the source read fails, the table cannot be resolved, or the dataframe schema is empty.
  - do not continue to the next table after a failed validation.
- Process Bronze tables sequentially in the exact order listed in the mapping above.

Mandatory write/discovery requirements:
- For every source table, write exactly one managed Delta table using:
  - `format('delta')`
  - `mode('overwrite')`
  - `option('overwriteSchema','true')`
  - `saveAsTable('bronze.<exact_target_name>')`
- Tables must be managed lakehouse tables materialized under the target lakehouse Tables area and discoverable as Bronze schema objects. Do not create temporary views, external-only registrations, shortcuts, or catalog entries without persisted Delta data.
- Do not create nested names such as:
  - `bronze.SalesLT_Address`
  - `bronze.saleslt.address`
  - `bronze.bronze_address`
  - any name other than the 10 exact targets listed above.
- Do not use path-based writes (`save(path)` or `save('Tables/...')`) for Bronze outputs.
- Immediately before and immediately after each write, verify the active lakehouse context remains the target lakehouse **o**.
- Immediately after each write:
  - assert `spark.catalog.tableExists('bronze.<table>')`
  - read back the table with `spark.read.table('bronze.<table>')`
  - execute a row count against the read-back dataframe and verify it is greater than or equal to zero.
  - capture row count in results output.
  - verify the table provider is Delta and the table is not temporary.
  - verify table metadata identifies schema `bronze` and a managed table type.
  - verify the table name appears in `SHOW TABLES IN bronze` before continuing to the next table.
  - raise a RuntimeError immediately if any validation fails.
- Maintain an execution manifest containing:
  - source table name
  - target table name
  - source row count
  - written row count
  - validation status
- Before notebook completion:
  - verify the execution manifest contains exactly 10 successful entries.
  - execute `SHOW TABLES IN bronze`
  - verify the discoverable table set equals exactly:
    - address
    - customer
    - customeraddress
    - product
    - productcategory
    - productdescription
    - productmodel
    - productmodelproductdescription
    - salesorderdetail
    - salesorderheader
  - verify all 10 tables are readable via `spark.read.table(...)`.
  - verify all 10 expected names are returned by `SHOW TABLES IN bronze`; count must equal exactly 10.
  - verify the Bronze schema contains 10 persisted managed tables and not views.
  - verify no expected table is missing from catalog enumeration results for schema `bronze`.
  - raise a RuntimeError if any expected table is missing or unreadable.
- Print a final JSON summary containing all 10 table names and row counts.
- Do not mark Bronze successful unless all 10 tables are discoverable, queryable through the Spark catalog, and present in the execution manifest with successful validation status.

## Silver

Standard transformations for all tables:
- Rename columns to snake_case.
- Preserve business keys.
- Add:
  - `_silver_ts`
  - `_record_source`
  - `_run_id`
- Remove exact duplicates.
- Standardize timestamps to UTC-compatible timestamp type.
- Retain rowguid and modified_date for lineage unless explicitly excluded in Gold.

Deduplication strategy:
- address: dedupe on `address_id`, keep latest `modified_date`.
- customer: dedupe on `customer_id`, keep latest `modified_date`.
- customeraddress: dedupe on composite (`customer_id`,`address_id`), keep latest `modified_date`.
- product: dedupe on `product_id`, keep latest `modified_date`.
- productcategory: dedupe on `product_category_id`, keep latest `modified_date`.
- productdescription: dedupe on `product_description_id`, keep latest `modified_date`.
- productmodel: dedupe on `product_model_id`, keep latest `modified_date`.
- productmodelproductdescription: dedupe on (`product_model_id`,`product_description_id`,`culture`), keep latest `modified_date`.
- salesorderheader: dedupe on `sales_order_id`, keep latest `modified_date`.
- salesorderdetail: dedupe on `sales_order_detail_id`, keep latest `modified_date`.

## Gold

Target star schema required by user request.

Dimensions:
- dim_order_date
- dim_ship_date
- dim_customer
- dim_sales_person
- dim_order
- dim_product

Fact:
- fact_sales_order

Use the requirements already defined in this specification.

## Test

Write all results to `test.test_results` using the tests defined in this specification.

## Semantic model

- Storage mode: Direct Lake
- Model tables:
  - dim_order_date
  - dim_ship_date
  - dim_customer
  - dim_sales_person
  - dim_order
  - dim_product
  - fact_sales_order

## Report

Create the report pages and visuals defined in this specification.

## Data Agent

Use the role, domain instructions, starter questions, and guardrails defined in this specification.