# Run Spec 20260515-180839-ba48c8

## Inputs
- Workspace: `c55162e7-0ef0-4efc-8343-5aa5e35bac3a`
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
- Target Lakehouse: **e2egpt54**

## Medallion build

You are a data agent following the FabricDataEngineer agent (https://github.com/microsoft/skills-for-fabric/blob/main/agents/FabricDataEngineer.agent.md) and the e2e-medallion-architecture skill.

Build 3 Spark notebooks (one per layer) — bronze, silver, gold — in a single lakehouse with schemas as the layer separator.

- **Bronze**: 1:1 ingestion of the selected source tables into the `bronze` schema, adding `ingestion_timestamp`, `source_path`, `batch_id`, `ingestion_date` metadata columns. Mode: overwrite + overwriteSchema=true. Partitioned by `ingestion_date`.
- **Silver**: dedupe, snake_case rename, drop fully-null columns, trim strings, add `ingestion_dt` and `source_dt` audit columns, `_silver_ts` timestamp. Mode: overwrite. After write run OPTIMIZE.
- **Gold** (AdventureWorksLT shape):
  - **Dim_Customer** — join `silver.customer` as `c` + `silver.customer_address` as `ca` + `silver.address` as `a` using `c.customer_id = ca.customer_id` and `ca.address_id = a.address_id`; keep all relevant customer/address fields. To avoid ambiguous-column failures, alias every input DataFrame and fully qualify all selected/joined columns with the alias prefix. In the final projection, use `ca.address_id` as the relationship/junction key and only include a single output column named `address_id`; any physical address attributes (for example address lines, city, state/province, country/region, postal code) must come from alias `a`. Do not use bare `address_id` anywhere in joins, selects, filters, or tests after the join.
  - **Dim_Product** — join product + productcategory (self-join: subcategory → parent category) + productdescription + productmodel + productmodelproductdescription; expose `category` and `subcategory`.
  - **Dim_Sales** — degenerate dimension on the cleaned salesperson from salesorderheader (strip leading `adventureworks/`, strip trailing digits).
  - **Fact_Sales** — join salesorderdetail + salesorderheader, keep all relevant fields, AND include the cleaned `sales_person` column (same cleaning as Dim_Sales) so reports can aggregate by it.

### Data quality tests (in the gold notebook)

Add a `test` schema with table `test_results` (run_id, test_name, layer, table_name, status, actual, expected, details, checked_at). APPEND rows so history is preserved. Run at minimum:

1. Row counts per layer (bronze ≈ silver within 1%).
2. No-null primary keys in gold dims.
3. Unique primary keys in gold dims.
4. Referential integrity (every fact_sales FK exists in its dim).
5. Salesperson cleaning sanity (no `adventureworks/`, no trailing digit).

## Semantic model

Apply: https://github.com/RuiRomano/powerbi-agentic-plugins/tree/main/plugins/powerbi/skills/powerbi-semantic-model-authoring (Direct Lake guidelines, modeling-guidelines, TMSL/TMDL syntax).

Create a **Direct Lake** semantic model named `{target_lakehouse_name}_sm` on the gold tables, with:
- Tables: `dim_customer`, `dim_product`, `dim_sales`, `fact_sales`.
- Relationships:
  - `fact_sales[customer_id]` → `dim_customer[customer_id]` (many-to-one)
  - `fact_sales[product_id]` → `dim_product[product_id]` (many-to-one)
  - `fact_sales[sales_person]` → `dim_sales[sales_person]` (many-to-one)
- Explicit measures (PascalCase, with format strings):
  - `[Total Sales] = SUM(fact_sales[line_total])` — currency
  - `[Total Discount Amount] = SUMX(fact_sales, fact_sales[line_total] * fact_sales[unit_price_discount])` — currency
  - `[Discount %] = DIVIDE([Total Discount Amount], [Total Sales])` — percent
  - `[Distinct Salespersons] = DISTINCTCOUNT(dim_sales[sales_person])`
  - `[Sales Count] = COUNTROWS(fact_sales)`
- Mark `fact_sales[order_date]` as a date column so time intelligence works without a separate date dim.
- Hide all FK columns from the report view.

- create hierarchies in all dimensions

## Report

Apply: https://github.com/RuiRomano/powerbi-agentic-plugins/tree/main/plugins/powerbi/skills/powerbi-report-authoring (PBIR format).

Create a Power BI report named `{target_lakehouse_name}_rpt` bound to the semantic model above. Pages:

1. **Sales overview** — card visuals for `[Total Sales]`, `[Distinct Salespersons]`, `[Sales Count]`; bar chart "Top salespersons by sales" (axis = `dim_sales[sales_person]`, value = `[Total Sales]`, sort desc, top 10); bar chart "Top salespersons by discount given" (axis = `dim_sales[sales_person]`, value = `[Total Discount Amount]`, sort desc, top 10); line chart sales by month.
2. **Product mix** — matrix of `[Total Sales]` by `dim_product[category]` rows × `dim_product[subcategory]` columns; bar chart top 10 products by `[Total Sales]`.
3. **Data quality** — table visual showing the latest `test_results` (filter to MAX(run_id)), columns: layer, table_name, test_name, status, actual, expected; conditional formatting on `status` (green=PASS, red=FAIL, amber=ERROR).

## Data Agent

Create a **Data Agent (AISkill)** named `{target_lakehouse_name}_agent`, **grounded on the semantic model** created above (not on the raw tables), so it answers through curated measures and relationships.

**System instructions for the agent:**

> You are a sales analytics assistant for the AdventureWorksLT business. Answer questions about sales performance, salespeople, products, customers, and data quality. Always use the curated semantic-model measures (`Total Sales`, `Total Discount Amount`, `Discount %`, `Distinct Salespersons`, `Sales Count`) and the cleaned `dim_sales[sales_person]` dimension. NEVER aggregate columns directly from `fact_sales` when an equivalent measure exists.
>
> When asked "who", return the cleaned salesperson name (no `adventureworks/` prefix, no trailing digit). When asked "how much", format numbers as currency with two decimals. When asked about data quality, query the latest `test_results` (filter to MAX(run_id)) and summarise PASS/FAIL counts per layer and table. If the question cannot be answered from this model, say so clearly and suggest a follow-up. Be concise (3-5 sentences) unless the user asks for a deep dive.

**Domain context to inject:**

- "Salesperson names are stored cleaned in `dim_sales[sales_person]`; the raw value in `salesorderheader[sales_person]` had an `adventureworks/` prefix and a trailing employee-number digit, both stripped."
- "Discount math: line discount amount = `fact_sales[line_total] * fact_sales[unit_price_discount]`. Use `[Total Discount Amount]` rather than inline math."
- "There is no separate date dimension — use `fact_sales[order_date]` for time aggregations."
- "Data quality lives in the `test` schema, table `test_results`, one row per (run_id, test_name)."

**Starter / example questions:**

- "Who are the top 5 salespersons by total sales?"
- "Which salespersons give the largest discounts (by total discount amount and as % of their sales)?"
- "How many distinct salespersons are there?"
- "What is the average order value by salesperson?"
- "Which products generate the most revenue?"
- "Which product category drives the most discount give-away?"
- "Show me sales trend by month over the last year."
- "What is the latest data quality status? Are any tests failing?"

**Guardrails:**

- Refuse questions outside this dataset (no external lookups, no other businesses).
- Do not invent column names that aren't in the semantic model — if a metric isn't exposed, propose adding it and stop.

If the AISkill REST endpoint is not yet GA in the tenant, the reporting notebook will print these full instructions and starter questions so they can be pasted when creating the Data Agent manually in the Fabric portal.

## Known constraints (from prior runs)
- In gold joins where two inputs share the same column name (especially `address_id` in `customer_address` and `address`), always alias both sides and reference columns as `alias.column`; never use bare column names after the join.
- Gold output tables must not carry duplicate logical columns with different aliases; project one canonical `address_id` column in `dim_customer`.