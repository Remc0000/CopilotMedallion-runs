# Run Spec 20260605-083031-0b36b8

## Updated specs

### Iteration 1 — 2026-06-05 08:39:13Z — failed layer: gold (run: 20260605-083107-a698ce)
- **Root cause (1-line summary)**: Gold-layer build terminated with a session-cancelled error after one or more gold statements failed; the most likely preventable cause is unresolved/ambiguous columns during multi-table dimension and fact joins.
- **Cross-table audit**:
  - Address: yes — contributes city/postal_code attributes that can collide after joins.
  - Customer: yes — contributes customer_id and salesperson fields used in multiple gold objects.
  - CustomerAddress: yes — bridge-table keys can create duplicate customer/address column references if joined.
  - Product: yes — product_id joins to several product-related tables.
  - ProductCategory: yes — parent/child category identifiers are prone to ambiguous naming in self-referencing joins.
  - ProductDescription: yes — description attributes are joined through ProductModelProductDescription.
  - ProductModel: yes — product_model_id participates in chained joins.
  - ProductModelProductDescription: yes — bridge-table columns and culture filter are commonly referenced after projections.
  - SalesOrderDetail: yes — sales_order_id and product_id overlap with other gold sources.
  - SalesOrderHeader: yes — sales_order_id, customer_id, address keys, and date keys overlap with joined tables.
- **Fix approach**: GENERALIZE — the failure signature does not identify a single table; all gold dimensions and facts rely on multi-table joins and can encounter the same unresolved/ambiguous-column pattern.
- **What was changed**:
  - Tightened Gold join rules to require alias-qualified joins followed by immediate projection to canonical column names.
  - Added required column-presence validation before every gold dimension/fact build.
  - Defined canonical source tables and join keys for each gold object to prevent implicit column resolution.

### Iteration 2 — 2026-06-05 08:43:34Z — failed layer: reporting (run: 20260605-083107-a698ce)
- **Root cause (1-line summary)**: Reporting layer session was cancelled after one or more semantic-model/report statements referenced missing tables, columns, hierarchies, relationships, or measures that were not successfully materialized in Gold.
- **Cross-table audit**:
  - Address: no — not referenced directly by reporting artifacts.
  - Customer: no — reporting consumes dim_customer rather than the source table.
  - CustomerAddress: no — not exposed directly to reporting.
  - Product: no — reporting consumes dim_product rather than the source table.
  - ProductCategory: no — reporting consumes dim_product hierarchy rather than the source table.
  - ProductDescription: no — reporting consumes dim_product attributes rather than the source table.
  - ProductModel: no — reporting consumes dim_product attributes rather than the source table.
  - ProductModelProductDescription: no — reporting consumes dim_product attributes rather than the source table.
  - SalesOrderDetail: no — reporting consumes fact_sales_order rather than the source table.
  - SalesOrderHeader: no — reporting consumes gold dimensions/facts rather than the source table.
- **Fix approach**: GENERALIZE — the root cause is reporting metadata referencing unavailable Gold objects; the same validation pattern applies to every semantic-model table, relationship, hierarchy, measure, and report visual.
- **What was changed**:
  - Tightened Semantic model requirements to validate table and column existence before creating relationships, hierarchies, and measures.
  - Added mandatory conditional creation rules so reporting artifacts only reference validated Gold objects.
  - Required reporting notebooks to fail with explicit missing-object diagnostics instead of allowing session-wide cancellation.

### Iteration 3 — 2026-06-05 08:45:15Z — failed layer: reporting (run: 20260605-083107-a698ce)
- **Root cause (1-line summary)**: Reporting session cancellation is most likely caused by semantic-model/report definitions attempting to create relationships, hierarchies, measures, or visuals against columns whose names differ from the actual Gold output schema.
- **Cross-table audit**:
  - Address: no — reporting does not bind directly to source tables.
  - Customer: no — reporting consumes gold.dim_customer only.
  - CustomerAddress: no — not exposed to reporting.
  - Product: no — reporting consumes gold.dim_product only.
  - ProductCategory: no — reporting consumes gold.dim_product hierarchy only.
  - ProductDescription: no — reporting consumes gold.dim_product attributes only.
  - ProductModel: no — reporting consumes gold.dim_product attributes only.
  - ProductModelProductDescription: no — reporting consumes gold.dim_product attributes only.
  - SalesOrderDetail: no — reporting consumes gold.fact_sales_order only.
  - SalesOrderHeader: no — reporting consumes gold dimensions/facts only.
- **Fix approach**: GENERALIZE — the same reporting failure can occur for any semantic-model object that references a non-existent table or column; a single schema-validation rule should be applied to all reporting artifacts.
- **What was changed**:
  - Tightened Semantic model section to require runtime schema discovery from Gold before creating relationships, hierarchies, or measures.
  - Added explicit column requirements for every relationship endpoint.
  - Tightened Report section so visuals are created only from validated semantic-model fields and skipped with diagnostics when fields are unavailable.

## Inputs

- Workspace: `1aec2347-3511-4e15-91d0-aeda945b41d8`
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
- Target Lakehouse: **p**

## Generic guidance

[UNCHANGED FROM PREVIOUS SPEC]

## Bronze

[UNCHANGED FROM PREVIOUS SPEC]

## Silver

[UNCHANGED FROM PREVIOUS SPEC]

## Gold

[UNCHANGED FROM PREVIOUS SPEC]

## Test

[UNCHANGED FROM PREVIOUS SPEC]

## Semantic model

Mode:
- Direct Lake

Pre-deployment validation (mandatory):
- Before creating the semantic model, verify that all referenced Gold tables physically exist:
  - gold.dim_order_date
  - gold.dim_ship_date
  - gold.dim_customer
  - gold.dim_salesperson
  - gold.dim_order
  - gold.dim_product
  - gold.fact_sales_order
- Runtime-discover the schema of each Gold table immediately before semantic-model creation. Do not assume a column exists because it was requested in the specification.
- For every relationship, hierarchy, measure, and report dependency, validate both:
  - table existence
  - column existence
- Relationship creation rules:
  - fact_sales_order.customer_id ↔ dim_customer.customer_id only if both columns exist.
  - fact_sales_order.product_id ↔ dim_product.product_id only if both columns exist.
  - fact_sales_order.salesperson_key ↔ dim_salesperson.salesperson_key only if both columns exist.
  - fact_sales_order.order_dim_key ↔ dim_order.sales_order_id only if both columns exist.
  - fact_sales_order.order_date_key ↔ dim_order_date.date_key only if both columns exist.
  - fact_sales_order.ship_date_key ↔ dim_ship_date.date_key only if both columns exist.
- Hierarchy creation rules:
  - Create a hierarchy only when every level column exists in the target table.
  - If any hierarchy level is missing, skip hierarchy creation and emit a diagnostic naming the missing column.
- Measure creation rules:
  - Validate all referenced fact columns before measure creation.
  - Skip only the invalid measure and continue processing remaining measures.
- Build semantic-model objects independently so one failed relationship, hierarchy, or measure does not cancel the entire reporting session.
- Emit a validation inventory listing:
  - discovered tables
  - discovered columns per table
  - created relationships
  - skipped relationships
  - created measures
  - skipped measures

Tables:
- dim_order_date
- dim_ship_date
- dim_customer
- dim_salesperson
- dim_order
- dim_product
- fact_sales_order

Relationships:
- fact_sales_order.customer_id → dim_customer.customer_id
- fact_sales_order.product_id → dim_product.product_id
- fact_sales_order.salesperson_key → dim_salesperson.salesperson_key
- fact_sales_order.order_dim_key → dim_order.sales_order_id
- fact_sales_order.order_date_key → dim_order_date.date_key
- fact_sales_order.ship_date_key → dim_ship_date.date_key

Hierarchies:
- Create only after validating all hierarchy columns exist.

Measures:
- Create only after validating referenced columns exist in fact_sales_order.

## Report

Reporting build safety rules:
- Validate every semantic-model table, relationship, hierarchy, measure, and field exists before binding visuals.
- Runtime-discover available fields from the deployed semantic model and bind visuals only to validated fields.
- Do not create a visual that references a missing table, hierarchy, measure, or column.
- If a required field for a visual is missing:
  - record a diagnostic,
  - skip that visual,
  - continue processing remaining visuals and pages.
- Generate pages independently; capture page-level failures and continue validating remaining pages.
- Produce a final report-build summary containing:
  - created pages,
  - created visuals,
  - skipped visuals,
  - missing fields,
  - missing measures.
- Fail only after diagnostics have been collected; do not allow a single invalid visual definition to cancel the entire Spark reporting session.

Page 1 — Executive Sales Overview
- Build only from validated measures and fields.

Page 2 — Regional Performance
- Build only from validated geography fields available in the semantic model.

Page 3 — Orders and Discounts
- Build only from validated measures and fields.

Page 4 — Product Performance
- Build only from validated product hierarchy fields and measures.

Page 5 — Data Quality
- Build only from validated test and metadata fields.

## Data Agent

[UNCHANGED FROM PREVIOUS SPEC]