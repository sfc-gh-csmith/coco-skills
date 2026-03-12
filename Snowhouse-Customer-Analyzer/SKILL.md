---
name: snowhouse-customer-analyzer
description: "Analyze a Snowflake customer's usage patterns from Snowhouse. Use for ALL requests to: analyze customer usage, understand user activity, investigate user queries, profile a customer, find what features a user uses. Triggers: customer usage, user activity, analyze user, customer profile, user investigation, snowhouse user, what does this user do."
---

# Snowhouse Customer Analyzer

## Workflow

### Step 1: Gather Inputs

**Ask** the user for:
1. **Account name** (org format, e.g. `TOYOTAFINANCIALSERVICES_TFSPROD`)
2. **User name** (human name, e.g. `Thomas Seive`)

### Step 2: Resolve Account

**Goal:** Find the account's deployment and ID.

Run this query to search across known deployments:

```sql
SELECT
  DPO:"AccountDPO:primary"."name"::STRING AS ACCOUNT_NAME,
  DPO:"AccountDPO:primary"."id"::NUMBER AS ACCOUNT_ID,
  DPO:"AccountDPO:primary"."internalName"::STRING AS INTERNAL_NAME,
  DPO:"AccountDPO:primary"."state"::NUMBER AS STATE,
  DPO:"AccountDPO:primary"."type"::STRING AS ACCOUNT_TYPE,
  '<DEPLOYMENT>' AS DEPLOYMENT
FROM SNOWHOUSE_CORE.<DEPLOYMENT>.ACCOUNT_ETL
WHERE DPO:"AccountDPO:primary"."name"::STRING = '<ACCOUNT_NAME>'
LIMIT 1
```

**Deployment search order:** Try these deployments until found: `VA2`, `VA`, `VA3`, `VA4`, `PROD1`, `PROD2`, `PROD3`, `AZEASTUS2PROD`, `AWSUSEAST1HASSIUM`.

Alternatively, check `SNOWHOUSE_IMPORT_SHARE_DB.<DEPLOYMENT>.CUSTOMER_USAGE_DAILY` for the account name.

Record: `ACCOUNT_ID`, `DEPLOYMENT`, `INTERNAL_NAME`.

**If account not found after all deployments:** Stop and ask the user to verify the account name.

### Step 3: Find the User

```sql
SELECT NAME, LOGIN_NAME, EMAIL, FIRST_NAME, LAST_NAME, DISPLAY_NAME,
       DEFAULT_WAREHOUSE, DEFAULT_ROLE, CREATED_ON, LAST_SUC_LOGIN, DELETED_ON
FROM SNOWHOUSE_IMPORT_SHARE_DB.<DEPLOYMENT>.USER_ETL_V
WHERE ACCOUNT_ID = <ACCOUNT_ID>
AND (LOWER(NAME) LIKE '%<SEARCH>%'
     OR LOWER(EMAIL) LIKE '%<SEARCH>%'
     OR LOWER(LAST_NAME) LIKE '%<SEARCH>%'
     OR LOWER(LOGIN_NAME) LIKE '%<SEARCH>%'
     OR LOWER(DISPLAY_NAME) LIKE '%<SEARCH>%')
LIMIT 20
```

Use the user's last name as `<SEARCH>`. If multiple matches, present them and ask the user to pick.

**Present User Profile:**
- Username, login, email, display name
- Default role and warehouse
- Created date, last successful login, deleted status

### Step 4: 6-Month Activity Summary

Run these queries in parallel:

**4a. Monthly query activity:**
```sql
SELECT DATE_TRUNC('MONTH', CREATED_ON) AS MONTH,
       COUNT(*) AS QUERY_COUNT,
       ROUND(SUM(TOTAL_DURATION)/1000/60, 1) AS TOTAL_DURATION_MINS,
       ROUND(SUM(CLOUD_SERVICES_CREDITS_USED), 4) AS CLOUD_CREDITS
FROM SNOWHOUSE_IMPORT_SHARE_DB.<DEPLOYMENT>.JOB_RAW_V
WHERE ACCOUNT_ID = <ACCOUNT_ID>
AND USER_NAME = '<USERNAME>'
AND CREATED_ON >= DATEADD('month', -6, CURRENT_TIMESTAMP())
GROUP BY 1 ORDER BY 1 DESC
```

**4b. Top warehouses:**
```sql
SELECT WAREHOUSE_NAME, COUNT(*) AS QUERY_COUNT,
       ROUND(SUM(TOTAL_DURATION)/1000/60, 1) AS TOTAL_DURATION_MINS,
       ROUND(SUM(CLOUD_SERVICES_CREDITS_USED), 4) AS CLOUD_CREDITS
FROM SNOWHOUSE_IMPORT_SHARE_DB.<DEPLOYMENT>.JOB_RAW_V
WHERE ACCOUNT_ID = <ACCOUNT_ID>
AND USER_NAME = '<USERNAME>'
AND CREATED_ON >= DATEADD('month', -6, CURRENT_TIMESTAMP())
GROUP BY 1 ORDER BY 2 DESC LIMIT 10
```

**4c. Top databases/schemas:**
```sql
SELECT DATABASE_NAME, SCHEMA_NAME, COUNT(*) AS QUERY_COUNT
FROM SNOWHOUSE_IMPORT_SHARE_DB.<DEPLOYMENT>.JOB_RAW_V
WHERE ACCOUNT_ID = <ACCOUNT_ID>
AND USER_NAME = '<USERNAME>'
AND CREATED_ON >= DATEADD('month', -6, CURRENT_TIMESTAMP())
GROUP BY 1, 2 ORDER BY 3 DESC LIMIT 20
```

**4d. Top query patterns (truncated descriptions):**
```sql
SELECT LEFT(DESCRIPTION, 80) AS QUERY_PREFIX, COUNT(*) AS QUERY_COUNT,
       ROUND(SUM(TOTAL_DURATION)/1000/60, 1) AS DURATION_MINS
FROM SNOWHOUSE_IMPORT_SHARE_DB.<DEPLOYMENT>.JOB_RAW_V
WHERE ACCOUNT_ID = <ACCOUNT_ID>
AND USER_NAME = '<USERNAME>'
AND CREATED_ON >= DATEADD('month', -6, CURRENT_TIMESTAMP())
GROUP BY 1 ORDER BY 2 DESC LIMIT 15
```

**Present:** Monthly table, warehouse breakdown, database/schema breakdown, and characterize whether the user is primarily interactive vs. automated based on query patterns (look for `write_pandas`, `SELECT 1`, `SYSTEM$` polling, `COPY INTO` patterns for automation indicators).

### Step 5: Current Month Deep-Dive

**5a. Daily breakdown this month:**
```sql
SELECT DATE_TRUNC('DAY', CREATED_ON) AS DAY,
       COUNT(*) AS QUERY_COUNT,
       ROUND(SUM(TOTAL_DURATION)/1000/60, 1) AS DURATION_MINS
FROM SNOWHOUSE_IMPORT_SHARE_DB.<DEPLOYMENT>.JOB_RAW_V
WHERE ACCOUNT_ID = <ACCOUNT_ID>
AND USER_NAME = '<USERNAME>'
AND CREATED_ON >= DATE_TRUNC('MONTH', CURRENT_DATE())
GROUP BY 1 ORDER BY 1
```

**5b. Today's activity (if any):**
```sql
SELECT DESCRIPTION, COUNT(*) AS QUERY_COUNT,
       ROUND(SUM(TOTAL_DURATION)/1000/60, 2) AS TOTAL_DURATION_MINS,
       WAREHOUSE_NAME, DATABASE_NAME, SCHEMA_NAME
FROM SNOWHOUSE_IMPORT_SHARE_DB.<DEPLOYMENT>.JOB_RAW_V
WHERE ACCOUNT_ID = <ACCOUNT_ID>
AND USER_NAME = '<USERNAME>'
AND CREATED_ON >= CURRENT_DATE()
GROUP BY 1, 4, 5, 6 ORDER BY 2 DESC LIMIT 20
```

**5c. Hourly pattern on a peak day** (pick the highest-volume day from 5a):
```sql
SELECT DATE_TRUNC('HOUR', CREATED_ON) AS HOUR,
       COUNT(*) AS QUERY_COUNT,
       ROUND(SUM(TOTAL_DURATION)/1000/60, 1) AS DURATION_MINS
FROM SNOWHOUSE_IMPORT_SHARE_DB.<DEPLOYMENT>.JOB_RAW_V
WHERE ACCOUNT_ID = <ACCOUNT_ID>
AND USER_NAME = '<USERNAME>'
AND CREATED_ON >= '<PEAK_DAY>' AND CREATED_ON < DATEADD('day', 1, '<PEAK_DAY>')
GROUP BY 1 ORDER BY 1
```

**Present:** Daily table, today's activity summary, and characterize their working hours (overnight automation vs. business hours interactive).

### Step 6: Full Data Footprint (All-Time)

```sql
SELECT DATABASE_NAME, SCHEMA_NAME,
       COUNT(*) AS QUERY_COUNT,
       COUNT(DISTINCT DATE_TRUNC('DAY', CREATED_ON)) AS DISTINCT_DAYS,
       MIN(CREATED_ON) AS FIRST_SEEN,
       MAX(CREATED_ON) AS LAST_SEEN
FROM SNOWHOUSE_IMPORT_SHARE_DB.<DEPLOYMENT>.JOB_RAW_V
WHERE ACCOUNT_ID = <ACCOUNT_ID>
AND USER_NAME = '<USERNAME>'
AND DATABASE_NAME IS NOT NULL AND DATABASE_NAME != ''
GROUP BY 1, 2 ORDER BY 3 DESC
```

**Present in tiers:**
- **Core / Daily Driver**: High query count + many distinct days + recently active
- **Regular / Moderate Use**: Moderate queries or days
- **Exploratory / Light Use**: Low query count, few days, possibly old

Summarize what data domains the user is most familiar with.

### Step 7: Feature Usage Analysis

Run these queries in parallel:

**7a. SQL command category breakdown:**
```sql
SELECT
  CASE
    WHEN UPPER(LEFT(DESCRIPTION, 30)) LIKE '%CREATE%STREAM%' OR UPPER(LEFT(DESCRIPTION, 30)) LIKE '%SHOW%STREAM%' THEN 'STREAMS'
    WHEN UPPER(LEFT(DESCRIPTION, 30)) LIKE '%CREATE%TASK%' OR UPPER(LEFT(DESCRIPTION, 30)) LIKE '%SHOW%TASK%' OR UPPER(LEFT(DESCRIPTION, 30)) LIKE '%ALTER%TASK%' THEN 'TASKS'
    WHEN UPPER(LEFT(DESCRIPTION, 30)) LIKE '%CREATE%PROCEDURE%' OR UPPER(LEFT(DESCRIPTION, 30)) LIKE '%CALL%' THEN 'STORED_PROCS'
    WHEN UPPER(LEFT(DESCRIPTION, 30)) LIKE '%CREATE%FUNCTION%' THEN 'UDFS'
    WHEN UPPER(LEFT(DESCRIPTION, 30)) LIKE '%MERGE%' THEN 'MERGE'
    WHEN UPPER(LEFT(DESCRIPTION, 30)) LIKE '%COPY INTO%' THEN 'COPY_INTO'
    WHEN UPPER(LEFT(DESCRIPTION, 30)) LIKE '%PUT %' THEN 'PUT_FILES'
    WHEN UPPER(LEFT(DESCRIPTION, 30)) LIKE '%GET %' THEN 'GET_FILES'
    WHEN UPPER(LEFT(DESCRIPTION, 30)) LIKE '%CREATE%VIEW%' THEN 'VIEWS'
    WHEN UPPER(LEFT(DESCRIPTION, 30)) LIKE '%CREATE%TABLE%' THEN 'CREATE_TABLE'
    WHEN UPPER(LEFT(DESCRIPTION, 30)) LIKE '%SELECT%' THEN 'SELECT'
    WHEN UPPER(LEFT(DESCRIPTION, 30)) LIKE '%INSERT%' THEN 'INSERT'
    WHEN UPPER(LEFT(DESCRIPTION, 30)) LIKE '%UPDATE%' THEN 'UPDATE'
    WHEN UPPER(LEFT(DESCRIPTION, 30)) LIKE '%DELETE%' THEN 'DELETE'
    WHEN UPPER(LEFT(DESCRIPTION, 30)) LIKE '%DROP%' THEN 'DROP'
    WHEN UPPER(LEFT(DESCRIPTION, 30)) LIKE '%TRUNCATE%' THEN 'TRUNCATE'
    WHEN UPPER(LEFT(DESCRIPTION, 30)) LIKE '%ALTER SESSION%' THEN 'ALTER_SESSION'
    WHEN UPPER(LEFT(DESCRIPTION, 30)) LIKE '%USE %' THEN 'USE_CONTEXT'
    WHEN UPPER(LEFT(DESCRIPTION, 30)) LIKE '%SHOW%' OR UPPER(LEFT(DESCRIPTION, 30)) LIKE '%DESCRIBE%' THEN 'METADATA'
    ELSE 'OTHER'
  END AS FEATURE_CATEGORY,
  COUNT(*) AS QUERY_COUNT
FROM SNOWHOUSE_IMPORT_SHARE_DB.<DEPLOYMENT>.JOB_RAW_V
WHERE ACCOUNT_ID = <ACCOUNT_ID> AND USER_NAME = '<USERNAME>'
GROUP BY 1 ORDER BY 2 DESC
```

**7b. Advanced feature detection (pattern matching on DESCRIPTION):**
```sql
SELECT LEFT(DESCRIPTION, 150) AS QUERY_PREFIX, COUNT(*) AS CNT
FROM SNOWHOUSE_IMPORT_SHARE_DB.<DEPLOYMENT>.JOB_RAW_V
WHERE ACCOUNT_ID = <ACCOUNT_ID> AND USER_NAME = '<USERNAME>'
AND (
  UPPER(DESCRIPTION) LIKE '%SNOWFLAKE.CORTEX%'
  OR UPPER(DESCRIPTION) LIKE '%SNOWFLAKE.ML%'
  OR UPPER(DESCRIPTION) LIKE '%DYNAMIC TABLE%'
  OR UPPER(DESCRIPTION) LIKE '%SNOWPARK%'
  OR UPPER(DESCRIPTION) LIKE '%NOTEBOOK%'
  OR UPPER(DESCRIPTION) LIKE '%STREAMLIT%'
  OR UPPER(DESCRIPTION) LIKE '%CORTEX%SEARCH%'
  OR UPPER(DESCRIPTION) LIKE '%COMPLETE(%'
  OR UPPER(DESCRIPTION) LIKE '%SENTIMENT(%'
  OR UPPER(DESCRIPTION) LIKE '%SUMMARIZE(%'
  OR UPPER(DESCRIPTION) LIKE '%TRANSLATE(%'
  OR UPPER(DESCRIPTION) LIKE '%EXTRACT_ANSWER%'
  OR UPPER(DESCRIPTION) LIKE '%CLASSIFY_TEXT%'
  OR UPPER(DESCRIPTION) LIKE '%FORECAST%'
  OR UPPER(DESCRIPTION) LIKE '%ANOMALY%'
  OR UPPER(DESCRIPTION) LIKE '%CREATE%PIPE%'
  OR UPPER(DESCRIPTION) LIKE '%SNOWPIPE%'
  OR UPPER(DESCRIPTION) LIKE '%MATERIALIZED VIEW%'
  OR UPPER(DESCRIPTION) LIKE '%CLUSTER%BY%'
  OR UPPER(DESCRIPTION) LIKE '%DATA%SHARING%'
  OR UPPER(DESCRIPTION) LIKE '%EXTERNAL%TABLE%'
  OR UPPER(DESCRIPTION) LIKE '%ICEBERG%'
)
GROUP BY 1 ORDER BY 2 DESC LIMIT 20
```

**7c. Client/driver and tool detection:**
```sql
SELECT LEFT(DESCRIPTION, 150) AS QUERY_PREFIX, COUNT(*) AS CNT
FROM SNOWHOUSE_IMPORT_SHARE_DB.<DEPLOYMENT>.JOB_RAW_V
WHERE ACCOUNT_ID = <ACCOUNT_ID> AND USER_NAME = '<USERNAME>'
AND (
  UPPER(DESCRIPTION) LIKE '%TABLEAU%'
  OR UPPER(DESCRIPTION) LIKE '%QUERY_TAG%'
  OR UPPER(DESCRIPTION) LIKE '%PYTHON:%'
  OR UPPER(DESCRIPTION) LIKE '%WORKSPACE%'
  OR UPPER(DESCRIPTION) LIKE '%WORKSHEET%'
)
GROUP BY 1 ORDER BY 2 DESC LIMIT 15
```

**7d. Browse-only feature detection (SHOW commands for features they don't CREATE):**
```sql
SELECT LEFT(DESCRIPTION, 120) AS QUERY_PREFIX, COUNT(*) AS CNT
FROM SNOWHOUSE_IMPORT_SHARE_DB.<DEPLOYMENT>.JOB_RAW_V
WHERE ACCOUNT_ID = <ACCOUNT_ID> AND USER_NAME = '<USERNAME>'
AND (UPPER(LEFT(DESCRIPTION, 30)) LIKE '%STREAM%' OR UPPER(LEFT(DESCRIPTION, 30)) LIKE '%TASK%')
GROUP BY 1 ORDER BY 2 DESC LIMIT 15
```

**Categorize features into three tiers:**

| Tier | Criteria |
|------|----------|
| **Heavily Used** | High query count, core to their workflow |
| **Browsed Only** | SHOW/DESCRIBE commands but no CREATE/ALTER/actual usage |
| **Not Used At All** | Zero matches in any query pattern |

**Feature checklist to evaluate:**

| Feature | Detection Pattern |
|---------|-------------------|
| Python Connector | `Python:snowflake.connector` in DESCRIPTION |
| Tableau | `tableau` in QUERY_TAG or Custom SQL Query pattern |
| Snowsight Worksheets | `workspace` / `worksheet_data` in PUT/GET |
| MERGE (CDC) | MERGE command category |
| COPY INTO | COPY INTO command category |
| Stored Procedures | CALL (non-SYSTEM$) |
| UDFs | CREATE FUNCTION |
| Streams | CREATE STREAM (not just SHOW) |
| Tasks | CREATE/ALTER TASK (not just SHOW) |
| Dynamic Tables | CREATE DYNAMIC TABLE (not just SHOW) |
| Cortex AI (LLM) | SNOWFLAKE.CORTEX, COMPLETE(, SENTIMENT(, SUMMARIZE(, etc. |
| Cortex Search | CORTEX SEARCH (not just SHOW) |
| Snowpark | SNOWPARK in description |
| ML / Forecasting | SNOWFLAKE.ML, FORECAST, ANOMALY |
| Notebooks | SHOW_NOTEBOOKS (browse) vs actual notebook execution |
| Streamlit | SHOW_STREAMLITS (browse) vs actual Streamlit |
| Snowpipe | CREATE PIPE |
| Materialized Views | MATERIALIZED VIEW |
| Clustering | CLUSTER BY |
| Data Sharing | DATA SHARING |
| External Tables | EXTERNAL TABLE |
| Iceberg Tables | ICEBERG |

### Step 8: Recommendations

Based on the feature analysis, recommend features that would:
1. **Replace inefficient patterns** (e.g., Dynamic Tables replacing CREATE→TRUNCATE→MERGE loops)
2. **Automate manual work** (e.g., Tasks replacing external schedulers)
3. **Unlock new value from existing data** (e.g., Cortex AI on text-heavy data they already query)
4. **Modernize their workflow** (e.g., Notebooks replacing worksheet SQL files)
5. **Increase compute efficiency** (e.g., Snowpark replacing write_pandas)

Tailor recommendations to their actual data domains and query patterns. Be specific about which tables/data would benefit.

## Stopping Points

- After Step 3: If multiple user matches found, ask user to pick
- After Step 4: Present 6-month summary before continuing (user may want to discuss)

## Output

A comprehensive user analysis report with:
1. User profile card
2. 6-month activity table with monthly trends
3. Current month daily breakdown + today's activity
4. Full data footprint organized by familiarity tier
5. Feature usage matrix (Used / Browsed / Not Used)
6. Tailored recommendations for incremental value + consumption

## Troubleshooting

**Account not found:** Try alternate deployments. The account name must match exactly (org format). Ask user to verify.

**User not found:** Try searching by first name, last name, email domain, or login name. Users may have non-obvious usernames.

**JOB_RAW_V query slow:** These queries can be expensive on large accounts. If timeouts occur, reduce the date range from 6 months to 3 months.

**Column errors:** The validated columns for JOB_RAW_V are: `DESCRIPTION`, `DATABASE_NAME`, `SCHEMA_NAME`, `WAREHOUSE_NAME`, `ROLE_NAME`, `CREATED_ON`, `END_TIME`, `TOTAL_DURATION`, `CLOUD_SERVICES_CREDITS_USED`, `USER_NAME`, `ACCOUNT_ID`, `CHILD_JOB_TYPE_NAME`. Do NOT use `START_TIME`, `QUERY_TYPE`, `USER_EMAIL` — these do not exist.

**Deployment mapping reference:**
- AWS US-WEST-2: VA, VA2, VA3, VA4
- AWS US-EAST-1: PROD1, PROD2, PROD3, AWSUSEAST1HASSIUM
- Azure East US 2: AZEASTUS2PROD
