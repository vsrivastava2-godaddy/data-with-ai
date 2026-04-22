# AI for Working with Data — 1-Pager

## Goal
By the end of this session, you should be able to:

- Access and set up Claude on your work PC
- Use AI to understand schemas, data sources, and the meaning of data
- Use AI to write, explain, and modify SQL
- Work more effectively with **MSSQL** and **Athena**
- Use AI for faster data discovery, analysis, and troubleshooting

---

# AI User Maturity Levels — Operations & Data Teams

## Level 1 — Beginner (The Asker)

Users ask AI one-off questions and get quick answers. No data or tools are connected.

| Task | Example |
|---|---|
| Terminology lookup | "What does response_code -18 mean in CPL?" |
| Query syntax help | "How do I write a DATEADD for the last 3 hours in SQL Server?" |
| Log interpretation | "What does this stack trace mean?" |
| Status page summaries | "Summarize this incident report for me" |
| Email/Slack drafting | "Write a message to the team about tonight's maintenance window" |
| Basic data formatting | "Convert this JSON payload into a readable table" |

**Productivity gain:** ~10–15% — saves time on lookups and routine writing

---

## Level 2 — Intermediate (The Analyst)

Users provide context (schemas, data samples, business rules) and iterate with AI to build queries, analyze data, and troubleshoot.

| Task | Example |
|---|---|
| SQL query generation | "Write a query to find all failed transactions for shopper 12345 in the last 48 hours" |
| Data analysis | "Here's a CSV export of error codes by hour — what's the trend?" |
| Incident triage | "Here's the error spike data and the deploy timeline — correlate them" |
| Report building | "Summarize this week's gateway failure rates by processor into a table for the ops review" |
| Runbook assistance | "Walk me through the steps to failover gateway 7 based on our runbook" |
| Regex / parsing | "Write a regex to extract transaction IDs from this gateway_raw field" |

**Productivity gain:** ~25–40% — faster query writing, data interpretation, and incident response

---

## Level 3 — Advanced (The Operator)

Users connect AI to live tools (databases, Jira, dashboards, APIs) and use it to drive multi-step operational workflows end-to-end.

| Task | Example |
|---|---|
| Schema-aware querying | `/cpl-query error code spikes in the last 24 hours` — AI knows the table schema, indexes, and generates optimized SQL |
| Incident investigation | "Query CPL for -18 errors, check which gateways are affected, pull the related Jira tickets" |
| Change order generation | AI fetches Jira ticket + PR details and writes the CO description automatically |
| Cross-system correlation | "Compare error rates in CPL with the deploy log in Jenkins — did the last release cause this?" |
| Automated reporting | "Every Monday, pull last week's error rates by source and gateway, format as a summary" |
| Alert enrichment | "This PagerDuty alert fired for gateway 5 — pull the last 15 min of CPL data and tell me if it's real" |

**Productivity gain:** ~50–70% — AI handles the data gathering and correlation; humans make the decisions

---

## Level 4 — Power User (The Automator)

Users build reusable AI-powered workflows, schedule autonomous monitoring, and extend AI capabilities into team-wide operational tooling.

| Task | Example |
|---|---|
| Custom skills for the team | Build a `/cpl-query` skill so anyone can query CPL without knowing the schema |
| Scheduled monitoring agents | "Every 30 minutes, check gateway failure rates and post to #payments-alerts if > 5%" |
| Automated incident response | AI detects anomaly in CPL, creates a Jira ticket, tags the on-call, and drafts an initial RCA |
| Self-service data tools | Build a Slack bot that lets non-technical ops staff query transaction status by order ID |
| Dashboard-to-action pipelines | AI reads Grafana metrics, correlates with CPL data, and recommends whether to page or wait |
| Knowledge capture | AI learns from resolved incidents and suggests similar fixes for new alerts |
| Onboarding acceleration | New team members use AI skills to query systems safely without memorizing schemas or runbooks |

**Productivity gain:** ~2–5x multiplier — operational toil becomes automated; team scales without headcount

---

## The Key Inflection Points

```
Level 1 → 2:  Giving AI your data and context, not just asking generic questions
Level 2 → 3:  Connecting AI to live systems (databases, Jira, monitoring) instead of copy-pasting
Level 3 → 4:  Building reusable, schedulable workflows that the whole team benefits from
```

> **The biggest ROI jump for ops teams is Level 2 → 3.** That's when AI stops being
> "help me write this query" and becomes "investigate this incident for me."
> The human still decides — but AI does the data gathering in seconds, not minutes.


## Setup

Please make sure you:

- Are connected to **VPN**
- Have access to **Claude**
- Have access to your data tools as needed:
  - **MSSQL**
  - **Athena**

### Claude Setup
Use this guide to set up Claude on your PC:  
[Claude setup link](https://tdl.gdcorp.tools/docs/products/ai/agentic-coding/claude/how-to-set-up-claude-code/)

---

## Connecting Integrations

### Atlassian Integration — Confluence + Jira (MCP)

Connects Claude directly to Confluence and Jira.

```bash
claude mcp add Atlassian --transport http https://mcp.atlassian.com/v1/mcp
```

Authenticate when prompted (uses your GoDaddy Atlassian account).

**What you can do:**
- Search and read Confluence pages
- Read acceptance criteria and requirements from Jira
- Auto-generate change order descriptions from tickets
- Add comments on Jira issues

---

### GitHub Integration (MCP)

```bash
claude mcp add github
```

You'll be prompted for a GitHub Personal Access Token.

**To create token:**
1. Go to https://github.com/settings/tokens
2. Generate new token (classic)
3. Scopes needed: `repo`, `read:org`, `read:user`
4. Copy token and paste when prompted

---

## What We’ll Cover

1. Accessing Claude
2. Prompting basics for data work
3. Finding schemas in GitHub
4. Using AI to make sense of data
5. Working with MSSQL
6. Working with Athena
7. Best practices and common pitfalls
8. Q&A

---

## 1) What AI Can Help With

Claude can help you:

- Translate business questions into SQL
- Explain existing queries in plain English
- Modify SQL queries
- Compare **MSSQL** and **Athena** syntax
- Understand table schemas and join paths
- Make sense of unfamiliar fields and datasets
- Summarize trends, spikes, drops, and outliers
- Help interpret anomalies or unexpected results
- Debug SQL errors
- Suggest ways to refine or optimize your analysis

---

## 2) Prompting Basics

A good prompt usually includes:

- **Business question**
- **SQL dialect** (`MSSQL` or `Athena`)
- **Relevant tables / schema**
- **Time range**
- **Filters**
- **Desired output**

### Prompt Template

    I’m working in [MSSQL / Athena].

    I need to answer this question:
    [business question]

    Relevant schema/tables:
    [table names, columns, or DDL]

    Constraints:
    - [time range]
    - [filters]
    - [output grain]

    Please:
    1. suggest an approach,
    2. write the SQL,
    3. explain the logic,
    4. call out any assumptions.

---

## 3) Finding Schemas in GitHub

In our workflow, schema information may live in a **GitHub repo**.

### What to look for
Common places to check:

- `schemas/`
- `sql/`
- `ddl/`
- `models/`
- `README.md`
- migration files
- view definitions
- table definitions

### How to use Claude with schemas
Paste in:

- `CREATE TABLE` statements
- column lists
- view definitions
- README documentation

Then ask:

- “What does this table likely contain?”
- “Which columns are likely keys?”
- “What tables might this join to?”
- “Which fields are useful for reporting?”

---

## 4) Using AI to Make Sense of Data

AI is useful not just for writing SQL, but also for helping interpret what the data means.

### Ways Claude can help

- Explain what a dataset is showing in plain English
- Summarize trends, spikes, drops, and outliers
- Suggest follow-up questions to investigate
- Help identify whether results look reasonable
- Highlight possible data quality issues
- Explain what a metric likely represents
- Compare segments, time periods, or categories
- Turn raw query output into a short business summary

### Example use cases

You can paste:

- query results
- metric summaries
- sample rows
- aggregated tables
- counts by day/week/status
- before vs after comparisons

Then ask Claude to help interpret them.

### Example prompts

    Here are daily chargeback counts for the last 30 days by processor.

    [PASTE RESULTS]

    Please summarize:
    1. the main trends,
    2. any unusual spikes or drops,
    3. what follow-up questions I should ask.

    Here is a table of fraud review volumes by queue and market.

    [PASTE RESULTS]

    Please explain the main takeaways in plain English for an operations audience.

    I expected these counts to be similar, but one is much higher.

    [PASTE RESULTS]

    What are possible reasons for the difference, including join issues, duplicates, filters, or business logic differences?

### What AI is especially good at here

- Turning raw data into a readable narrative
- Helping non-technical teams understand outputs
- Suggesting next-step analysis
- Identifying where results may need validation

### Important reminder

AI can help interpret results, but you still need to validate:

- business definitions
- join logic
- filters
- time ranges
- duplicate inflation
- whether the metric actually answers the business question

---

## 5) Working with MSSQL

Use Claude to help with:

- SELECTs, joins, filters
- CTEs
- aggregations
- date logic
- window functions
- debugging SQL Server syntax

### Example prompt

    I’m working in MSSQL.

    I need a query to count disputes by day for the last 30 days, broken out by dispute_reason and payment_method.

    The dispute table has:
    - dispute_id
    - created_at
    - dispute_reason
    - transaction_id

    The transaction table has:
    - transaction_id
    - payment_method
    - status

    Only include successful transactions.

    Please write the SQL and explain it.

---

## 6) Working with Athena

Use Claude to help with:

- Athena / Presto SQL syntax
- date functions
- partition-aware filtering
- query troubleshooting
- converting MSSQL queries to Athena

### Example prompt

    I’m working in Athena.

    I need weekly counts of fraud cases for the last 8 weeks by market and decision.

    The table has:
    - case_id
    - created_at
    - market
    - decision

    Please write Athena-compatible SQL and explain the logic.

---

## 7) MSSQL vs Athena: Key Differences

| Task | MSSQL | Athena |
|---|---|---|
| Current date | `GETDATE()` | `current_date` / `current_timestamp` |
| Add days | `DATEADD(DAY, -7, GETDATE())` | `date_add('day', -7, current_date)` |
| Limit rows | `TOP 100` | `LIMIT 100` |
| Date truncation | varies | `date_trunc()` |

### Useful prompt

    Convert this MSSQL query to Athena syntax and explain every change.

---

## 8) Best Practices

### Do
- Be specific
- Include schema/context
- Specify **MSSQL** or **Athena**
- Ask Claude to explain assumptions
- Use AI to summarize and interpret query results
- Validate output before using it

### Don’t
- Paste sensitive data unless approved
- Assume the first answer is correct
- Ignore business definitions
- Forget to validate joins and filters
- Treat AI interpretation as final without checking the underlying logic

---

## 9) Common Pitfalls

- Vague prompts
- Missing SQL dialect
- Incorrect joins
- Wrong business logic
- Expensive or slow queries
- Using syntactically correct SQL that answers the wrong question
- Accepting a good-sounding interpretation without validating the data

---

## 10) Good Reusable Prompts

### Understand a schema

    Here is a table definition from our schema repo.

    [PASTE DDL OR COLUMN LIST]

    Please explain:
    1. what this table likely represents,
    2. which columns are likely keys,
    3. which columns are useful for reporting,
    4. what tables it might join to.

### Write a query

    I’m working in [MSSQL / Athena].

    Business question:
    [QUESTION]

    Relevant schema:
    [PASTE TABLES/COLUMNS]

    Constraints:
    - [TIME RANGE]
    - [FILTERS]
    - [OUTPUT GRAIN]

    Please:
    1. propose an approach,
    2. write the SQL,
    3. explain the logic,
    4. list assumptions.

### Interpret query results

    Here are the query results for [BUSINESS QUESTION].

    [PASTE RESULTS]

    Please:
    1. summarize the main trends,
    2. call out anything unusual,
    3. suggest possible explanations,
    4. recommend next questions to investigate.

### Debug a query

    I’m working in [MSSQL / Athena].

    Here is my query:
    [PASTE QUERY]

    Here is the error:
    [PASTE ERROR]

    Please explain the issue, fix it, and describe what changed.

---

## 11) Key Takeaway

AI can help you move faster with data by making it easier to:

- understand schemas,
- draft and modify SQL,
- translate business questions into analysis,
- interpret results,
- and troubleshoot issues.

The most important habit: **give context and validate the output**.
