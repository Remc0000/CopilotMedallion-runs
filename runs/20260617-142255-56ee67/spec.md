# Run Spec 20260617-142114-6cd029

## Updated specs

### Iteration 1 — 2026-06-17 14:29:39Z — failed layer: silver (run: 20260617-142255-56ee67)
- **Root cause (1-line summary)**: Silver processing terminated with a session-wide cancellation after one statement failed; the traceback does not identify a specific table, so Silver must be hardened for per-table isolation and pre-transform schema validation.
- **Cross-table audit**:
  - customeraddress: yes — junction table has optional audit columns and composite keys that commonly trigger Silver transform failures.
  - salesorderdetail: yes — dedup and derived-column logic may reference missing/renamed columns.
  - productdescription: yes — rename/projection steps can remove columns later referenced.
  - customer: yes — sales_person and email transformations depend on column existence after standardization.
  - productcategory: yes — parent/child category processing can reference renamed keys.
  - productmodel: yes — model attributes may be projected away before downstream use.
  - salesorderheader: yes — date and audit-column logic frequently drives Silver failures.
  - productmodelproductdescription: yes — bridge table often lacks audit columns and uses composite keys.
  - product: yes — product-status derivations depend on source columns surviving projections.
  - address: yes — address-standardization logic depends on expected location columns.
- **Fix approach**: GENERALIZE — the failure signature is session-wide and not tied to a single named table or column, so a uniform Silver validation and isolation rule is safer than table-specific fixes.
- **What was changed**:
  - Tightened Silver processing to require per-table execution with independent try/except handling and schema validation.
  - Added explicit source-to-silver key mappings that must exist after snake_case standardization before deduplication.
  - Required column-existence assertions before every deduplication, derived-column, and table-specific transformation.

## Inputs
- Workspace: `7babcc03-ecd1-49b8-a140-5a5ec6415f8a`
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
- Target Lakehouse: **lakehouse**

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
- Use defensive column references and validate required columns before every join, filter, aggregation, window, and withColumn operation.
- After every join, immediately project alias-qualified columns and rename to flat names.
- Assert groupBy/agg columns exist before aggregation.
- Use defensive REST handling with `if x is None: raise` before any `.get()`.
- Create schemas with `CREATE SCHEMA IF NOT EXISTS`.
- Write only through schema-qualified Delta tables using `saveAsTable('<schema>.<table>')`.
- Never use raw abfss `.save()` for target lakehouse outputs.
- Use parameter cells for workspace, lakehouse, run id, and source tables.
- Use idempotent overwrite patterns with `overwriteSchema=true`.
- Use error-loud try/except blocks that call `_save_error(layer, e)` and re-raise.
- All notebooks must emit discoverable Delta tables in bronze, silver, gold, and test schemas.
- Every code cell must start with a short comment block explaining purpose and intent.

### Global Spark column-reference rules (apply to ALL layers: Bronze, Silver, Gold)
Apply Rules A-L exactly as currently defined in this specification.

ALSO REQUIRE for every generated notebook: EACH code cell must start with a short markdown comment block (Python `# ---` divider + 1-3 lines of `# ` comments) describing what the cell is doing and why — never emit a cell with no leading comment.

## Bronze

Land each source table unchanged into the `bronze` schema:
- bronze.customeraddress
- bronze.salesorderdetail
- bronze.productdescription
- bronze.customer
- bronze.productcategory
- bronze.productmodel
- bronze.salesorderheader
- bronze.productmodelproductdescription
- bronze.product
- bronze.address

Common Bronze behavior:
- Preserve source schema and data types.
- Add ingestion metadata:
  - _run_id
  - _bronze_loaded_at
  - _source_table
- Partition large transactional tables:
  - salesorderheader by year(orderdate)
  - salesorderdetail by salesorderid
- All other tables written unpartitioned.
- Overwrite mode with schema evolution enabled.
- Capture row counts in execution summary output.

## Silver

Standardize all table and column names to snake_case.

Silver execution requirements:
- Process each Silver table independently in a loop: read bronze table → validate schema → transform → write silver table.
- A failure in one table must be logged with `_save_error('silver', e, table=<table>)` and must not prevent schema validation and processing attempts for the remaining Silver tables.
- Before deduplication, derived columns, text normalization, email normalization, or product-status logic, assert that every referenced column exists in the current DataFrame.
- Immediately after snake_case conversion, validate the expected key columns below. Fail with a table-specific error message that includes the available column list.

Required snake_case key columns after standardization:
- customer: `customer_id`
- address: `address_id`
- salesorderheader: `sales_order_id`
- salesorderdetail: `sales_order_detail_id`
- product: `product_id`
- productcategory: `product_category_id`
- productmodel: `product_model_id`
- productdescription: `product_description_id`
- customeraddress: `customer_id`, `address_id`, `address_type`
- productmodelproductdescription: `product_model_id`, `product_description_id`, `culture`

Deduplication strategy:
- customer: customer_id using latest modified_date.
- address: address_id using latest modified_date.
- salesorderheader: sales_order_id using latest modified_date.
- salesorderdetail: sales_order_detail_id using latest modified_date.
- product: product_id using latest modified_date.
- productcategory: product_category_id using latest modified_date.
- productmodel: product_model_id using latest modified_date.
- productdescription: product_description_id using latest modified_date.
- customeraddress: composite key (customer_id, address_id, address_type).
- productmodelproductdescription: composite key (product_model_id, product_description_id, culture).

Additional deduplication safeguards:
- For any table where `modified_date` is absent after standardization, follow Generic Guidance Rule I and use a deterministic fallback ranking that only references columns confirmed to exist.
- Never reference `modified_date` without first checking `'modified_date' in df.columns`.

Silver transformations:
- Remove exact duplicates.
- Standardize text trimming.
- Normalize email addresses to lowercase only when `email_address` exists.
- Create sales_person_username from customer.sales_person:
  - First assert `sales_person` exists on the customer table.
  - Extract username from values like domain\username.
  - Remove trailing numeric suffixes when present only for display attributes.
- Add:
  - _silver_loaded_at
  - _source_modified_date
- Product enrichment preparation:
  - Create product_status flags from sell_start_date, sell_end_date, discontinued_date only after validating each referenced column exists or providing a typed-null fallback per Rule G.
- Address standardization:
  - Normalize city, state_province, country_region only when those columns exist.
- Optimize and vacuum Silver tables after successful loads.

## Gold

Target star schema requested by user.

Dimensions:

1. gold.dim_order_date
- Source: salesorderheader.order_date.
- One row per date.
- Attributes:
  - date_key
  - full_date
  - day
  - month
  - month_name
  - quarter
  - year
  - year_month
- Hierarchy:
  - Year > Quarter > Month > Date

2. gold.dim_ship_date
- Source: salesorderheader.ship_date.
- Same structure as dim_order_date.
- Hierarchy:
  - Year > Quarter > Month > Date

3. gold.dim_customer
- Source: customer + address.
- User requested bypassing customeraddress bridge.
- NOTE: no direct customer-to-address key exists in provided schema. CustomerAddress is the only relationship table available.
- Implement pragmatic model:
  - Preferred build uses customeraddress to resolve address and then projects a flattened customer dimension.
  - Final dimension hides bridge complexity and exposes only relevant customer/address attributes.
- Include:
  - customer_id
  - company_name
  - customer_name
  - email_address
  - phone
  - city
  - state_province
  - country_region
  - postal_code

4. gold.dim_salesperson
- Source: customer.sales_person.
- One row per distinct salesperson.
- Transform:
  - Extract username portion after "\".
  - Remove numeric suffix for display name when appropriate.
- Include:
  - salesperson_key
  - salesperson_username
  - salesperson_display_name

5. gold.dim_order
- Source: salesorderheader.
- Include:
  - sales_order_id
  - revision_number
  - status
  - online_order_flag
  - purchase_order_number
  - account_number
  - ship_method
  - credit_card_approval_code
  - comment

6. gold.dim_product
- Source:
  - product
  - productcategory
  - productmodel
  - productmodelproductdescription
  - productdescription
- Filter productmodelproductdescription to culture='en'.
- Build parent-child category flattening.
- Include:
  - product_id
  - product_name
  - product_number
  - color
  - size
  - weight
  - standard_cost
  - list_price
  - model_name
  - description
  - category_name
  - subcategory_name
  - product_status

7. gold.fact_sales_order
- Grain:
  - One row per sales order detail line.
- Sources:
  - salesorderheader
  - salesorderdetail
- Foreign keys:
  - order_date_key
  - ship_date_key
  - customer_id
  - salesperson_key
  - sales_order_id
  - product_id

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

Required tests:
1. Row Count Reconciliation
2. Gold Dimension PK Null Check
3. Gold Dimension PK Uniqueness
4. Referential Integrity
5. Business Rule Validation

## Semantic model

Mode:
- Direct Lake

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
- OrderDate: Year > Quarter > Month > Date
- ShipDate: Year > Quarter > Month > Date
- Product: Category > Subcategory > Product
- Customer: Country > State/Province > City > Customer

Measures:
- Total Sales = SUM(net_sales_amount)
- Gross Sales = SUM(gross_sales_amount)
- Total Discount Amount = SUM(discount_amount)
- Average Discount Amount = AVERAGE(discount_amount)
- Discount % = DIVIDE([Total Discount Amount],[Gross Sales])
- Average Discount % = AVERAGE(unit_price_discount)
- Total Orders = DISTINCTCOUNT(sales_order_id)
- Total Quantity = SUM(order_qty)
- Average Sales = AVERAGE(net_sales_amount)
- Maximum Sales = MAX(net_sales_amount)
- Average Order Value = DIVIDE([Total Sales],[Total Orders])
- Maximum Order Value = MAXX(VALUES(sales_order_id), CALCULATE([Total Sales]))
- Distinct Customers = DISTINCTCOUNT(customer_id)

Formatting:
- Currency formatting on sales measures.
- Percentage formatting on discount measures.
- Date hierarchies enabled.

## Report

Theme:
- Product-category-driven visual design.
- Category and subcategory imagery, color palettes, and navigation driven from productcategory content.
- Modern executive dashboard layout.

Page 1: Executive Sales Overview
- KPI cards:
  - Total Sales
  - Average Sales
  - Maximum Sales
  - Total Orders
  - Average Order Value

Page 2: Regional Performance

Page 3: Orders & Discount Analysis

Page 4: Product Performance

Page 5: Data Quality

## Data Agent

Role:
- Sales Performance and Regional Analytics Copilot grounded on the Direct Lake semantic model.

Domain hints:
- Sales orders.
- Product categories and subcategories.
- Regional performance.
- Discounts.
- Salesperson effectiveness.
- Customer purchasing behavior.
- Monthly trends.

Starter questions:
- Which regions generated the highest sales this month?
- Which regions generated the lowest sales this quarter?
- What is the average and maximum sales value by region?
- Which product categories contribute most to revenue?
- Which subcategories are growing fastest?
- Which salespeople offered the largest discounts?
- What is the average discount percentage by salesperson?
- How are monthly sales trending over time?
- Which customers generate the most revenue?
- Which orders have the highest net sales amount?

Guardrails:
- Answer only using semantic model data.
- Do not fabricate regions, customers, products, or salespeople.
- Prefer aggregated reporting over row-level sensitive details.
- Surface uncertainty when requested data is unavailable.
- Do not expose password_hash or password_salt fields.
- Use approved business measures from the semantic model whenever possible.
- Respect model relationships and filter context when calculating results.