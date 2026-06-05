# Run Spec 20260605-113141-2256d6

## Updated specs

### Iteration 1 — 2026-06-05 11:38:46Z — failed layer: silver (run: 20260605-113544-195040)
- **Root cause (1-line summary)**: Silver layer terminated with `System_Cancelled_Session_Statements_Failed`, indicating one Silver table failure cancelled the Spark session and prevented completion of remaining Silver outputs.
- **Cross-table audit**:
  - Address: yes — any Silver transform failure can cancel the shared session.
  - Customer: yes — contains derived-column logic and dedup logic that could fail independently.
  - CustomerAddress: yes — junction-table handling and composite-key dedup could fail independently.
  - Product: yes — derived flags from optional columns could fail independently.
  - ProductCategory: yes — dedup logic could fail independently.
  - ProductDescription: yes — dedup logic could fail independently.
  - ProductModel: yes — dedup logic could fail independently.
  - ProductModelProductDescription: yes — junction-table handling and culture filtering could fail independently.
  - SalesOrderDetail: yes — dedup and type-standardization logic could fail independently.
  - SalesOrderHeader: yes — date-derived columns and dedup logic could fail independently.
- **Fix approach**: GENERALIZE — the failure pattern is systemic and can affect every Silver table; enforce per-table Silver execution, write isolation, schema validation, and result tracking for all tables.
- **What was changed**:
  - Tightened the Silver section to require one independent read-transform-write unit per Silver table.
  - Added mandatory per-table schema validation before dedup, derived columns, and writes.
  - Required Silver result tracking and deferred failure reporting only after all Silver tables have been attempted.

### Iteration 1 — 2026-06-05 11:46:23Z — failed layer: reporting (run: 20260605-113544-195040)
- **Root cause (1-line summary)**: Reporting layer ended with `System_Cancelled_Session_Statements_Failed`; a failure in one reporting artifact likely cancelled creation of remaining semantic/reporting assets.
- **Cross-table audit**:
  - Address: yes — contributes indirectly through reporting dimensions and missing outputs can break model generation.
  - Customer: yes — drives Customer and SalesPerson reporting assets.
  - CustomerAddress: yes — may affect Customer geography attributes if used.
  - Product: yes — drives Product dimension and report visuals.
  - ProductCategory: yes — drives Product hierarchy assets.
  - ProductDescription: yes — drives Product descriptive attributes.
  - ProductModel: yes — drives Product dimension enrichment.
  - ProductModelProductDescription: yes — drives Product description assembly.
  - SalesOrderDetail: yes — drives fact measures and report visuals.
  - SalesOrderHeader: yes — drives fact measures, date dimensions, and report visuals.
- **Fix approach**: GENERALIZE — reporting failures caused by a single artifact can cancel the entire reporting build; enforce independent validation and creation of every reporting artifact.
- **What was changed**:
  - Added reporting-layer isolation and artifact validation requirements in Generic guidance.
  - Tightened Semantic model generation to validate required Gold tables and relationships before model publication.
  - Required independent creation and validation of semantic model, report, and data agent assets with result tracking.

### Iteration 2 — 2026-06-05 11:49:23Z — failed layer: reporting (run: 20260605-113544-195040)
- **Root cause (1-line summary)**: Reporting execution was cancelled again; reporting artifacts require stricter pre-validation of every referenced table, column, hierarchy, relationship, and measure before semantic model and report publication.
- **Cross-table audit**:
  - Address: yes — contributes customer geography attributes that may be referenced by reporting assets.
  - Customer: yes — provides Customer and SalesPerson reporting fields.
  - CustomerAddress: yes — may participate in customer geography enrichment.
  - Product: yes — provides product attributes used by visuals and hierarchies.
  - ProductCategory: yes — contributes category hierarchy fields.
  - ProductDescription: yes — contributes product description fields.
  - ProductModel: yes — contributes product-model identifiers.
  - ProductModelProductDescription: yes — contributes description joins and culture filtering.
  - SalesOrderDetail: yes — provides fact measures and detail-level report content.
  - SalesOrderHeader: yes — provides dates, order attributes, and fact relationships.
- **Fix approach**: GENERALIZE — the cancellation pattern can be triggered by any invalid reporting dependency; enforce universal metadata validation across all reporting artifacts.
- **What was changed**:
  - Tightened Generic guidance with mandatory reporting dependency validation and artifact-level logging.
  - Strengthened Semantic model requirements to validate every referenced table, column, relationship, hierarchy, and measure before publish.
  - Strengthened Report and Data Agent requirements so only validated semantic-model objects may be referenced.

## Inputs
- Workspace: `58810d23-9208-474f-899f-119dbfc70bd3`
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
- Target Lakehouse: **a**

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
- Use defensive column references and validate required columns before joins, filters, aggregations, windows, and derived-column logic.
- Alias-qualify all join projections and explicitly rename overlapping columns immediately after joins.
- Assert column existence before every groupBy/agg operation.
- Use defensive REST handling: `if x is None: raise RuntimeError(...)` before any `.get()` access.
- Always create schemas (`bronze`, `silver`, `gold`, `test`) before writes.
- Use schema-qualified writes via `saveAsTable('<schema>.<table>')`.
- Never write target outputs via raw abfss `.save()` paths on the schema-enabled lakehouse.
- Include notebook parameter cells for run_id, workspace_id, source_lakehouse_id, target_lakehouse_name, and processing options.
- Use idempotent overwrite patterns with `overwriteSchema=true`.
- Use error-loud try/except handling that calls `_save_error(layer, e)` and re-raises.
- Process source tables independently with per-table isolation and result tracking.
- Emit discoverable Delta tables in every layer and fail if no tables are produced.
- Every notebook code cell must begin with a short comment block using a `# ---` divider and human-readable purpose comments.
- Reporting artifacts must be built independently: semantic model → report → data agent. Validate each artifact exists before attempting the next artifact.
- Maintain a reporting results registry capturing success/failure for semantic model, report, and data agent creation.
- Do not create reporting assets in a single chained operation; one artifact failure must not prevent validation and attempted creation of remaining artifacts.
- Before creating any reporting artifact, validate that every referenced table, column, hierarchy, relationship, and measure exists in the source metadata. Fail with a descriptive validation error before publication attempts if any dependency is missing.
- Record validation results separately from publication results for semantic model, report, and data agent artifacts.

## Bronze

Land all source tables unchanged into the `bronze` schema with technical metadata.

## Silver

Apply standardized cleansing and conformance.

## Gold

Target star schema aligned to user requirements.

## Test

Write all results to:
- test.test_results

## Semantic model

Storage mode:
- Direct Lake

Build requirements:
- Validate existence and readability of all required Gold tables before semantic model creation.
- Required tables: gold.fact_sales_order, gold.dim_customer, gold.dim_product, gold.dim_salesperson, gold.dim_order, gold.dim_order_date, gold.dim_ship_date.
- Validate every semantic-model table, column, hierarchy level, relationship endpoint, and measure expression against the actual Gold schema before publication.
- Create the semantic model in its own isolated step with dedicated error handling and result logging.
- After publication, validate that all configured tables, relationships, hierarchies, and measures exist before marking the semantic model as successful.
- Do not publish a partial semantic model. If validation fails, log the specific missing dependency and mark semantic-model creation as failed without attempting deployment.

## Report

Build requirements:
- Create the report only after semantic model validation succeeds.
- Generate each report page independently and record page-level success/failure.
- Validate that every visual references an existing semantic-model table, column, hierarchy, or measure before publishing.
- Validate all report dependencies page-by-page before visual creation; unsupported visuals must be skipped with explicit logging rather than causing report-build cancellation.
- After report publication, verify the report is discoverable and bound to the intended semantic model.

### Page 1 — Executive Sales Overview
Visuals:
- KPI: Total Sales
- KPI: Average Sales
- KPI: Maximum Sales
- KPI: Total Orders
- Monthly sales trend line chart
- Sales by salesperson bar chart
- Top products by sales

### Page 2 — Regional Performance
Visuals:
- Map visual using city and postal code
- Bubble size: Total Sales
- Color scale: Average Sales
- Ranked city performance table
- High-performing vs low-performing city chart
- Maximum sales by city

### Page 3 — Orders and Discounts
Visuals:
- Order status breakdown
- Discount % trend over time
- Top salespeople by discount percentage
- Top salespeople by discount amount
- Sales order detail matrix

### Page 4 — Product Performance
Visuals:
- Product sales ranking
- Product category/subcategory hierarchy drilldown
- Quantity by product
- Average and maximum sales by product

### Page 5 — Data Quality
Visuals:
- Test result summary
- Failed test table
- Layer row-count comparison
- Referential integrity status

## Data Agent

Role:
- Sales Performance Intelligence Agent for the SalesLT reporting platform.

Build requirements:
- Create the data agent in a separate step after semantic model validation.
- Validate the agent is bound to the published semantic model before completion.
- Record agent creation success/failure independently from semantic model and report outcomes.
- Validate all referenced measures, hierarchies, and tables against the published semantic model before agent publication.
- Do not publish the agent if semantic-model validation reports unresolved dependencies.

Domain knowledge:
- Customer sales analysis
- Order performance
- Product performance
- Discount analysis
- Salesperson effectiveness
- Geographic performance using city/postal-code data
- Monthly and trend-based sales reporting

Instructions:
- Answer only using the semantic model.
- Prefer measures over raw column aggregation.
- Explain calculations using model measures when asked.
- Distinguish Gross Sales, Discount Amount, and Net Sales.
- For geographic questions, explain that reporting is based on city/postal-code geography from source data.
- Surface data-quality concerns if related tests fail.
- Use date hierarchies when comparing periods.
- Recommend drilldowns from category to product and from city to customer when relevant.
- Never invent regions, countries, territories, or product attributes not present in the model.
- If information depends on unavailable source fields, explicitly state the limitation.

Starter questions:
- Which cities generate the highest sales?
- Which cities have the lowest average sales?
- What are the monthly sales trends this year?
- Which salespeople offer the largest discounts?
- What is the average discount percentage by salesperson?
- Which products generate the most net sales?
- Which customers place the highest-value orders?
- How do gross sales compare to net sales over time?
- What is the maximum sales amount recorded for a single order line?
- Which product categories contribute the most revenue?

Guardrails:
- Do not answer outside the semantic model.
- Do not expose technical IDs unless explicitly requested.
- Do not infer missing geography levels.
- Do not fabricate product model names or category names absent from source data.
- Always prefer aggregated business insights over row-level detail unless requested.