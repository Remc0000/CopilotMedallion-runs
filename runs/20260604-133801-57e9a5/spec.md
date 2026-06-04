# Run Spec 20260604-133728-ace6dc

## Updated specs

### Iteration 1 — 2026-06-04 13:47:13Z — failed layer: reporting (run: 20260604-133801-57e9a5)
- **Root cause (1-line summary)**: Reporting-stage Spark session failed without a column-level traceback; the most likely reporting risk is unresolved semantic-model field references caused by display names not matching Gold table column names.
- **Cross-table audit**:
  - Address: no — source table is not directly referenced by report measures.
  - Customer: yes — customer-related report visuals depend on semantic-model field mappings.
  - CustomerAddress: no — intentionally bypassed in Gold and reporting.
  - Product: yes — product visuals require consistent semantic-model field mappings.
  - ProductCategory: yes — category hierarchy feeds product reporting.
  - ProductDescription: yes — product description attributes may surface in reporting.
  - ProductModel: yes — product dimension construction depends on this source.
  - ProductModelProductDescription: yes — product dimension construction depends on this source.
  - SalesOrderDetail: yes — all sales measures originate from fact-level fields.
  - SalesOrderHeader: yes — order, date, customer, and freight-related reporting fields originate here.
- **Fix approach**: GENERALIZE — the failure occurred in the reporting layer and could affect any report visual or measure that references friendly names instead of actual semantic-model fields.
- **What was changed**:
  - Tightened the Semantic model section with explicit table and field naming requirements.
  - Added explicit measure-to-column mappings and required fact-column names.
  - Added report-authoring constraints requiring visuals to bind only to validated semantic-model fields.

### Iteration 2 — 2026-06-04 13:48:32Z — failed layer: reporting (run: 20260604-133801-57e9a5)
- **Root cause (1-line summary)**: Reporting stage failed again with a generic session-cancelled error; the most likely remaining reporting risk is creation of relationships, hierarchies, measures, or visuals that reference semantic-model objects not successfully created.
- **Cross-table audit**:
  - Address: no — only contributes attributes through dim_customer.
  - Customer: yes — customer hierarchy and regional visuals depend on valid semantic-model fields.
  - CustomerAddress: no — not used in reporting model.
  - Product: yes — product hierarchy and visuals depend on validated model objects.
  - ProductCategory: yes — category/subcategory hierarchy may be referenced by visuals.
  - ProductDescription: yes — product attributes may be exposed in reports.
  - ProductModel: yes — model-related attributes may be referenced if present.
  - ProductModelProductDescription: yes — contributes product descriptive attributes.
  - SalesOrderDetail: yes — source of all fact measures.
  - SalesOrderHeader: yes — source of dates, customer linkage, and order metrics.
- **Fix approach**: GENERALIZE — the same semantic-model dependency issue can affect all reporting objects regardless of source table.
- **What was changed**:
  - Added explicit relationship key mappings that must exist before relationship creation.
  - Required semantic-model object existence validation before creating hierarchies, measures, and report visuals.
  - Required skipping unsupported report elements instead of failing semantic-model or report publication.

## Inputs
- Workspace: `8f7be003-abaa-46b6-8b1c-0dba2cfc3bc6`
- Source Lakehouse: **SalesLT** (`47f5fdf7-1902-471b-958f-5a1e9430070e`)
- Tables to ingest into Bronze:
  - `Address`
  - `Customer`
  - `CustomerAddress`
  - `Product`
  - `ProductCategory`
  - `ProductDescription`
  - `ProductModel`
  - `ProductModelProductDescription`
  - `SalesOrderDetail`
  - `SalesOrderHeader`
- Target Lakehouse: **g**

## Generic guidance

Apply these reference skills/agents at all times:
- FabricDataEngineer agent: https://github.com/microsoft/skills-for-fabric/blob/main/agents/FabricDataEngineer.agent.md
- e2e-medallion-architecture skill: https://github.com/microsoft/skills-for-fabric/tree/main/skills/e2e-medallion-architecture
- spark-authoring-cli skill: https://github.com/microsoft/skills-for-fabric/tree/main/skills/spark-authoring-cli
- powerbi-authoring-cli skill: https://github.com/microsoft/skills-for-fabric/tree/main/skills/powerbi-authoring-cli
- powerbi-consumption-cli skill: https://github.com/microsoft/skills-for-fabric/tree/main/skills/powerbi-consumption-cli
- powerbi-semantic-model-authoring: https://github.com/RuiRomano/powerbi-agentic-plugins/tree/main/plugins/powerbi/skills/powerbi-semantic-model-authoring
- powerbi-report-authoring: https://github.com/RuiRomano/powerbi-agentic-plugins/tree/main/plugins/powerbi/skills/powerbi-report-authoring

## Bronze

Land each source table into the `bronze` schema as Delta tables with schema preservation.

Common metadata columns:
- `ingestion_timestamp`
- `run_id`
- `source_table`
- `source_file_modified_time` (if available)
- `bronze_loaded_at`

Write mode:
- Initial load: overwrite
- Subsequent loads: merge/upsert where practical using business keys and `ModifiedDate`
- Preserve all source columns including binary fields

Partitioning:
- `SalesOrderHeader`: partition by year/month derived from `OrderDate`
- `SalesOrderDetail`: partition by year/month derived after joining to header during optimization strategy, otherwise unpartitioned initially
- All other tables: unpartitioned due to expected small dimension size

Table-specific landing intent remains unchanged.

## Silver

Apply to all tables:
- Rename columns to snake_case
- Standardize timestamps to UTC
- Trim string values
- Add:
  - `silver_created_at`
  - `silver_updated_at`
  - `record_source`
- OPTIMIZE and VACUUM according to Fabric best practices

Deduplication keys and business transformations remain unchanged.

## Gold

Create a business-oriented star schema optimized for sales analytics.

Dimensions and fact definitions remain unchanged, including:
- `dim_order_date`
- `dim_ship_date`
- `dim_customer`
- `dim_salesperson`
- `dim_order`
- `dim_product`
- `fact_sales_order`

## Test

Retain all tests defined in the current specification.

## Semantic model

Mode:
- Direct Lake

Tables:
- Fact Sales Order (source table: `fact_sales_order`)
- Customer (source table: `dim_customer`)
- SalesPerson (source table: `dim_salesperson`)
- Product (source table: `dim_product`)
- Order (source table: `dim_order`)
- OrderDate (source table: `dim_order_date`)
- ShipDate (source table: `dim_ship_date`)

Relationships:
- Create relationships only when both tables and both key columns exist in the published semantic model.
- Required relationship mappings:
  - Fact Sales Order.`customer_key` → Customer.`customer_key`
  - Fact Sales Order.`salesperson_key` → SalesPerson.`salesperson_key`
  - Fact Sales Order.`product_key` → Product.`product_key`
  - Fact Sales Order.`order_key` → Order.`order_key`
  - Fact Sales Order.`order_date_key` → OrderDate.`date_key`
  - Fact Sales Order.`ship_date_key` → ShipDate.`date_key`
- Do not infer alternate key names.
- If any required key is absent, skip that relationship and record a deployment warning rather than failing publication.

Field binding requirements:
- Semantic-model measures, hierarchies, relationships, and report visuals must reference actual semantic-model object names and actual Gold columns only.
- Validate existence of every table, column, hierarchy level, relationship endpoint, and measure dependency before creation.
- Required fact columns:
  - `sales_order_id`
  - `order_quantity`
  - `gross_sales_amount`
  - `discount_amount`
  - `net_sales_amount`
  - `discount_percentage`
  - `freight_allocation`
  - `product_key`
  - `customer_key`
  - `salesperson_key`
- If a required column is missing, do not create dependent measures.

Hierarchies:
- OrderDate: Year → Quarter → Month → Date
- ShipDate: Year → Quarter → Month → Date
- Product: Category → Subcategory → Product
- Customer: City → Customer

Hierarchy creation rule:
- Create a hierarchy only if all referenced levels exist in the target semantic-model table.
- Otherwise skip the hierarchy and continue deployment.

Measures:
- Total Sales = Sum(`net_sales_amount`)
- Gross Sales = Sum(`gross_sales_amount`)
- Total Discount Amount = Sum(`discount_amount`)
- Average Sales = Average(`net_sales_amount`)
- Maximum Sales = Max(`net_sales_amount`)
- Total Orders = DistinctCount(`sales_order_id`)
- Total Quantity Sold = Sum(`order_quantity`)
- Average Discount Percentage = Average(`discount_percentage`)
- Maximum Discount Percentage = Max(`discount_percentage`)
- Average Order Value = Divide([Total Sales], [Total Orders])
- Average Freight = Average(`freight_allocation`)
- Maximum Order Value = Max(`net_sales_amount`)
- Distinct Customers = DistinctCount(`customer_key`)
- Distinct Products Sold = DistinctCount(`product_key`)

## Report

Before creating visuals:
- Validate that every referenced table, hierarchy, measure, relationship, and field exists in the published semantic model.
- Do not bind visuals to inferred field names, friendly captions, or untranslated business labels unless the corresponding semantic-model object exists.
- If a requested field, hierarchy, relationship, or measure is unavailable, omit the dependent visual and record the limitation rather than failing report generation.
- Report publication must succeed even when one or more optional visuals are skipped.

Page definitions remain unchanged:
- Executive Sales Overview
- Regional Performance
- Orders and Discounts
- Product Performance
- Data Quality

## Data Agent

Role:
- Sales Performance Intelligence Agent for the SalesLT sales analytics platform.

Guardrails:
- Use only data exposed through the semantic model.
- Do not reference semantic-model tables, measures, hierarchies, or relationships that were not successfully published.
- Clearly identify when requested information is unavailable.
- All other existing guardrails remain unchanged.