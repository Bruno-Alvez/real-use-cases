# Technical Decisions

## Recurring rule versioning via new records over in-place updates

The "update future only" use case requires that past transactions remain associated with the rule definition that was active when they were created. An in-place update destroys that link: after the update, there is no way to reconstruct what the rule said before.

The decision was to treat each rule change with future scope as a record creation event, not a mutation event. The current rule is marked as superseded and a new record is created with `original_rule_id` pointing back to the chain origin and `effective_from_date` marking when the new version takes over. Each version stores its full rule definition, not a diff, so any version is independently readable.

The "update all" path uses in-place mutation because there is no historical divergence to preserve: the user is saying the old rule was wrong and all transactions should reflect the correction.

**Alternatives considered:** soft delete with recreation (adds deletion semantics that don't apply here), event sourcing (more expressive but significantly more complex for this use case), single version table with audit log (separates the history from the active record in a way that makes common queries harder).

**Trade-offs accepted:** Multiple version records per rule means queries for the active rule must filter on `superseded_by IS NULL`. Joining transactions back to their rule version requires walking the `original_rule_id` chain. These are acceptable costs for a system where historical accuracy is a core correctness requirement.

---

## RLS and company_id double enforcement

The most dangerous class of bug in a multi-tenant system is a data leak: one tenant's data returned to another tenant's request. Application code can have this bug, whether through a missing `WHERE` clause, a cached query result, or an unexpected code path. The question is what happens when it does.

The decision was to treat RLS and application-layer `company_id` filtering as two independent enforcement layers, neither of which is assumed to be sufficient on its own. Every tenant table has an RLS policy that evaluates `company_id` against the session context. Every repository method includes an explicit `company_id` filter. If the application layer fails, the database layer catches it. If the RLS policy is misconfigured for a specific query pattern, the application layer catches it.

This is belt-and-suspenders engineering. It adds a small amount of redundancy to every query, and that redundancy is intentional.

The PostgreSQL client runs with the `service_role` key, which bypasses RLS. This is necessary for server-side operations that span tenants or that need elevated privileges. The mitigation is that all queries go through the repository layer, where `company_id` filtering is consistently applied. The RLS policies serve as the backstop for any case where a query escapes the repository pattern.

**Alternatives considered:** trust application layer only (eliminates one layer of protection with no upside), trust RLS only (makes the application code fragile to any future change in the client credential model), include company_id in JWT claims (creates a synchronization problem when company membership changes).

**Trade-offs accepted:** The `company_users` lookup on every authenticated request adds one database round-trip per API call. This cost is accepted in exchange for membership state that is always current, without requiring token refresh when a user's company access changes.

---

## Synchronous processing for CSV import and recurring rules

Heavy operations, CSV parsing and transaction insertion, recurring rule evaluation across all active rules, are processed synchronously within the HTTP request. This is a deliberate choice for the current operational profile, not a deferred task.

The service layer is structured as a strict boundary between the router and the repository: routers call service methods, service methods contain business logic, repositories handle data access. This boundary means the processing logic has no dependency on the HTTP request/response cycle. Migrating a service method to be invoked by a background queue worker instead of an HTTP handler requires changing the invocation site, not the method itself.

The synchronous approach provides immediate feedback to the caller: a CSV import returns a complete result, including per-row success and failure, within the response. An async model would require a polling or webhook mechanism to deliver that result, adding complexity that is not justified by the current data volumes.

**Alternatives considered:** FastAPI BackgroundTasks (defers work but loses response-level feedback and provides no retry mechanism), Celery or RQ (adds infrastructure dependency and operational overhead not warranted at current scale), async task table with status polling (correct for high volume but premature for current workload).

**Trade-offs accepted:** Long-running operations block the HTTP connection. A CSV file with thousands of rows will hold a connection open for its entire processing duration. This is the known constraint, and the service layer boundary ensures it can be addressed by introducing a queue without changing the business logic.

---

## Plan limits at the service layer over middleware

Feature gating could live in middleware that intercepts requests before they reach the handler. The appeal is centralization: one place defines what is and isn't allowed per plan. The problem is that some limits are stateful, requiring a database query to evaluate (for example, "has this company reached its monthly import count?"), and middleware should not be making data access calls.

The decision was to centralize limit logic in `PlanLimitsService`, which exposes a method per enforced limit, and to call those methods explicitly at the start of each relevant service method. The check happens before any data operation. If the limit is exceeded, no data is written and a structured error is returned.

Explicit calls are visible in the code path. There is no indirection to trace when debugging why a request was rejected. The enforcement is testable in isolation by unit testing `PlanLimitsService` directly.

**Alternatives considered:** middleware (can't access business context cleanly), database triggers (enforce after setup, can't provide useful error messages), decorator pattern (elegant but adds indirection that obscures where the check happens for each feature).

**Trade-offs accepted:** Each new feature that has a plan limit must explicitly call the relevant check method. There is no mechanism that automatically enforces this. The discipline is maintained through code review rather than automated enforcement.

---

## In-memory deduplication cache with database fallback

During recurring rule processing, the same rule might generate the same transaction twice within a single run if date ranges overlap. Checking the database for every potential duplicate adds N queries to a batch operation where N is the number of transactions being considered.

The solution uses a two-level check. An in-memory set accumulates transaction keys for the current processing run. Before inserting a transaction, the key is checked against the set first. If not found, the database is queried. If neither check finds a match, the transaction is inserted and its key is added to the set.

The in-memory cache is request-scoped and not shared across concurrent runs. This is an accepted constraint: in the current deployment model, the recurring rule processor runs as a single triggered call. If the processing model changes to support concurrent processing of different rule sets, the in-memory cache provides no cross-process deduplication and the database check becomes the sole mechanism.

**Alternatives considered:** database-only check (correct but adds one query per transaction in a batch context), unique constraint on the transactions table (would also solve the problem at the database level but requires a schema change and changes the error model from a pre-check to an exception handler), Redis distributed cache (adds infrastructure dependency not justified at current scale).

**Trade-offs accepted:** The in-memory cache does not protect against concurrent runs of the same processing job. The current deployment model makes this a non-issue, and the database fallback handles the cross-request case.

---

## Forecast results stored in the database over on-demand calculation

Forecast generation involves EWMA smoothing over historical transactions, trend detection via linear regression, and seasonality factor computation. Running this on every dashboard request would mean re-reading all historical transactions and re-running the full computation for each page load.

The decision was to store forecast results at generation time, including the intermediate values: `ewma_revenue`, `trend_revenue`, and `seasonality_factor`, alongside the final projected values. The dashboard reads stored results. Recalculation is triggered explicitly, either by the user requesting a fresh forecast or by a scheduled operation.

Storing the intermediate values has an additional benefit: any stored forecast is auditable. Given the stored parameters, the output can be reconstructed independently, and changes in algorithm behavior across versions can be detected by comparing stored parameters against what the current algorithm would produce.

**Alternatives considered:** on-demand calculation (correct output always, but expensive and adds latency to dashboard load), in-memory cache with TTL (avoids database writes but is lost on restart and doesn't support auditability), calculate and store only final values (cheaper storage but loses auditability).

**Trade-offs accepted:** Stored forecasts become stale when new transactions are added. The system does not automatically invalidate and recalculate forecasts on transaction changes. Users see the forecast as of the last generation time, which is displayed alongside the forecast values.

---

## Composite indexes on (company_id, date) for all tenant tables

Every meaningful query against a tenant table includes a `company_id` filter and a date range or date ordering. A single-column index on either field handles only half the query. An index on `(company_id, date)` covers the full predicate: the database can use the company_id prefix to narrow to the tenant's data, then use the date ordering within that partition to find the relevant range.

This index structure becomes more important as individual tenant data volumes grow. A company with three years of transaction history will have a large partition of rows sharing their `company_id`. Without the composite index, a date range query within that company scans the full partition rather than seeking to the date range directly.

**Alternatives considered:** single-column index on company_id (doesn't cover date range), single-column index on date (scans all tenants for a date range), no index beyond primary key (linear scan at scale).

**Trade-offs accepted:** Write operations pay the cost of maintaining the composite index. For this workload, reads vastly outnumber writes, so the index maintenance cost is acceptable. Storage overhead is proportional to the number of tenant table rows, which is expected and planned for.
