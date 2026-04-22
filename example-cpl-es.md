---
name: cpl-es
description: Query CPL (Common Purchase Log) data from Elasticsearch. Accepts an order_id for lookup, or runs anomaly detection on recent transactions.
---

You query and analyze CPL data from Elasticsearch. Based on `$ARGUMENTS`:

- **If an order_id is provided** (e.g., `4070190440`): look up all transactions for that order
- **If a shopper_id is provided** (e.g., `shopper 12345`): look up recent transactions for that shopper
- **If a time period or topic is mentioned** (e.g., `last 1 hour`, `errors today`): query that window and analyze
- **If no arguments or just `anomalies`**: run anomaly detection on the last 1 hour — compare error rates, flag spikes by response code, gateway, source, and country

## Execution

Build the ES query DSL, execute it via curl, then analyze and present results.

```bash
curl -s -X POST "https://ecommplatform-prod-usw2.es.us-west-2.aws.found.io:9243/cpl-legacy-prod-*/_search" \
  -H "Authorization: ApiKey $CPL_ES_API_KEY" \
  -H "Content-Type: application/json" \
  -d '<QUERY_JSON>'
```

If `CPL_ES_API_KEY` is not set, ask the user to run:
```bash
export CPL_ES_API_KEY="<their-api-key>"
```

## Field Reference

All CPL fields are nested under `payload.*`. Text fields have `.keyword` for exact match / aggregations.

### Key fields

| Field | Type | Use |
|---|---|---|
| @timestamp | date | Time filter (use `now-Xh` syntax) |
| payload.order_id.keyword | keyword | Order lookup |
| payload.shopper_id | text (no .keyword) | Use `match` query |
| payload.response_code | long | 1,2,3,665=success; everything else=error |
| payload.reason_code | long | Sub-reason within response code |
| payload.source.keyword | keyword | Calling service |
| payload.processor.keyword | keyword | Payment processor |
| payload.payment_type.keyword | keyword | Card brand (Visa, MC, etc.) |
| payload.payment.keyword | keyword | Payment method (credit_card, etc.) |
| payload.billing_country_code.keyword | keyword | Country code |
| payload.gdshop_gatewayID | long | Gateway ID |
| payload.gatewayActionType | long | 1=Auth, 2=Capture, 3=AuthCapture |
| payload.total | long | Amount in cents USD |
| payload.transactionCurrencyTotal | long | Amount in local currency |
| payload.transactionCurrencyType.keyword | keyword | Currency code |
| payload.response_time | long | Gateway response time (ms) |
| payload.gateway_raw | text | Raw gateway response (full-text searchable) |
| payload.gateway_raw.keyword | keyword | Raw gateway response (exact match) |
| payload.trans_id.keyword | keyword | Processor transaction ID |
| payload.auth_code.keyword | keyword | Auth code |
| payload.avs_code.keyword | keyword | AVS result |
| payload.cvv_code.keyword | keyword | CVV result |
| payload.httpStatus | long | HTTP status from gateway |
| payload.attempts | long | Attempt count |
| payload.billing_attempt | long | Billing attempt number |
| payload.website.keyword | keyword | Originating URL |
| payload.basket_type.keyword | keyword | Basket type |
| payload.pathway.keyword | keyword | Pathway GUID |
| payload.PrivateLabelID | long | Private label ID |
| payload.merchantAccountID | long | Merchant account |
| payload.pp_shopperPaymentMethodID | long | Shopper payment method ID |
| payload.date_entered | date | Original DB timestamp |

## Response Codes

| Code | Meaning |
|---|---|
| 1, 2, 3, 665 | Success (approved) |
| 4 | Gateway/processor error |
| -8 | Timeout |
| -9 | Connection error |
| -10 | Processing error |
| -16 | Validation failure |
| -18 | GATEWAY_NOT_FOUND |
| -22 | Declined by processor |
| -24 | Authentication failure |
| -26 | Configuration error |
| -30 | Rate limited / throttled |
| -35 | Fraud / risk decline |
| Other negative | System/infrastructure errors |
| Other positive | Processor-specific decline codes |

## Query Patterns

### Order lookup
```json
{
  "size": 50,
  "query": {"bool": {"must": [
    {"term": {"payload.order_id.keyword": "ORDER_ID"}}
  ]}},
  "sort": [{"@timestamp": "asc"}],
  "_source": ["@timestamp", "payload.order_id", "payload.total", "payload.response_code",
    "payload.reason_code", "payload.source", "payload.payment_type", "payload.processor",
    "payload.gateway_raw", "payload.gdshop_gatewayID", "payload.gatewayActionType",
    "payload.trans_id", "payload.auth_code", "payload.billing_country_code",
    "payload.response_time", "payload.attempts"]
}
```

### Anomaly detection (default — last 1 hour)

Run these queries in parallel and synthesize:

**Query 1 — Error volume by hour (current vs previous hours)**
```json
{
  "size": 0,
  "query": {"bool": {"must": [
    {"range": {"@timestamp": {"gte": "now-6h"}}},
    {"bool": {"must_not": [{"terms": {"payload.response_code": [1, 2, 3, 665]}}]}}
  ]}},
  "aggs": {
    "over_time": {
      "date_histogram": {"field": "@timestamp", "fixed_interval": "1h"},
      "aggs": {
        "by_code": {"terms": {"field": "payload.response_code", "size": 15}}
      }
    }
  }
}
```

**Query 2 — Error breakdown by source + gateway (last 1 hour)**
```json
{
  "size": 0,
  "query": {"bool": {"must": [
    {"range": {"@timestamp": {"gte": "now-1h"}}},
    {"bool": {"must_not": [{"terms": {"payload.response_code": [1, 2, 3, 665]}}]}}
  ]}},
  "aggs": {
    "by_source": {
      "terms": {"field": "payload.source.keyword", "size": 15},
      "aggs": {
        "by_code": {"terms": {"field": "payload.response_code", "size": 10}},
        "by_gateway": {"terms": {"field": "payload.gdshop_gatewayID", "size": 10}}
      }
    }
  }
}
```

**Query 3 — Total volume (success + error) for context (last 1 hour)**
```json
{
  "size": 0,
  "query": {"range": {"@timestamp": {"gte": "now-1h"}}},
  "aggs": {
    "success_vs_error": {
      "filters": {
        "filters": {
          "success": {"terms": {"payload.response_code": [1, 2, 3, 665]}},
          "error": {"bool": {"must_not": [{"terms": {"payload.response_code": [1, 2, 3, 665]}}]}}
        }
      }
    },
    "by_payment_type": {
      "terms": {"field": "payload.payment_type.keyword", "size": 10},
      "aggs": {
        "errors": {"filter": {"bool": {"must_not": [{"terms": {"payload.response_code": [1, 2, 3, 665]}}]}}}
      }
    },
    "by_country": {
      "terms": {"field": "payload.billing_country_code.keyword", "size": 15},
      "aggs": {
        "errors": {"filter": {"bool": {"must_not": [{"terms": {"payload.response_code": [1, 2, 3, 665]}}]}}}
      }
    },
    "avg_response_time": {"avg": {"field": "payload.response_time"}}
  }
}
```

### Shopper history
```json
{
  "size": 50,
  "query": {"bool": {"must": [
    {"match": {"payload.shopper_id": "SHOPPER_ID"}},
    {"range": {"@timestamp": {"gte": "now-5d"}}}
  ]}},
  "sort": [{"@timestamp": "desc"}],
  "_source": ["@timestamp", "payload.order_id", "payload.total", "payload.response_code",
    "payload.reason_code", "payload.source", "payload.payment_type", "payload.processor",
    "payload.gateway_raw", "payload.gdshop_gatewayID", "payload.response_time"]
}
```

## Output Format

### For order/shopper lookups
Present results as a table with key fields. Highlight any errors. Summarize the transaction timeline.

### For anomaly detection
Present a structured report:

1. **Summary** — Total volume, error rate %, comparison to prior hours
2. **Error Code Trends** — Which codes are increasing/decreasing vs prior hours. Flag any code that jumped >50% or is new
3. **Top Error Sources** — Which services are generating the most errors
4. **Gateway Health** — Any gateway with elevated failure rates
5. **Geographic/Payment Patterns** — Errors concentrated by country or card type
6. **Response Time** — Average and any degradation
7. **Recommendation** — Actionable next steps (drill-down queries, teams to alert)

Format numbers clearly. Use tables. Bold anything that looks abnormal.
