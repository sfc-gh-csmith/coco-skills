# Snowhouse Customer Analyzer

A Cortex Code skill that analyzes individual Snowflake user activity from Snowhouse internal telemetry. Given an account name and a user's name, it produces a comprehensive usage profile.

## What It Produces

1. **User Profile** — Username, email, default role/warehouse, account creation date, last login
2. **6-Month Activity Summary** — Monthly query counts, compute duration, cloud credits, top warehouses, top databases/schemas, and top query patterns
3. **Current Month Deep-Dive** — Daily query breakdown, today's activity, hourly patterns on peak days, automated vs. interactive characterization
4. **Full Data Footprint** — Every database.schema the user has ever queried, ranked by volume and recency, tiered into Core / Regular / Exploratory
5. **Feature Usage Matrix** — Which Snowflake features are heavily used, browsed only (SHOW but no CREATE), or completely untouched
6. **Tailored Recommendations** — Features that would replace inefficient patterns, unlock new value from existing data, and drive incremental consumption

## How to Use

### Register the Skill

Add the skill to your Cortex Code configuration so it triggers automatically. The skill activates on phrases like:
- "analyze customer usage"
- "investigate user activity"
- "what does this user do"
- "customer profile"
- "snowhouse user"

### Invoke It

Ask Cortex Code something like:

```
Analyze the usage of John Smith on account ACME_PROD
```

The skill will prompt you for:
1. **Account name** — The org-format account name (e.g., `TOYOTAFINANCIALSERVICES_TFSPROD`)
2. **User name** — The person's human name (e.g., `Thomas Seive`)

### What Happens

The skill resolves the account across Snowhouse deployments, fuzzy-matches the user, then runs a series of queries against `JOB_RAW_V` and `USER_ETL_V` to build the full profile. Results are presented incrementally across each phase.

## Requirements

- Access to `SNOWHOUSE_CORE` and `SNOWHOUSE_IMPORT_SHARE_DB` databases
- Queries run against the deployment matching the customer's account (VA2, PROD1, etc.)

## Snowhouse Tables Used

| Table | Purpose |
|-------|---------|
| `SNOWHOUSE_CORE.<DEPLOYMENT>.ACCOUNT_ETL` | Resolve account name to ID and deployment |
| `SNOWHOUSE_IMPORT_SHARE_DB.<DEPLOYMENT>.USER_ETL_V` | Look up user by name/email/login |
| `SNOWHOUSE_IMPORT_SHARE_DB.<DEPLOYMENT>.JOB_RAW_V` | All query history and usage analysis |

## Example

See the Thomas Seive investigation on `TOYOTAFINANCIALSERVICES_TFSPROD` for a full example of the output this skill produces — including the discovery that 95% of his 80K monthly queries were automated Python ETL pipelines, his primary domain is Consumer lending, and the top recommendations were Dynamic Tables and Cortex AI.
