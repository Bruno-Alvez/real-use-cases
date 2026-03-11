# Results

## DRE automation

Monthly DRE generation went from a manual spreadsheet consolidation process, typically taking hours across multiple people, to an automated query that runs in under 30 seconds. The output covers both monthly and annual views, is consistent across every run for the same input data, and is available on demand rather than at month-end only.

| Dimension | Before | After |
|---|---|---|
| DRE generation time | Hours (manual spreadsheet consolidation) | Under 30 seconds |
| Consistency | Variable (manual errors, version drift) | Deterministic for same input data |
| Availability | Month-end only | On demand |
| Annual statement | Manual, rarely produced | Automated, same engine as monthly |

## Recurring rules reliability

The deduplication model was one of the higher-stakes design choices in the system. Recurring rules that process multiple times, whether due to a retry, a reprocess after a bug fix, or a manual re-run, must not create duplicate transactions. The composite transaction key, combining date, amount, description, category, and account, makes each generated transaction uniquely identifiable. The check runs against an in-memory cache first for same-request deduplication, then falls back to a database lookup for cross-request detection.

| Dimension | Outcome |
|---|---|
| Duplicate transactions in production | 0 |
| Deduplication strategy | Composite key: date + amount + description + category_id + account_id |
| Deduplication layers | In-memory cache (same request) + database fallback (cross-request) |
| Rule versioning: "update future only" | Creates new version; original preserved via original_rule_id |
| Rule versioning: "update all" | In-place update; no version created |
| Historical accuracy after rule change | Preserved: past transactions link to the rule version active at the time |

## Multi-tenant isolation

Tenant isolation is enforced at the PostgreSQL layer via Row Level Security, not exclusively in application code. This distinction matters: an RLS policy active on a table will block a query that forgets a `WHERE company_id = ?` clause. The application layer also filters by company_id in every repository query, but the database is the enforcement boundary. Neither layer alone is treated as sufficient.

| Dimension | Outcome |
|---|---|
| Tables with RLS enabled | 100% (all tenant tables) |
| Enforcement boundary | PostgreSQL RLS, independent of application code |
| Application-layer filtering | company_id filter in every repository query (belt-and-suspenders) |
| Cross-tenant data leaks observed | 0 |
| RLS roles | service_role (write operations), authenticated (read), anon (public endpoints) |
| Indexes | Composite (company_id, date) on all tenant tables |

## Forecast accuracy (Pro plan)

The Pro plan forecasting engine applies three layers of computation to historical transaction data. EWMA smoothing reduces the influence of outlier months without discarding them. Trend detection runs linear regression over the smoothed values to identify whether the underlying trajectory is increasing or decreasing. Seasonality adjustment applies a factor derived from historical month-over-month patterns, so a month that historically sees elevated revenue relative to its neighbors will be projected accordingly.

Division-by-zero conditions in seasonality and trend calculations are handled explicitly: when there is insufficient historical data to compute a meaningful factor, the algorithm falls back gracefully rather than returning a corrupted value. Forecast results are stored in the database with their algorithm metadata, including the EWMA parameters, trend coefficient, and seasonality factors used, making any forecast reproducible and auditable.

| Dimension | Outcome |
|---|---|
| Algorithm | EWMA + linear trend + seasonality (Pro plan) |
| Basic plan algorithm | Historical average |
| Edge case handling | Explicit fallback for insufficient data and division-by-zero conditions |
| Result storage | Stored with full algorithm metadata (ewma_revenue, trend_revenue, seasonality_factor) |
| Auditability | Any forecast can be reproduced from stored parameters |

## Operational profile

Each CSV import is tracked as a record in the database: filename, total rows, success count, failure count, and processing status. A file that contains 10 valid rows and 3 invalid rows will process all 10 valid rows and report exactly 3 failures with per-row error messages. The invalid rows do not block the valid ones, and the import record gives operators a complete view of what was accepted and what was rejected.

| Dimension | Outcome |
|---|---|
| CSV import model | Partial success: valid rows always process, invalid rows reported individually |
| Import tracking | Per-import record: filename, total rows, success count, failure count, status |
| Health check | `/health` endpoint tests database connection; returns "healthy" or "degraded" |
| Docker health check | Interval 30s, timeout 10s, retries 3, start period 5s |
| Plan limit enforcement timing | Before any data operation, at service layer entry |
| Recurring rule processing counters | Processed, skipped, error counts returned per run |
