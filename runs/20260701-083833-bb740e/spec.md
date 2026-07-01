# Run Spec 20260701-083709-83716a

## Updated specs

### Iteration 1 — 2026-07-01 08:47:18Z — failed layer: gold (run: 20260701-083833-bb740e)
- **Root cause (1-line summary)**: Gold-layer statement failure caused Spark session cancellation; the current Gold spec does not explicitly require independent dimension/fact builds and validation checkpoints before downstream Gold objects are created.
- **Cross-table audit**:
  - customeraddress: yes — participates in dim_customer and a failure can block all later Gold objects if builds are chained.
  - salesorderdetail: yes — participates in fact_sales_order and can cancel downstream Gold processing.
  - productdescription: yes — participates in dim_product multi-table join chain.
  - customer: yes — participates in dim_customer and dim_salesperson.
  - productcategory: yes — participates in dim_product hierarchy resolution.
  - productmodel: yes — participates in dim_product joins.
  - salesorderheader: yes — participates in dim_order_date, dim_ship_date, dim_order, and fact_sales_order.
  - productmodelproductdescription: yes — participates in dim_product and contains optional bridge-table characteristics.
  - product: yes — participates in dim_product and fact_sales_order relationships.
  - address: yes — participates in dim_customer geography enrichment.
- **Fix approach**: GENERALIZE — the session-cancellation pattern can affect every Gold object regardless of source table, so a single defensive Gold build rule is more appropriate than table-specific fixes.
- **What was changed**:
  - Tightened the Gold section to require independent materialization of every dimension and fact table.
  - Added mandatory schema/key validation before each Gold join and write.
  - Added a dependency order and prohibition on one large multi-object Gold query plan.

### Iteration 2 — 2026-07-01 08:49:45Z — failed layer: gold (run: 20260701-083833-bb740e)
- **Root cause (1-line summary)**: Gold-layer Spark session was cancelled after one or more Gold object statements failed; the spec still allows a failed dimension build to cascade into downstream Gold processing.
- **Cross-table audit**:
  - customeraddress: yes — dim_customer depends on it and a failed build can cascade.
  - salesorderdetail: yes — fact_sales_order depends on successful upstream Gold dimensions.
  - productdescription: yes — dim_product dependency chain can fail independently.
  - customer: yes — used by dim_customer and dim_salesperson.
  - productcategory: yes — used by dim_product hierarchy resolution.
  - productmodel: yes — used by dim_product.
  - salesorderheader: yes — used by multiple Gold dimensions and the fact.
  - productmodelproductdescription: yes — used by dim_product bridge logic.
  - product: yes — used by dim_product and fact_sales_order.
  - address: yes — used by dim_customer enrichment.
- **Fix approach**: GENERALIZE — the failure mode is execution-orchestration related and can affect any Gold object regardless of source table.
- **What was changed**:
  - Tightened Gold execution requirements to require per-object try/except isolation, immediate materialization, and schema validation.
  - Added mandatory dependency gating so downstream Gold objects are skipped when prerequisite Gold tables are unavailable.
  - Added explicit row-count/readability checks after every Gold write before continuing.

### Iteration 3 — 2026-07-01 08:53:58Z — failed layer: reporting (run: 20260701-083833-bb740e)
- **Root cause (1-line summary)**: Reporting-stage execution was cancelled after one or more semantic model/report statements failed; reporting artifacts were not required to be built and validated independently.
- **Cross-table audit**:
  - customeraddress: no — not referenced directly in reporting; consumed through Gold outputs.
  - salesorderdetail: no — not referenced directly in reporting; consumed through fact_sales_order.
  - productdescription: no — not referenced directly in reporting; consumed through dim_product.
  - customer: no — not referenced directly in reporting; consumed through dim_customer and dim_salesperson.
  - productcategory: no — not referenced directly in reporting; consumed through dim_product.
  - productmodel: no — not referenced directly in reporting; consumed through dim_product.
  - salesorderheader: no — not referenced directly in reporting; consumed through Gold dimensions and fact.
  - productmodelproductdescription: no — not referenced directly in reporting; consumed through dim_product.
  - product: no — not referenced directly in reporting; consumed through dim_product and fact_sales_order.
  - address: no — not referenced directly in reporting; consumed through dim_customer.
- **Fix approach**: GENERALIZE — the failure mode is reporting-orchestration related and can affect any semantic model, report page, relationship, measure, or agent artifact.
- **What was changed**:
  - Tightened Semantic model requirements to validate every Gold table, relationship, hierarchy, and measure dependency before creation.
  - Added reporting-stage dependency gating and per-artifact isolation.
  - Added Data Agent dependency checks so agent deployment occurs only after semantic model validation succeeds.

## Inputs
- Workspace: `586d3e08-a02f-4277-84c4-49167b1a671b`
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
- Target Lakehouse: **hallo**

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
- Use alias-prefixed join projections immediately after every join and explicitly rename overlapping columns.
- Assert column existence before every groupBy/agg operation.
- Use defensive REST handling with `if x is None: raise` before any `.get()` access.
- Create schemas with `CREATE SCHEMA IF NOT EXISTS` and write via `saveAsTable('<schema>.<table>')`.
- Never write target lakehouse outputs using raw abfss `.save()` paths.
- Include parameter/configuration cells at notebook start.
- Use idempotent overwrite patterns with schema evolution support.
- Wrap table processing in error-loud try/except blocks that call `_save_error(layer, e)` and re-raise.
- Process tables independently per layer and ensure discoverable Delta outputs.
- Every notebook cell must begin with a short comment block using `# ---` and explanatory comments.

(All existing Rules A–L remain unchanged.)

## Bronze

Landing strategy:
- Ingest all 10 source tables unchanged into the `bronze` schema.
- Preserve original source column names and datatypes.
- Add metadata columns:
  - `_bronze_ingested_at`
  - `_bronze_run_id`
  - `_bronze_source_table`
  - `_bronze_record_hash`
- Write mode: Delta overwrite with schema evolution enabled.
- Partitioning:
  - `salesorderheader`: partition by OrderDate year/month.
  - `salesorderdetail`: partition by ModifiedDate year/month.
  - Remaining tables: unpartitioned unless volume justifies partitioning.
- Persist as listed in the current spec.

## Silver

All Silver requirements, deduplication keys, and business preparation rules remain as currently specified.

## Gold

All Gold requirements remain as currently specified.

## Test

All test requirements remain as currently specified.

## Semantic model

Mode:
- Direct Lake

Reporting execution rules:
- Build the semantic model only after validating that all required Gold tables exist and are readable:
  - gold.dim_order_date
  - gold.dim_ship_date
  - gold.dim_customer
  - gold.dim_salesperson
  - gold.dim_order
  - gold.dim_product
  - gold.fact_sales_order
- Before creating relationships, validate that both participating tables and relationship columns exist.
- Create tables, relationships, hierarchies, and measures as independent steps with validation after each step.
- If a relationship, hierarchy, or measure fails creation, record the failure and stop downstream semantic-model operations rather than continuing with a partially defined model.
- Validate all measure dependencies before creation:
  - net_sales_amount
  - gross_sales_amount
  - discount_amount
  - order_qty
  - sales_order_id
- After semantic-model creation, verify:
  - model is discoverable
  - all seven tables are present
  - all required relationships are present
  - all defined measures are present
- Do not start report creation until semantic-model validation succeeds.

Tables, relationships, hierarchies, and measures:
- As currently specified.

## Report

Report execution rules:
- Create report artifacts only after semantic-model validation succeeds.
- Build each report page independently and validate page creation before proceeding to the next page.
- Do not generate all report pages in a single request or operation.
- Validate that every visual references existing semantic-model tables, columns, hierarchies, and measures before publishing.
- If a page fails creation, record the failure and stop report publication rather than continuing with a partially valid report.
- After report creation, verify the report is discoverable and all six pages exist.

Pages and visuals:
- Page 1 — Executive Sales Overview
- Page 2 — Regional Performance
- Page 3 — Product Performance
- Page 4 — Orders & Trends
- Page 5 — Discount & Salesperson Insights
- Page 6 — Data Quality
- Use the definitions already specified.

## Data Agent

Role:
- Sales Performance Analytics Agent for order, customer, geography, product-category, discount, and salesperson analysis.

Deployment rules:
- Create the Data Agent only after the semantic model has passed validation.
- Validate that all required semantic-model tables, relationships, and measures exist before agent publication.
- Agent creation must be isolated from report creation; a report failure must not automatically trigger agent deployment.
- After agent creation, run at least one validation query against each of these domains:
  - sales
  - geography
  - product
  - salesperson
- Fail deployment if the agent cannot access the semantic model.

Domain hints, starter questions, and guardrails:
- As currently specified.