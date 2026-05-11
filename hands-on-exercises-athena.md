# Hands-On Workshop: AI for Business Data Users — Athena Edition

A guided session to help you use Claude for querying and analyzing data in AWS Athena — no coding experience required.

---

## What You'll Walk Away With

By the end of this session, you will have:

1. A working **Athena Query Generator skill** — give it your table schema, ask a business question, get a query
2. A working **Athena Results Interpreter skill** — paste query output, get a plain-English summary
3. Confidence to explore data in Athena using natural language

---

## Prerequisites

Before we start, make sure you have:

- [ ] Claude Code installed and working (run `claude` in your terminal to verify)
- [ ] Connected to **VPN**
- [ ] Access to AWS Athena (via AWS Console or a query tool)
- [ ] Know which database/catalog your tables live in

> Need setup help? See the [Claude Setup Guide](ai-for-working-with-data-1-pager.md#claude-setup--windows-quick-start)

---

## Quick Primer: How Athena Differs from MSSQL

If you've used MSSQL before, here are the key differences:

| Concept | MSSQL | Athena |
|---------|-------|--------|
| Limit rows | `SELECT TOP 100` | `SELECT ... LIMIT 100` |
| Current date | `GETDATE()` | `current_date` or `current_timestamp` |
| Subtract days | `DATEADD(DAY, -7, GETDATE())` | `date_add('day', -7, current_date)` |
| Truncate date | `CAST(order_date AS DATE)` | `date_trunc('day', order_date)` |
| String matching | `LIKE '%text%'` | `LIKE '%text%'` (same) |
| Partition filter | Not applicable | Always filter on partition columns for speed/cost |

> **Important:** Athena charges per data scanned. Always include partition filters (like `year`, `month`, `day`, or `dt`) to avoid scanning entire tables and running up costs.

---

## Exercise 1: Create an Athena Query Generator Skill

**What it does:** You give it your Athena table schema once, then ask business questions in plain English. It writes Athena-compatible SQL for you.

### Step 1: Create the skill file

1. Open your terminal and navigate to your project folder.
2. Create a folder for your skills (if it doesn't exist):
   ```powershell
   mkdir .claude/skills
   ```
3. Create a new file called `.claude/skills/athena-query-generator.md` and paste the following:

```markdown
---
name: athena-query-generator
description: Generates AWS Athena (Presto/Trino) SQL queries from business questions using the provided schema
---

# Athena Query Generator

You are a SQL query generator for AWS Athena (Presto/Trino SQL dialect).

## Your Role

- You help business users get data by writing Athena-compatible queries
- You translate plain-English questions into correct Athena SQL syntax
- You explain what each query does in simple terms

## Rules

1. Always use the schema provided below — never guess table or column names
2. Always use Athena/Presto SQL syntax (NOT MSSQL or MySQL syntax)
3. Always include a `LIMIT 100` unless the user says otherwise
4. Always include partition filters (year/month/day/dt) when the table has them — this saves cost
5. Add a brief plain-English explanation of what the query returns
6. If the question is ambiguous, ask a clarifying question before writing SQL
7. Use date functions correctly:
   - `current_date` for today
   - `date_add('day', -7, current_date)` to go back in time
   - `date_trunc('week', some_date)` to group by period
   - `date_format(some_date, '%Y-%m-%d')` to format dates
8. Use `CAST` or `TRY_CAST` for type conversions
9. Warn the user if a query might scan a lot of data (missing partition filter)

## Common Athena Patterns

- Group by time: `date_trunc('week', event_date)`
- Filter recent: `WHERE dt >= date_format(date_add('day', -30, current_date), '%Y-%m-%d')`
- Count distinct: `COUNT(DISTINCT user_id)`
- Null handling: `COALESCE(column, 'default')`
- Array columns: `CROSS JOIN UNNEST(array_column) AS t(item)`

## Schema

[PASTE YOUR SCHEMA HERE — see examples below]

<!-- Example format:

### Database: analytics_db

### Table: page_views
| Column | Type | Description | Partition? |
|--------|------|-------------|:----------:|
| event_id | STRING | Unique event ID | |
| user_id | STRING | Visitor identifier | |
| page_url | STRING | Page that was viewed | |
| referrer | STRING | Where the user came from | |
| device_type | STRING | desktop, mobile, tablet | |
| country | STRING | Visitor country | |
| event_timestamp | TIMESTAMP | When the view happened | |
| dt | STRING | Date partition (YYYY-MM-DD) | Yes |

### Table: conversions
| Column | Type | Description | Partition? |
|--------|------|-------------|:----------:|
| conversion_id | STRING | Unique conversion ID | |
| user_id | STRING | Who converted | |
| product | STRING | What they bought | |
| revenue | DOUBLE | Revenue in USD | |
| channel | STRING | marketing channel | |
| conversion_date | TIMESTAMP | When the conversion happened | |
| dt | STRING | Date partition (YYYY-MM-DD) | Yes |

-->
```

### Step 2: Add your real schema

Replace the `[PASTE YOUR SCHEMA HERE]` section with your actual Athena tables. Use this format:

```markdown
### Database: your_database_name

### Table: your_table_name
| Column | Type | Description | Partition? |
|--------|------|-------------|:----------:|
| column_name | DATA_TYPE | What this column means | Yes/No |
```

> **Tip:** If you don't know your schema, run this in Athena:
> ```sql
> SHOW COLUMNS IN your_database.your_table;
> ```
> Or browse tables in the AWS Athena console under "Database" in the left sidebar.

### Step 3: Try it out

Launch Claude and use your skill:

```
claude

> /athena-query-generator
```

Then ask questions like:

| Business Question | What It Does |
|-------------------|--------------|
| "How many page views did we get last week by device type?" | Aggregation with partition filter |
| "What are our top 10 converting products this month?" | Ranking with revenue |
| "Show daily unique visitors for the past 30 days" | Time series with COUNT DISTINCT |
| "Which marketing channels drive the most revenue?" | Group by + sum |
| "Compare mobile vs desktop conversion rates" | Calculated metrics across segments |

### What good output looks like

You ask:
> "What are the top 5 marketing channels by revenue for the last 30 days?"

Claude returns:

```sql
SELECT
    channel,
    COUNT(*) AS total_conversions,
    ROUND(SUM(revenue), 2) AS total_revenue,
    ROUND(AVG(revenue), 2) AS avg_revenue_per_conversion
FROM conversions
WHERE dt >= date_format(date_add('day', -30, current_date), '%Y-%m-%d')
GROUP BY channel
ORDER BY total_revenue DESC
LIMIT 5;
```

**Explanation:** This pulls the last 30 days of conversions (using the partition filter `dt` to keep costs low), groups them by marketing channel, and shows total conversions, total revenue, and average revenue per conversion — sorted by highest revenue first.

**Cost note:** This query only scans the last 30 days of data thanks to the `dt` partition filter.

---

## Exercise 2: Create an Athena Results Interpreter Skill

**What it does:** You paste Athena query results (or any data table), and it gives you a plain-English summary with insights tailored for business decisions.

### Step 1: Create the skill file

Create a new file called `.claude/skills/athena-data-interpreter.md` and paste the following:

```markdown
---
name: athena-data-interpreter
description: Reads Athena query output and provides plain-English summaries, insights, and recommended actions
---

# Athena Data Interpreter

You are a data analyst who explains Athena query results to business stakeholders.

## Your Role

- You turn raw Athena output into clear, actionable summaries
- You highlight what matters — trends, anomalies, and key takeaways
- You write for a business audience, not a technical one

## How to Respond

When given data, always provide:

1. **Summary** (2-3 sentences) — What is this data showing at a high level?
2. **Key Findings** (bullet points) — The most important things to notice
3. **Anomalies or Concerns** — Anything unusual that might need attention
4. **Suggested Next Steps** — What questions to ask next or actions to consider

## Rules

1. Never use jargon without explaining it
2. Use actual numbers from the data — don't be vague
3. If the data is too small to draw conclusions, say so
4. Compare values to each other (e.g., "Paid search drives 3x more revenue than social")
5. If you spot something that looks wrong (data quality issue), flag it
6. Format numbers for readability (e.g., "$1.2M" not "$1,234,567.89")
7. When looking at time series data, call out the direction of the trend (growing, declining, flat)
8. Always consider whether a metric should be looked at as absolute value OR as a rate/percentage

## Context to Keep in Mind

- Athena data often comes from event logs — duplicates can happen
- Null values in Athena output might mean "not captured" not "zero"
- Partition columns (dt, year, month) are date-based — check for completeness
- If counts seem surprisingly round (1000, 5000), the data might be sampled
```

### Step 2: Try it out

Launch Claude and use your skill:

```
claude

> /athena-data-interpreter
```

Then paste any Athena output. For example:

```
channel          | total_conversions | total_revenue | avg_revenue_per_conversion
-----------------|-------------------|---------------|---------------------------
paid_search      | 4,521             | 892,340.50    | 197.38
organic          | 3,887             | 445,210.75    | 114.54
email            | 2,103             | 312,880.00    | 148.73
social           | 1,456             | 98,440.25     | 67.61
direct           | 987               | 201,550.80    | 204.21
```

### What good output looks like

**Summary:**  
Paid search is the dominant revenue channel, generating $892K (45% of total) across 4,521 conversions in the last 30 days. Organic drives high volume but at a lower per-conversion value. Direct has the highest average revenue per conversion despite low volume.

**Key Findings:**
- Total revenue across all channels: ~$1.95M from 12,954 conversions
- Paid search leads in both volume and total revenue
- Direct traffic has the highest avg revenue per conversion ($204) — these may be returning/high-intent buyers
- Social has the lowest revenue per conversion ($68) — roughly 1/3 of paid search

**Anomalies or Concerns:**
- Social has decent volume (1,456 conversions) but very low revenue per conversion — could indicate lower-value products or higher discount usage
- Direct traffic's high average ($204) with low volume (987) — worth checking if this includes B2B or enterprise purchases skewing the average

**Suggested Next Steps:**
- Break down social conversions by product to see what's driving the low average
- Compare these numbers to last month — is paid search growing or flat?
- Look at cost per channel to calculate actual ROI (revenue alone doesn't tell the full story)
- Investigate direct traffic — who are these high-value converters?

---

## Exercise 3: Use Both Skills Together (Full Investigation)

Combine both skills for a complete data investigation workflow in Athena.

### The workflow:

```
Step 1: Start with a business question
         ↓
Step 2: /athena-query-generator writes the query
         ↓
Step 3: Run the query in AWS Athena Console
         ↓
Step 4: Copy the results
         ↓
Step 5: /athena-data-interpreter explains what it means
         ↓
Step 6: Ask follow-up questions → repeat
```

### Walk-through example:

**Round 1 — The big picture:**
> "I want to understand how our website traffic has changed over the past 4 weeks"

Use `/athena-query-generator` → get a weekly page view query → run it → paste results into `/athena-data-interpreter`

**Round 2 — Drill down:**
> "Traffic dropped in week 3. Break it down by device type to see if it's mobile or desktop"

Use `/athena-query-generator` again → run → interpret

**Round 3 — Root cause:**
> "Mobile dropped significantly. Show me the top referrers for mobile in week 3 vs week 2"

Use `/athena-query-generator` → run → interpret → now you have an answer

**Result:** In three rounds, you went from "what happened?" to "here's why and what to do about it" — without writing a single query by hand.

---

## Exercise 4: Cost-Aware Querying Practice

Athena charges by data scanned. This exercise teaches you to ask for efficient queries.

### Try these prompts:

**Prompt 1 — Catch missing partitions:**
> "How many events do we have per day for the last year?"
>
> Claude should warn you if the query would scan too much data and suggest using partition filters.

**Prompt 2 — Ask for cost estimate:**
> "How much data will this query scan approximately?"
>
> Claude can estimate based on columns selected and date range.

**Prompt 3 — Request optimization:**
> "I need total revenue by product for all of 2025. What's the most cost-efficient way to query this?"
>
> Claude should suggest partition filters, selecting only needed columns, and using approximations if exact counts aren't needed.

### Cost-saving habits to build:

| Habit | Why It Matters |
|-------|---------------|
| Always filter on partition columns (`dt`, `year`, `month`) | Reduces data scanned from TB to GB |
| Select only columns you need (avoid `SELECT *`) | Less data read = lower cost |
| Use `LIMIT` during exploration | Prevents runaway queries |
| Use `COUNT(*)` before `SELECT *` | Check row counts before pulling full data |
| Ask Claude: "Is this query efficient?" | Catches costly mistakes before you run them |

---

## Bonus: Athena-Specific Prompts That Work Well

### Time-based analysis

| What You Want | Prompt |
|---------------|--------|
| Daily trend | "Show me daily [metric] for the past 30 days" |
| Weekly rollup | "Aggregate [metric] by week for the last 3 months" |
| Month-over-month | "Compare this month to last month for [metric] by [dimension]" |
| Day-of-week pattern | "Is there a day-of-week pattern in [metric]?" |
| Hour-of-day pattern | "What time of day has the highest [metric]?" |

### Funnel and conversion analysis

| What You Want | Prompt |
|---------------|--------|
| Conversion rate | "What percentage of page views turn into conversions, by channel?" |
| Drop-off | "Where in the funnel are we losing the most users?" |
| Time to convert | "How many days between first visit and conversion?" |
| Repeat behavior | "What percentage of converters come back within 30 days?" |

### Comparison and segmentation

| What You Want | Prompt |
|---------------|--------|
| Segment comparison | "Compare [metric] across [segments]. Which stands out?" |
| New vs returning | "Break down revenue by new vs returning users" |
| Top N | "What are the top 10 [items] by [metric]?" |
| Bottom performers | "Which [items] have declined the most vs last month?" |

### Data quality checks

| What You Want | Prompt |
|---------------|--------|
| Null check | "What percentage of rows have null values in [column]?" |
| Duplicate check | "Are there duplicate event_ids in the last 7 days?" |
| Completeness | "Do we have data for every day in the last 30 days, or are there gaps?" |
| Volume sanity | "How does today's row count compare to the average for this table?" |

---

## Cheat Sheet: Athena SQL Patterns

Keep this handy as a quick reference:

```sql
-- Last N days with partition filter
WHERE dt >= date_format(date_add('day', -30, current_date), '%Y-%m-%d')

-- Group by week
GROUP BY date_trunc('week', event_timestamp)

-- Percentage calculation
ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2) AS percentage

-- Running total
SUM(revenue) OVER (ORDER BY dt) AS running_total

-- Rank within groups
ROW_NUMBER() OVER (PARTITION BY channel ORDER BY revenue DESC) AS rank

-- Handle nulls
COALESCE(referrer, 'direct') AS referrer_clean

-- Flatten arrays
CROSS JOIN UNNEST(tags) AS t(tag)

-- Approximate count (faster, cheaper for large datasets)
approx_distinct(user_id) AS unique_users_approx
```

---

## Recap

| Skill | What It Does | When to Use It |
|-------|--------------|----------------|
| `/athena-query-generator` | Turns business questions into Athena SQL | When you need data from Athena but don't want to write SQL from scratch |
| `/athena-data-interpreter` | Turns raw query results into insights | When you have results and need to understand or share them |

**The goal:** You focus on *what you want to know*. Claude handles the Athena syntax, partition filters, and analysis.

---

## Need Help?

- Slack: [#claude-code-development](https://godaddy.enterprise.slack.com/archives/C08SL5SGRL1)
- Full setup guide: [AI for Working with Data 1-Pager](ai-for-working-with-data-1-pager.md)
- MSSQL version: [Hands-On Exercises (MSSQL)](hands-on-exercises.md)
- Athena docs: [AWS Athena SQL Reference](https://docs.aws.amazon.com/athena/latest/ug/ddl-sql-reference.html)
