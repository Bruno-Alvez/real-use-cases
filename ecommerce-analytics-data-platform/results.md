# Results

## Data warehouse quality

The reconstruction eliminated every data quality issue identified in the audit.

| Metric | Before | After |
|---|---|---|
| Quality score | 2/10 | 9/10 |
| Bot attack records (fraud) | 62,466 | 0 |
| Orphan records | 26,921 | 0 |
| Duplicate records | 22,586+ | 0 |
| Total problematic records eliminated | 89,487 | |
| Tables with primary key | 64% (14/22) | 100% (18/18) |
| Tables with RLS | 23% (5/22) | 100% (18/18) |
| Foreign key constraints | 0 | 7 |
| Monetary fields as TEXT | 16 | 0 |
| Records migrated (validated) | | 61,349 |

## Orders coverage

The WooCommerce Product Bundles bug was the most significant data integrity issue in the fact layer. After correction:

| Metric | Before | After |
|---|---|---|
| fato_itens_vendas coverage | 26% of expected items | 100% |
| Missing bundle items | 27,020 | 0 |
| Items corrected via metadata parse | | 31,138 |
| Customers with corrected produto_favorito | | 6 |

The orders coverage metric affects every downstream product: RFV scores, product popularity rankings, revenue calculations, and customer segmentation. All were recalculated after the correction and are now derived from complete data.

## Performance

| Metric | Before | After |
|---|---|---|
| p95 API latency | >30 seconds (timeout) | <2 seconds |
| Endpoints without authentication | 15 | 0 |
| Client-side data volume | 100,000 records per request | 50 (server-side pagination) |
| Largest source file | 3,963 lines | <300 lines per module |
| N+1 query instances | 101 queries per segment page | 1 JOIN |

The p95 latency improvement from >30s to <2s came primarily from three changes: server-side pagination replacing full-dataset client fetching, index coverage on join columns, and eliminating the blocking I/O pattern where `httpx.get()` was called synchronously inside async FastAPI handlers.

## LGPD compliance

| Risk | Before | After |
|---|---|---|
| CPFs accessible without auth | 12,609 | 0 |
| Endpoints with open PII access | 15 | 0 |
| OAuth2 credentials in plaintext DB | Yes (Bling tokens) | No (Secrets Manager) |
| Tables without RLS | 17/22 | 0/18 |

Before the intervention, 12,609 CPFs were accessible via a simple HTTP GET request with no credentials. This was a violation of LGPD Art. 46, which establishes liability for failure to adopt technical security measures. The risk was not hypothetical: anyone who discovered the endpoint URL had full read access to the customer table.

## Infrastructure and operations

| Metric | Before (N8N pipelines) | After (AWS Lambda) |
|---|---|---|
| Pipeline versionability | None | Git (Terraform IaC) |
| Observability | Zero | 28 CloudWatch alarms |
| MTTD (pipeline failure) | >24 hours (manual discovery) | <5 minutes (CloudWatch alarm) |
| MTTR | >24 hours | <5 minutes (alert to resolution) |
| Disaster recovery | Impossible | 15-minute RTO via terraform apply |

## Uptime since production deploy

System has been in production since February 5, 2026. In the period through early March 2026:

- 9 Lambda functions: 100% operational
- 0 CloudWatch alarms in ALARM state (silence = healthy pipelines)
- 18 tables, approximately 94,000 records and growing organically
- 28 alarms monitoring: 0 false positives, 2 true positives (both Bling API outages, both detected in <5 minutes, both resolved without data loss on the next scheduled run)
