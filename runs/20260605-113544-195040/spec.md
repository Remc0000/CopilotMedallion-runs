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
- powerbi-report-authoring: https://github.com/RuiRomano/powerbi-agentic-plugins/tree/main/plugins/powerbi/skills/powerbi-report-authoring

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

## Bronze

Land all source tables unchanged into the `bronze` schema with technical metadata.

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

Common metadata:
- ingestion_ts
- run_id
- source_table
- source_lakehouse
- bronze_load_date

Partitioning:
- salesorderheader partition by year(orderdate)
- salesorderdetail partition by salesorderid hash strategy if supported
- remaining tables unpartitioned due to small dimension size

Write mode:
- Delta overwrite with schema evolution enabled
- saveAsTable into bronze schema

## Silver

Apply standardized cleansing and conformance.

Execution requirements (mandatory for all Silver tables):
- Build each Silver table in a completely independent unit of work: read bronze table → validate schema → transform → write Silver table.
- Maintain a Silver results registry capturing success/failure per table.
- Wrap every Silver table build in its own try/except block and record the error via `_save_error('silver', e, table=<table_name>)`.
- Attempt all Silver tables before raising a final aggregated Silver-layer failure.
- After writing each Silver table, immediately verify the table exists and is readable from the `silver` schema.
- Before deduplication, assert all configured dedup-key columns exist for that specific table.
- Before every derived-column transformation, assert source columns exist or apply an explicit fallback path.

Common transformations:
- Rename all columns to snake_case.
- Preserve source business keys.
- Add silver_created_ts.
- Standardize timestamps.
- Remove exact duplicate rows.
- Retain rowguid for lineage but exclude from most Gold dimensions.

Silver tables and dedup strategy:
- silver.address
  - Dedup key: address_id
  - Keep latest modified_date
- silver.customer
  - Dedup key: customer_id
  - Keep latest modified_date
  - Create cleaned_sales_person from sales_person
  - Extract username from values like domain\username
  - Remove domain prefix and numeric suffix where possible for display purposes
- silver.customeraddress
  - Dedup key: (customer_id, address_id)
- silver.product
  - Dedup key: product_id
  - Create is_discontinued flag from discontinued_date
  - Create is_active_product flag
- silver.productcategory
  - Dedup key: product_category_id
- silver.productdescription
  - Dedup key: product_description_id
- silver.productmodel
  - Dedup key: product_model_id
  - NOTE: requested Product dimension asks for ProductModel Name. No Name column exists in ProductModel schema. Gold model will use ProductModelID only unless an upstream source is extended.
- silver.productmodelproductdescription
  - Dedup key: (product_model_id, product_description_id, culture)
  - Filter culture='en' during Gold assembly
- silver.salesorderheader
  - Dedup key: sales_order_id
  - Derive order_year, order_month, ship_year, ship_month
- silver.salesorderdetail
  - Dedup key: sales_order_detail_id

Performance:
- OPTIMIZE all Silver tables.
- ZORDER salesorderheader on customer_id and order_date.
- ZORDER salesorderdetail on product_id and sales_order_id.

## Gold

Target star schema aligned to user requirements.

Dimensions:

### gold.dim_order_date
Source:
- salesorderheader.order_date

Attributes:
- date_key
- full_date
- year
- quarter
- month
- month_name
- week
- day

Hierarchy:
- Year → Quarter → Month → Date

### gold.dim_ship_date
Source:
- salesorderheader.ship_date

Attributes:
- date_key
- full_date
- year
- quarter
- month
- month_name
- week
- day

Hierarchy:
- Year → Quarter → Month → Date

### gold.dim_customer
Source:
- customer joined directly to address per user request

Join approach:
- CustomerID matched to AddressID only if business validation confirms relationship.
- NOTE: schema does not contain a direct Customer-to-Address key. The available relational path is Customer → CustomerAddress → Address. User requested not to use CustomerAddress. This relationship is therefore not directly supported by the provided schema. Build should either:
  - use CustomerAddress despite the preference, or
  - create a customer-only dimension.
- Default implementation: customer-only attributes until clarified.

Relevant attributes:
- customer_id
- company_name
- title
- email_address
- city
- postal_code

Geography fields:
- city
- postal_code

### gold.dim_salesperson
Source:
- customer.sales_person

Attributes:
- salesperson_key
- salesperson_name
- original_sales_person

Transform:
- Extract username from domain\username pattern.
- Remove domain prefix.
- Use cleaned username as reporting attribute.

Hierarchy:
- SalesPerson

### gold.dim_order
Source:
- salesorderheader

Attributes:
- sales_order_id
- revision_number
- status
- ship_method
- credit_card_approval_code
- comment

Purpose:
- Move descriptive order attributes out of fact table.

### gold.dim_product
Source:
- product
- productcategory
- productmodel
- productmodelproductdescription
- productdescription

Business logic:
- Filter ProductModelProductDescription to culture='en'.
- Join Product → ProductCategory.
- Resolve parent-child category structure into:
  - category_id
  - category_name (if available)
  - subcategory_id
  - subcategory_name (if available)
- Join Description from ProductDescription.
- Join ProductModel.

NOTE:
- ProductCategory contains IDs only and no category names.
- ProductModel contains no Name column.
- Product dimension will expose available identifiers and English description; category/model names cannot be produced from provided schema.

Relevant attributes:
- product_id
- product_number
- color
- size
- weight
- standard_cost
- list_price
- description
- product_category_id
- parent_product_category_id
- product_model_id
- is_active_product

Hierarchy:
- Category → Subcategory → Product

### gold.fact_sales_order

Source:
- salesorderheader joined to salesorderdetail

Grain:
- One row per sales order detail line.

Keys:
- sales_order_id
- sales_order_detail_id
- customer_id
- product_id
- salesperson_key
- order_date_key
- ship_date_key

Measures:
- order_qty
- unit_price
- unit_price_discount
- gross_sales_amount = order_qty * unit_price
- discount_amount = order_qty * unit_price * unit_price_discount
- net_sales_amount = gross_sales_amount - discount_amount
- subtotal
- tax_amt
- freight

Fact/dimension joins:
- CustomerID → dim_customer
- ProductID → dim_product
- SalesPerson → dim_salesperson
- OrderDate → dim_order_date
- ShipDate → dim_ship_date
- SalesOrderID → dim_order

Regional reporting note:
- Region-level reporting is limited by available geography. Source contains city and postal code only. No state, province, territory, or country columns are available.

## Test

Write all results to:
- test.test_results

Schema:
- run_id
- test_name
- layer
- table_name
- status
- actual
- expected
- details
- checked_at

Tests:

1. Row count reconciliation
- Compare Bronze vs Silver counts.
- Expected variance ≤ 1%.

2. Gold dimension PK not null
- dim_customer.customer_id
- dim_product.product_id
- dim_order.sales_order_id
- dim_order_date.date_key
- dim_ship_date.date_key
- dim_salesperson.salesperson_key

3. Gold dimension PK uniqueness
- Validate uniqueness of all dimension primary keys.

4. Referential integrity
- fact_sales_order.product_id exists in dim_product
- fact_sales_order.customer_id exists in dim_customer
- fact_sales_order.salesperson_key exists in dim_salesperson
- fact_sales_order.order_date_key exists in dim_order_date
- fact_sales_order.ship_date_key exists in dim_ship_date

5. Business-rule sanity check
- net_sales_amount <= gross_sales_amount
- discount_amount >= 0
- unit_price >= 0
- order_qty > 0

## Semantic model

Storage mode:
- Direct Lake

Build requirements:
- Validate existence and readability of all required Gold tables before semantic model creation.
- Required tables: gold.fact_sales_order, gold.dim_customer, gold.dim_product, gold.dim_salesperson, gold.dim_order, gold.dim_order_date, gold.dim_ship_date.
- Create the semantic model in its own isolated step with dedicated error handling and result logging.
- After publication, validate that all configured tables, relationships, hierarchies, and measures exist before marking the semantic model as successful.

Tables:
- Fact Sales Order
- Customer
- Product
- SalesPerson
- Order
- OrderDate
- ShipDate

Relationships:
- Fact Sales Order → Customer
- Fact Sales Order → Product
- Fact Sales Order → SalesPerson
- Fact Sales Order → Order
- Fact Sales Order → OrderDate
- Fact Sales Order → ShipDate

Hierarchies:
- OrderDate: Year → Quarter → Month → Date
- ShipDate: Year → Quarter → Month → Date
- Product: Category → Subcategory → Product
- Customer: City → Customer
- SalesPerson: SalesPerson

Measures:
- Total Sales = SUM(net_sales_amount)
- Gross Sales = SUM(gross_sales_amount)
- Total Discount Amount = SUM(discount_amount)
- Average Sales = AVERAGE(net_sales_amount)
- Maximum Sales = MAX(net_sales_amount)
- Total Orders = DISTINCTCOUNT(sales_order_id)
- Total Quantity = SUM(order_qty)
- Average Order Value = DIVIDE([Total Sales],[Total Orders])
- Discount % = DIVIDE([Total Discount Amount],[Gross Sales])
- Average Discount % = AVERAGE(unit_price_discount)
- Maximum Discount % = MAX(unit_price_discount)
- Average Unit Price = AVERAGE(unit_price)
- Maximum Unit Price = MAX(unit_price)

Reporting focus measures:
- Monthly Sales
- Sales by City
- Sales by SalesPerson
- Highest Discount SalesPerson

## Report

Build requirements:
- Create the report only after semantic model validation succeeds.
- Generate each report page independently and record page-level success/failure.
- Validate that every visual references an existing semantic-model table, column, hierarchy, or measure before publishing.
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