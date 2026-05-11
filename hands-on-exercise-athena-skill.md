# Hands-On Exercise: Build an Athena Query Skill (AWS CLI)

Create a Claude Code skill that directly queries AWS Athena via the CLI — no console needed. Ask a question, get results, get analysis, all in the terminal.

---

## What You'll Build

A skill that:
1. Takes a business question or order ID as input
2. Builds an Athena SQL query with proper partition filters
3. Executes it via `aws athena` CLI
4. Waits for results and analyzes them

**End result:** You type `/cpl-query last 100 orders` and Claude fetches and analyzes real CPL data.

---

## Prerequisites

- [ ] Claude Code installed (`claude` command works in terminal)
- [ ] Connected to **VPN**
- [ ] AWS CLI installed (`aws --version` returns a result)
- [ ] Okta AWS credentials configured (see Step 1 below)
- [ ] Access to the `signals_platform_cln` Athena database

---

## Step 1: Configure AWS Credentials

The skill needs AWS credentials to call Athena. At GoDaddy we use Okta-sourced credentials.

### 1a. Verify your Okta setup

Run this in your terminal:

```bash
source ~/.aws/okta-env.sh
aws sts get-caller-identity --region us-west-2
```

You should see output like:
```json
{
    "Account": "613446412771",
    "Arn": "arn:aws:sts::613446412771:assumed-role/GD-AWS-USA-GPD-EcommPayments-Ops/yourname@godaddy.com"
}
```

> **Important:** You need the **Ops** role (not Dev-Private-PowerUser) to access the Athena tables in the shared data catalog.

### 1b. If `okta-env.sh` doesn't exist

Ask your team lead for Okta AWS access, or run:
```bash
# Check if you have okta2aws or gimme-aws-creds installed
which okta2aws gimme-aws-creds
```

Follow your team's standard process to generate `~/.aws/okta-env.sh` with `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, and `AWS_SESSION_TOKEN`.

### 1c. Tell Claude how to get credentials

When building the skill, you'll include a step that sources credentials before every query. The key line is:

```bash
source ~/.aws/okta-env.sh
```

---

## Step 2: Verify Athena Access

Before building the skill, confirm you can query Athena from the CLI. Ask Claude:

```
Run this query on Athena in us-west-2 using the signals_platform_cln database:
SELECT * FROM "signals_platform_cln"."ecomm_payments_cpl_event_cln" LIMIT 5

Source credentials from ~/.aws/okta-env.sh first.
Use s3://athena-workspace-us-west-2/claude-results/ as the output location.
```

Claude should:
1. Source credentials
2. Submit the query via `aws athena start-query-execution`
3. Poll with `aws athena get-query-execution` until it succeeds
4. Fetch results with `aws athena get-query-results`

If this works, you're ready to build the skill.

### Troubleshooting

| Problem | Solution |
|---------|----------|
| "You must specify a region" | Credentials didn't load. Check `~/.aws/okta-env.sh` exists and exports `AWS_REGION` or always pass `--region us-west-2` |
| "TABLE_NOT_FOUND" | Wrong account/role. You need the Ops role, not Dev |
| "Access Denied" on S3 output | Try a different output bucket your role can write to |
| Query returns 0 rows | Data has lag. Try widening the time window or drop time filters |

---

## Step 3: Create the Skill

Create the file `.claude/skills/cpl-query.md` in your project directory:

```bash
mkdir -p .claude/skills
```

Then create the skill file with the following content:

````markdown
---
name: cpl-query
description: Query CPL (Common Purchase Log) data from AWS Athena. Accepts a business question, order_id, or time range. Runs the query via AWS CLI and analyzes results.
---

You query and analyze CPL data from AWS Athena. Based on `$ARGUMENTS`:

- **If an order_id is provided** (e.g., `4081323387`): look up all transactions for that order
- **If a time range is mentioned** (e.g., `last 100 orders`, `last 1 hour`): query that window
- **If a business question** (e.g., `approval rate by processor`): generate appropriate SQL
- **If `anomalies` or no arguments**: fetch last 100 records and analyze for anomalies

## Execution Steps

### 1. Source AWS credentials
```bash
source ~/.aws/okta-env.sh
```

### 2. Build the Athena query

Use the schema and partition strategy below. Always include partition filters.

### 3. Submit and poll

```bash
EXECUTION_ID=$(aws athena start-query-execution \
  --query-string "$QUERY" \
  --query-execution-context Database=signals_platform_cln \
  --result-configuration "OutputLocation=s3://athena-workspace-us-west-2/claude-results/" \
  --region us-west-2 \
  --output text --query 'QueryExecutionId')

# Poll until complete
for i in $(seq 1 30); do
  STATE=$(aws athena get-query-execution \
    --query-execution-id "$EXECUTION_ID" \
    --region us-west-2 \
    --output text --query 'QueryExecution.Status.State')
  if [[ "$STATE" == "SUCCEEDED" || "$STATE" == "FAILED" || "$STATE" == "CANCELLED" ]]; then
    break
  fi
  sleep 2
done
```

### 4. Fetch and analyze results

```bash
aws athena get-query-results \
  --query-execution-id "$EXECUTION_ID" \
  --region us-west-2 \
  --output json
```

Parse the JSON output and present analysis.

## Table Schema

**Database:** `signals_platform_cln`  
**Table:** `ecomm_payments_cpl_event_cln`

### Partition Columns (ALWAYS filter on these)

| Column | Type | Description |
|--------|------|-------------|
| src_receive_utc_year_num | int | Year data was received (UTC) |
| src_receive_utc_month_num | int | Month data was received (UTC) |
| src_receive_utc_day_num | int | Day data was received (UTC) |
| src_receive_utc_hour_num | int | Hour data was received (UTC) |

> **Important:** Use `cln_create_utc_ts` for time-based filtering (when the record was created in the clean layer). The partition columns control which S3 files are scanned, so always include them for cost efficiency.

### Key Columns

| Column | Type | Description |
|--------|------|-------------|
| order_id | string | Unique order identifier |
| customer_id | string | Customer identifier |
| total_amt | int | Transaction amount (cents) |
| response_code | int | Response bucket (1=Approval, 4=External Decline) |
| response_code_description | string | Human-readable response |
| reason_code | int | Decline reason subcategory |
| processor_name | string | Payment processor name |
| payment | string | Method: CREDIT_CARD, E_WALLET, etc. |
| payment_type | string | Subtype: Visa, MasterCard, AMEX, etc. |
| billing_country_code | string | ISO Alpha-2 country |
| gateway_action_type_description | string | Auth, Capture, Auth/Capture, Refund |
| response_time_ms | int | Gateway response time in ms |
| date_entered_utc_ts | timestamp | When transaction was submitted to gateway |
| cln_create_utc_ts | timestamp | When record entered the clean layer |
| gateway_raw | string | Raw gateway response |
| basket_type | string | Basket type (gdshop, etc.) |
| product_purchase_indicator | string | NEW_ONLY, RENEWAL_ONLY, MIXED |
| network_token_present_flag | boolean | Network token used |
| cvv_supplied_flag | boolean | CVV was provided |
| gateway_selection_process | string | How the gateway was chosen (ML model, weight-based) |
| available_gateway_list | string | Gateways available for selection |
| processor_success_ratio_json | string | Success ratios per gateway |

### All Columns (for reference)

producer_event_id, order_id, customer_id, total_amt, response_code,
response_code_description, reason_code, auth_code, trans_id,
address_validation_result, payment_attempt_cnt, date_entered_utc_ts,
request_source, private_label_id, merchant_account_id, http_status,
response_time_ms, processor, processor_name, payment, payment_type,
billing_country_code, gateway_raw, billing_map_id, card_info_modified,
billing_attempt_cnt, gateway_action_type, gateway_action_type_description,
gateway_selection_process, gd_shop_gateway_id, explicit_market_place_shop_id,
basket_type, cvv_validation_result, purchase_pathway, primary_key_id,
transaction_currency_type, transaction_currency_total_amt,
pp_shopper_payment_method_id, cavv_code_flag, card_range_id,
installment_term, publisher_hash, available_gateway_list,
processor_success_ratio_json, trace_id, src_create_utc_ts,
cln_create_utc_ts, cln_update_utc_ts, network_token_present_flag,
network_token_expiration_month, network_token_expiration_year,
network_token_create_utc_ts, instrument_last_successful_use_utc_ts,
instrument_last_successful_cit_use_utc_ts, instrument_last_successful_gateway,
instrument_update_utc_ts, predicted_gateway_model_scores, pm_priority,
card_entry_type, proc_rsp_code, proc_rsp_msg, payment_uri, gateway_detail,
product_purchase_indicator, cvv_supplied_flag

## Query Patterns

### Latest N orders (default)
```sql
SELECT *
FROM "signals_platform_cln"."ecomm_payments_cpl_event_cln"
WHERE src_receive_utc_year_num = <CURRENT_YEAR>
  AND src_receive_utc_month_num = <CURRENT_MONTH>
  AND src_receive_utc_day_num = <CURRENT_DAY>
  AND src_receive_utc_hour_num IN (<PREV_HOUR>, <CURRENT_HOUR>)
ORDER BY cln_create_utc_ts DESC
LIMIT 100
```

### Order lookup
```sql
SELECT *
FROM "signals_platform_cln"."ecomm_payments_cpl_event_cln"
WHERE order_id = '<ORDER_ID>'
  AND src_receive_utc_year_num = <CURRENT_YEAR>
  AND src_receive_utc_month_num = <CURRENT_MONTH>
ORDER BY cln_create_utc_ts DESC
```

### Approval rate by processor
```sql
SELECT
    processor_name,
    COUNT(*) AS total,
    SUM(CASE WHEN response_code = 1 THEN 1 ELSE 0 END) AS approvals,
    ROUND(100.0 * SUM(CASE WHEN response_code = 1 THEN 1 ELSE 0 END) / COUNT(*), 2) AS approval_rate
FROM "signals_platform_cln"."ecomm_payments_cpl_event_cln"
WHERE src_receive_utc_year_num = <CURRENT_YEAR>
  AND src_receive_utc_month_num = <CURRENT_MONTH>
  AND src_receive_utc_day_num = <CURRENT_DAY>
  AND src_receive_utc_hour_num IN (<PREV_HOUR>, <CURRENT_HOUR>)
GROUP BY processor_name
ORDER BY total DESC
```

## Output Format

### For order lookups
Present a timeline of transaction attempts with key fields in a table. Highlight errors.

### For aggregate queries
Present a structured report:

1. **Summary** — Key metrics (approval rate, volume, avg amount)
2. **Breakdown** — By processor, country, payment type as relevant
3. **Decline Analysis** — Top decline reasons with counts
4. **Performance** — Response time stats (median, p95, max)
5. **Anomalies** — Anything unusual (spikes, missing data, unexpected patterns)
6. **Recommendations** — What to investigate next

Format amounts from cents to dollars. Use tables. Bold anything abnormal.
````

---

## Step 4: Test Your Skill

Launch Claude and try these prompts:

```bash
claude
```

### Test 1: Basic query
```
/cpl-query last 100 orders
```

Expected: Claude fetches 100 most recent records and provides an approval rate analysis.

### Test 2: Order lookup
```
/cpl-query 4081323387
```

Expected: Claude finds all transaction attempts for that order and shows the timeline.

### Test 3: Business question
```
/cpl-query approval rate by processor for today
```

Expected: Claude builds an aggregation query, runs it, and presents results.

### Test 4: Anomaly detection
```
/cpl-query anomalies
```

Expected: Claude fetches recent data and flags anything unusual (high decline rates, slow response times, etc.)

---

## Step 5: Customize for Your Team

### Add more tables

If your team uses other Athena tables, add their schemas to the skill under a new section:

```markdown
## Additional Tables

### Table: your_other_table
| Column | Type | Description |
|--------|------|-------------|
| ... | ... | ... |
```

### Change the output bucket

Replace `s3://athena-workspace-us-west-2/claude-results/` with your team's preferred bucket.

### Add team-specific queries

Add common queries your team runs under the "Query Patterns" section. For example:

```markdown
### Network token impact
\```sql
SELECT
    network_token_present_flag,
    COUNT(*) AS total,
    ROUND(100.0 * SUM(CASE WHEN response_code = 1 THEN 1 ELSE 0 END) / COUNT(*), 2) AS approval_rate
FROM "signals_platform_cln"."ecomm_payments_cpl_event_cln"
WHERE src_receive_utc_year_num = <CURRENT_YEAR>
  AND src_receive_utc_month_num = <CURRENT_MONTH>
  AND src_receive_utc_day_num = <CURRENT_DAY>
GROUP BY network_token_present_flag
\```
```

---

## How It Works (Under the Hood)

```
You ask a question
       ↓
Claude reads the skill (schema + patterns)
       ↓
Builds an Athena SQL query
       ↓
Runs: aws athena start-query-execution
       ↓
Polls: aws athena get-query-execution (every 2s)
       ↓
Fetches: aws athena get-query-results
       ↓
Parses JSON → analyzes → presents insights
```

The entire flow happens in your terminal. No console switching, no copy-pasting.

---

## Tips and Gotchas

| Tip | Why |
|-----|-----|
| Always include partition filters | Athena charges per data scanned — no filter = full table scan = $$$  |
| Use `cln_create_utc_ts` for "recent" queries | This is when the record entered the clean layer, not when the transaction happened |
| Data has lag (minutes to hours) | The `date_entered_utc_ts` may be older than the partition date — the pipeline processes in batches |
| Credentials expire | If you get auth errors, re-run `source ~/.aws/okta-env.sh` or re-authenticate via Okta |
| Quote table names | Use `"database"."table"` syntax — Athena needs it for names with underscores in some catalogs |
| LIMIT your queries | Always use LIMIT during exploration to control costs |

---

## Bonus: Combine with the ES Skill

If you also have the [CPL Elasticsearch skill](example-skill-cpl-es.md), you can use both:

| Use Case | Which Skill |
|----------|-------------|
| Real-time (last few minutes) | `/cpl-es` (Elasticsearch) |
| Historical analysis (hours/days) | `/cpl-query` (Athena) |
| Order lookup (need it fast) | `/cpl-es` (Elasticsearch) |
| Aggregations over large time ranges | `/cpl-query` (Athena) |
| Anomaly detection (live) | `/cpl-es` (Elasticsearch) |
| Trend analysis (week over week) | `/cpl-query` (Athena) |

---

## Need Help?

- Slack: [#claude-code-development](https://godaddy.enterprise.slack.com/archives/C08SL5SGRL1)
- AWS Athena docs: [SQL Reference](https://docs.aws.amazon.com/athena/latest/ug/ddl-sql-reference.html)
- Full setup guide: [AI for Working with Data 1-Pager](ai-for-working-with-data-1-pager.md)
- ES skill example: [example-skill-cpl-es.md](example-skill-cpl-es.md)
