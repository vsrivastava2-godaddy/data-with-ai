# Hands-On Workshop: AI for Business Data Users

A guided session to help you use Claude for everyday data tasks — no coding experience required.

---

## What You'll Walk Away With

By the end of this session, you will have:

1. A working **SQL Generator skill** — give it your database schema, ask a business question, get a query
2. A working **Data Interpreter skill** — paste query results, get a plain-English summary
3. Confidence to ask Claude data questions on your own

---

## Prerequisites

Before we start, make sure you have:

- [ ] Claude Code installed and working (run `claude` in your terminal to verify)
- [ ] Connected to **VPN**
- [ ] Access to a MSSQL database (or a schema you want to work with)

> Need setup help? See the [Claude Setup Guide](ai-for-working-with-data-1-pager.md#claude-setup--windows-quick-start)

---

## Exercise 1: Create a SQL Generator Skill

**What it does:** You give it your database schema once, then ask business questions in plain English. It writes the SQL for you.

### Step 1: Create the skill file

1. Open your terminal and navigate to your project folder.
2. Create a folder for your skills:
   ```powershell
   mkdir .claude/skills
   ```
3. Create a new file called `.claude/skills/sql-generator.md` and paste the following:

```markdown
---
name: sql-generator
description: Generates MSSQL queries from business questions using the provided schema
---

# SQL Generator

You are a SQL query generator for Microsoft SQL Server (MSSQL).

## Your Role

- You help business users get data by writing SQL queries
- You translate plain-English questions into correct MSSQL syntax
- You explain what each query does in simple terms

## Rules

1. Always use the schema provided below — never guess table or column names
2. Write clean, readable SQL with proper formatting
3. Add a brief plain-English explanation of what the query returns
4. Include a `TOP 100` unless the user says otherwise (to prevent accidental large pulls)
5. If the question is ambiguous, ask a clarifying question before writing SQL
6. Always include relevant WHERE clauses for date ranges when the question implies recent data

## Schema

[PASTE YOUR SCHEMA HERE — see examples below]

<!-- Example format:

### Table: orders
| Column | Type | Description |
|--------|------|-------------|
| order_id | INT | Primary key |
| customer_id | INT | FK to customers |
| order_date | DATETIME | When the order was placed |
| total_amount | DECIMAL | Order total in USD |
| status | VARCHAR(50) | pending, completed, cancelled, refunded |

### Table: customers
| Column | Type | Description |
|--------|------|-------------|
| customer_id | INT | Primary key |
| name | VARCHAR(100) | Customer name |
| region | VARCHAR(50) | US, EMEA, APAC |
| signup_date | DATETIME | Account creation date |

-->
```

### Step 2: Add your real schema

Replace the `[PASTE YOUR SCHEMA HERE]` section with your actual database tables. Use this format for each table:

```markdown
### Table: your_table_name
| Column | Type | Description |
|--------|------|-------------|
| column_name | DATA_TYPE | What this column means |
```

> **Tip:** If you don't know your schema, ask your DBA or run this in MSSQL:
> ```sql
> SELECT TABLE_NAME, COLUMN_NAME, DATA_TYPE
> FROM INFORMATION_SCHEMA.COLUMNS
> WHERE TABLE_SCHEMA = 'dbo'
> ORDER BY TABLE_NAME, ORDINAL_POSITION;
> ```

### Step 3: Try it out

Launch Claude and use your skill:

```
claude

> /sql-generator
```

Then ask questions like:

| Business Question | What It Does |
|-------------------|--------------|
| "How many orders did we get last week?" | Counts by date range |
| "Show me top 10 customers by revenue this quarter" | Aggregation + sorting |
| "Which regions have the most cancellations?" | Group by + filter on status |
| "Compare this month vs last month order counts" | Period-over-period |

### What good output looks like

You ask:
> "How many orders were cancelled last month by region?"

Claude returns:

```sql
SELECT
    c.region,
    COUNT(*) AS cancelled_orders
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.status = 'cancelled'
  AND o.order_date >= DATEADD(MONTH, -1, CAST(GETDATE() AS DATE))
  AND o.order_date < CAST(GETDATE() AS DATE)
GROUP BY c.region
ORDER BY cancelled_orders DESC;
```

**Explanation:** This counts all cancelled orders from last month, broken out by customer region, with the highest counts first.

---

## Exercise 2: Create a Data Interpreter Skill

**What it does:** You paste query results (or any data table), and it gives you a plain-English summary with insights.

### Step 1: Create the skill file

Create a new file called `.claude/skills/data-interpreter.md` and paste the following:

```markdown
---
name: data-interpreter
description: Reads data output and provides plain-English summaries and insights
---

# Data Interpreter

You are a data analyst who explains query results and data to business stakeholders.

## Your Role

- You turn raw data into clear, actionable summaries
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
4. Compare values to each other (e.g., "Region A is 3x higher than Region B")
5. If you spot something that looks wrong (data quality issue), flag it
6. Format numbers for readability (e.g., "$1.2M" not "$1,234,567.89")
```

### Step 2: Try it out

Launch Claude and use your skill:

```
claude

> /data-interpreter
```

Then paste any data. For example:

```
region    | cancelled_orders
----------|------------------
US        | 342
EMEA      | 187
APAC      | 89
LATAM     | 12
```

### What good output looks like

**Summary:**  
The US region accounts for the majority of cancellations (54%), with EMEA second at 30%. LATAM has minimal cancellation volume.

**Key Findings:**
- Total cancellations across all regions: 630
- US cancellations are nearly 2x EMEA and 4x APAC
- LATAM is negligible at just 2% of total

**Anomalies or Concerns:**
- The US number seems high relative to other regions — worth checking if this is proportional to order volume or if there's a higher cancellation *rate*

**Suggested Next Steps:**
- Pull total order counts by region to calculate cancellation *rate* (not just count)
- Check if the US spike is recent or a consistent pattern
- Look at cancellation reasons if that data is available

---

## Exercise 3: Use Both Skills Together (The Full Workflow)

This is where it gets powerful. Combine both skills for a complete data investigation.

### The workflow:

```
Step 1: Ask a business question
         ↓
Step 2: /sql-generator writes the query
         ↓
Step 3: Run the query in MSSQL
         ↓
Step 4: /data-interpreter explains the results
         ↓
Step 5: Ask follow-up questions → repeat
```

### Try it yourself:

1. **Start with a question:**
   > "I want to understand which product categories have declining sales over the past 3 months"

2. **Use `/sql-generator`** — it writes the SQL for you

3. **Run the query** in your database tool (SSMS, Azure Data Studio, etc.)

4. **Copy the results** and use `/data-interpreter`** — it tells you what the data means

5. **Ask a follow-up:**
   > "Now break this down by region to see if the decline is everywhere or just one market"

6. **Repeat** — each round gets you closer to the answer

---

## Bonus: Quick Tips for Talking to Claude About Data

### Be specific about what you want

| Vague (less helpful) | Specific (much better) |
|----------------------|------------------------|
| "Show me sales data" | "Show me weekly sales totals for the last 8 weeks, by product category" |
| "Why is this number high?" | "Why is the US cancellation count 342 when EMEA is only 187? Is it proportional to volume?" |
| "Give me a report" | "Summarize this data for my manager — focus on the quarter-over-quarter trend" |

### Tell Claude who the audience is

- "Explain this for my VP who wants a 2-sentence summary"
- "I need to put this in a slide deck for the ops review"
- "Help me write a Slack message to my team about this finding"

### Ask Claude to check its own work

- "Does this query look right given the schema?"
- "Are there any edge cases this SQL might miss?"
- "What assumptions are baked into this analysis?"

---

## Cheat Sheet: Useful Prompts for Business Data Work

| What You Want | Prompt |
|---------------|--------|
| Generate a query | "Write a MSSQL query to [business question]. Use the schema I provided." |
| Explain results | "Here are my query results. Summarize the key takeaways for a business audience." |
| Spot trends | "Look at this data over time. What trends or patterns do you see?" |
| Compare periods | "Compare last month vs this month. What changed?" |
| Find anomalies | "Is there anything unusual or unexpected in this data?" |
| Simplify for stakeholders | "Rewrite this as 3 bullet points for a non-technical audience." |
| Suggest next steps | "Based on these results, what should I investigate next?" |
| Debug a query | "This query returned 0 rows but I expected data. What might be wrong?" |

---

## Recap

| Skill | What It Does | When to Use It |
|-------|--------------|----------------|
| `/sql-generator` | Turns business questions into SQL | When you need to pull data but don't want to write SQL from scratch |
| `/data-interpreter` | Turns raw data into insights | When you have results and need to understand or share them |

**The goal:** You focus on the *questions* and *decisions*. Claude handles the SQL and analysis.

---

## Need Help?

- Slack: [#claude-code-development](https://godaddy.enterprise.slack.com/archives/C08SL5SGRL1)
- Full setup guide: [AI for Working with Data 1-Pager](ai-for-working-with-data-1-pager.md)
- Claude docs: [claude.ai/docs](https://docs.anthropic.com/en/docs/claude-code/overview)
