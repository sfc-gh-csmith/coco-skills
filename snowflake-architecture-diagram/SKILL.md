---
name: snowflake-architecture-diagram
description: "Generate Snowflake account architecture diagrams from Snowhouse metadata. Use for ALL requests to: create architecture diagram, build architecture overview, QBR architecture slide, account architecture, visualize snowflake account. Triggers: architecture diagram, account architecture, QBR diagram, snowflake architecture, architecture overview."
---

# Snowflake Account Architecture Diagram

Generates a left-to-right SVG architecture diagram for any Snowflake account using Snowhouse metadata. Produces an HTML file showing: Data Sources → EL/ETL → Bronze/Silver/Gold Medallion → Consumers/AI Services.

## Workflow

### Step 1: Gather Account Info

**Ask** user for:
- **Account locator** (e.g., RJ49656, DS88921) or account name
- **Deployment** (e.g., awsuseast2, va2, azeastus2prod)
- **Output directory** (default: current working directory)
- **Customer name** (optional — if not provided, infer from data patterns)

### Step 2: Resolve Account in Snowhouse

**Goal:** Find the account ID and internal name.

1. **Search standard deployments first** (VA, VA2, VA3, PROD1, PROD2, PROD3, AZEASTUS2PROD, AWSUSEAST1HASSIUM), then user-specified deployment.

2. **SQL pattern for account resolution:**
```sql
SELECT
  DPO:"AccountDPO:primary"."name"::STRING AS ACCOUNT_NAME,
  DPO:"AccountDPO:primary"."id"::NUMBER AS ACCOUNT_ID,
  DPO:"AccountDPO:primary"."internalName"::STRING AS INTERNAL_NAME
FROM SNOWHOUSE_CORE.<DEPLOYMENT>.ACCOUNT_ETL
WHERE DPO:"AccountDPO:primary"."name"::STRING = '<LOCATOR>'
   OR DPO:"AccountDPO:primary"."internalName"::STRING = '<LOCATOR>'
   OR DPO:"AccountDPO:primary"."id"::STRING = '<NUMERIC_ID>'
LIMIT 5
```

3. **If not found:** Discover deployment schemas:
```sql
SHOW SCHEMAS LIKE '%<DEPLOYMENT_PATTERN>%' IN DATABASE SNOWHOUSE_CORE
```
Also check `SNOWHOUSE_IMPORT_SHARE_DB` schemas match.

4. **Key lesson:** Account locators may be stored as the `name` field, not `internalName`. Always search all three fields.

**⚠️ STOP if account not found.** Ask user to verify locator/deployment.

### Step 3: Gather Architecture Data

Run these queries against `SNOWHOUSE_IMPORT_SHARE_DB.<DEPLOYMENT>.JOB_RAW_V` with `ACCOUNT_ID = <ID>` and `CREATED_ON >= DATEADD('month', -6, CURRENT_TIMESTAMP())`. Use `timeout_seconds: 300` for all queries.

**Load** `references/snowhouse-queries.md` for the complete SQL queries.

Collect:
1. **Total queries + date range** — overall scale
2. **Users** — who uses the account (USER_NAME, query count, compute hours)
3. **Databases** — all databases by query count
4. **Database-schema breakdown** — for medallion tier classification
5. **Warehouses** — by query count and compute hours
6. **Roles** — role distribution
7. **Features** — AI/advanced feature detection (Cortex, ML, Dynamic Tables, Notebooks, Streamlit, etc.)
8. **Connectors/clients** — Tableau, Hevo, JDBC, ODBC, Power BI, etc. from DESCRIPTION patterns
9. **Top query patterns** — LEFT(DESCRIPTION, 120) grouped, for understanding data flow
10. **CHILD_JOB_TYPE_NAME** — stored proc usage, streaming, etc.

### Step 4: Classify Architecture

From the gathered data, determine:

**Medallion Tiers:**
- **Bronze/Raw**: Databases with names containing RAW, LANDING, STAGE, INGEST, or where COPY INTO / PUT / Hevo / ingestion patterns dominate
- **Silver/Curated**: Databases with INT, CURATED, STAGING, TRANSFORM, or where stored procs / merge patterns dominate
- **Gold/Analytics**: Databases with MART, ANALYTICS, REPORTING, DW, CONSUMPTION, WSP, EXPLORE, or where SELECT-heavy BI patterns dominate
- **System/Governance**: SNOWFLAKE db, Trust Center, security schemas — shown in governance section
- **Data Sharing**: databases with SHARE in name — shown as separate items

**Left Side (Sources & Ingestion):**
- External source systems (from DESCRIPTION patterns and user names)
- EL/ETL tools (Hevo, ADF, Talend, COPY INTO, stored procs, tasks/streams)
- Connectors (JDBC, ODBC, Python, specific tools like Tableau, ServiceNow)
- Data shares inbound

**Right Side (Consumption & Output):**
- AI Services (Cortex, ML Functions, Dynamic Tables, Notebooks, Streamlit, Snowpark)
- Consumers (Tableau, Power BI, Snowsight, ODBC/JDBC clients)
- Data sharing outbound
- Data export

**⚠️ MANDATORY CHECKPOINT:** Present the classified architecture to the user for review before building the diagram. Show: customer name, medallion tier assignments, left-side sources, right-side consumers, and any notable patterns.

### Step 5: Build HTML Diagram

**Load** `assets/template.html` — this is the base SVG template.

**Adapt the template** with account-specific data:
1. Update header: customer name, account locator, deployment, date range, total query count
2. **Left side**: Populate source systems, EL/ETL tools, connectors, data sharing based on Step 4
3. **Center panel**: Set medallion tier counts, database lists under each tier, key tables, warehouses, governance, users
4. **Right side**: Populate AI services, consumers, data sharing out, data export
5. **Connections**: Curved Bezier paths from left→bronze, horizontal dashed lines from gold→right with count labels
6. Update footer with account ID

**Layout constants (keep consistent):**
- SVG width: 1800, height: 820-940 (scale to content)
- Center panel: CL=595, CR=1200, fill=#f0f8ff
- Medallion boxes: m1=615, m2=820, m3=1025, width=155, height=65
- Box dimensions: BW=158, BH=28, BS=36 (vertical spacing)
- Left column: starts at x=240 for tools, x=28 for source boxes
- Right column: starts at x=1260

**Color scheme:**
- Bronze: #faf3e6 bg, #8d6e3b border, #6d4c11 text
- Silver: #f5f5f5 bg, #90a4ae border, #546e7a text
- Gold: #fff9e6 bg, #f9a825 border, #f57f17 text
- ETL tools: #fff8e1 bg, #ff9800 border
- Connectors: #e3f2fd bg, #2196f3 border
- AI Services: #f3e5f5 bg, #7b1fa2 border
- Data Sharing: #e8f5e9 bg, #4caf50 border
- Governance: #e0f2f1 bg, #26a69a border
- Source systems: #fff bg, #e53935 border

**Write** file as `simplified_<account_short>_architecture.html` in the output directory.

### Step 6: Verify & Present

1. If a local HTTP server is available, open the diagram in the browser and screenshot it
2. Otherwise, tell the user the file path and suggest opening in a browser
3. Present key architecture insights:
   - Account scale (total queries, user count)
   - Dominant patterns (e.g., "Tableau-heavy", "ETL-dominated", "AI-exploratory")
   - Notable features or gaps

## Stopping Points

- ✋ Step 2: If account not found in Snowhouse
- ✋ Step 4: After architecture classification — present for user review before building diagram
- ✋ Step 6: After diagram built — present for feedback

## Troubleshooting

**Account not found:**
- Try searching by name, internalName, AND numeric ID
- Use `SHOW SCHEMAS LIKE '%<pattern>%' IN DATABASE SNOWHOUSE_CORE` to discover deployment schemas
- Non-standard deployments exist (AWSUSEAST2, AWSEUWEST2, etc.) beyond the common VA/PROD ones

**Queries timeout:**
- Use `timeout_seconds: 300` for JOB_RAW_V queries
- For very large accounts (>100M queries), add `LIMIT` or narrow the date range

**JOB_RAW_V column reference (validated):**
Good: DESCRIPTION, DATABASE_NAME, SCHEMA_NAME, WAREHOUSE_NAME, ROLE_NAME, CREATED_ON, END_TIME, TOTAL_DURATION, CLOUD_SERVICES_CREDITS_USED, USER_NAME, ACCOUNT_ID, CHILD_JOB_TYPE_NAME
**Do NOT use:** START_TIME, QUERY_TYPE, USER_EMAIL — these columns do not exist.

## Output

An HTML file with a self-contained SVG architecture diagram. No external dependencies. Opens in any browser.
