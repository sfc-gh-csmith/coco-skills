# Snowflake Architecture Diagram Generator

A Cortex Code skill that generates self-contained SVG architecture diagrams for any Snowflake account using Snowhouse metadata.

## What It Does

Given an account locator and deployment, this skill queries Snowhouse to understand how a Snowflake account is being used and produces a single HTML file with a left-to-right architecture flow diagram:

```
Data Sources  -->  EL/ETL  -->  Bronze / Silver / Gold  -->  AI Services / Consumers
```

The diagram includes:
- **Left side**: Source systems, EL/ETL tools (Hevo, ADF, Talend, etc.), connectors (JDBC, ODBC, Tableau), and inbound data shares
- **Center**: Medallion architecture with Bronze (raw), Silver (curated), and Gold (analytics) tiers, database lists, key data domains, warehouse compute, governance controls, and user breakdown
- **Right side**: AI services (Cortex, ML, Dynamic Tables, Notebooks, Streamlit), consumers (Tableau, Power BI, Snowsight), and outbound data sharing

The output is a single HTML file with inline SVG -- no external dependencies, opens in any browser.

## Example Output

Diagrams have been generated for accounts ranging from ~500K queries (small dealership) to 374M queries (large enterprise):

| Account | Scale | Deployment |
|---------|-------|------------|
| Toyota Financial Services | 374M queries | VA2 |
| Toyota Material Handling NA | 44.5M queries | Azure East US 2 |
| Subaru New England | 500K queries | AWS US East 2 |

## How to Use

Invoke the skill by saying any of:
- "Create an architecture diagram"
- "Build a QBR architecture slide"
- "Visualize this Snowflake account"
- "Account architecture overview"

The skill will:
1. Ask for the **account locator** and **deployment**
2. Resolve the account in Snowhouse
3. Run 11 queries against `JOB_RAW_V` to gather 6 months of usage data
4. Classify databases into medallion tiers and identify sources/consumers
5. **Pause for your review** of the classification before building
6. Generate the HTML diagram and present it

## File Structure

```
snowflake-architecture-diagram/
├── SKILL.md                        # Main skill workflow (157 lines)
├── references/
│   └── snowhouse-queries.md        # 11 SQL query templates for data gathering
├── assets/
│   └── template.html               # Parameterized SVG template with helpers
└── README.md
```

## Data Sources

All data comes from Snowhouse internal metadata:
- **Account resolution**: `SNOWHOUSE_CORE.<DEPLOYMENT>.ACCOUNT_ETL`
- **Query history**: `SNOWHOUSE_IMPORT_SHARE_DB.<DEPLOYMENT>.JOB_RAW_V`

The skill analyzes the last 6 months of query history to determine:
- Which databases exist and how heavily they're used
- What ETL tools and connectors are in play
- Which AI/ML features are adopted
- Who the users are and how they consume data
- Warehouse compute distribution

## Requirements

- Access to Snowhouse (`SNOWHOUSE_CORE` and `SNOWHOUSE_IMPORT_SHARE_DB`)
- The account's deployment name (e.g., VA2, AZEASTUS2PROD, AWSUSEAST2)
- Cortex Code with the skill installed
