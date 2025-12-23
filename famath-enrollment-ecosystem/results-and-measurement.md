# Results and Measurement

## Outcomes

**Enrollment Automation**: Students can complete enrollment via Portal or WhatsApp without manual intervention. Document validation and status updates happen automatically via database triggers.

**Payment Integration**: Checkout experience via Asaas. Dynamic pricing adapts to enrollment windows. Voucher system enables targeted discounts.

**Operational Efficiency**: Admin backoffice reduces manual database operations. Kanban interface provides visibility into enrollment pipeline. Campaign and voucher management automated.

**Data Consistency**: Database triggers ensure enrollment status matches document approval state. Timeline events provide audit trail. RLS policies enforce access control.

## What is Measured Today

**Application Logs**: HTTP request logs (method, path, status, duration) via Morgan (Portal) and Winston (Admin). Structured logs with sanitization in Agent (Pino). Error logs with stack traces in development mode.

**Database Metrics**: Query performance via Supabase dashboard (if available). Table row counts. Storage usage.

**No Application-Level Metrics**: No request rate tracking, error rate monitoring, latency percentiles, or business metrics (enrollment conversion, checkout success rate).

## What to Measure Next

1. **Enrollment Conversion Rate**: Percentage of started enrollments that complete payment. Query `enrollments` and `payments` tables. Calculate: `COUNT(DISTINCT enrollments.id WHERE status = 'approved') / COUNT(DISTINCT enrollments.id WHERE status != 'cancelled')`.

2. **Checkout Success Rate**: Percentage of checkout session creations that succeed. Instrument `checkoutService.createPixCheckoutSession()`. Count successful vs failed API calls to Asaas.

3. **Asaas API Error Rate**: Percentage of Asaas API calls that fail. Instrument all `fetch()` calls to Asaas API. Track 4xx and 5xx responses.

4. **WhatsApp Message Processing Latency**: Time from webhook reception to user response. Measure job processing time in Agent worker. Log start and end timestamps for each job.

5. **Document Approval Time**: Average time from upload to approval. Query `documents` table with `uploaded_at` and `reviewed_at` timestamps. Calculate: `AVG(reviewed_at - uploaded_at) WHERE status = 'approved'`.

6. **Voucher Usage Rate**: Percentage of checkouts that use a voucher. Query `checkout_sessions` and `voucher_usage` tables. Calculate: `COUNT(DISTINCT checkout_sessions.id WHERE voucher_id IS NOT NULL) / COUNT(DISTINCT checkout_sessions.id)`.

7. **Agent Conversation Completion Rate**: Percentage of conversations that result in enrollment creation. Count logs "Enrollment created" vs "Webhook received". Track in Redis or database.

8. **Legacy Sync Success Rate**: Percentage of enrollment syncs to SQL Server that succeed. Log sync attempts and outcomes. Track in database or logs.

## How to Measure

**Instrumentation**: Add metrics middleware to Express apps (Prometheus client). Export metrics endpoint `/api/metrics`. Wrap external API calls with timing and error tracking. Add database query timing in repository layer. Add correlation IDs to all requests.

**Dashboards**: Grafana dashboard with Prometheus data source. Panels for request rate, error rate, latency percentiles, business metrics. Supabase dashboard for database metrics. Custom SQL queries for business metrics.

**Log Analysis**: Parse structured logs (Pino, Winston) for error patterns. Use log aggregation tool (ELK, Loki) for correlation. Track correlation IDs across components.

**Database Queries**: Use SQL queries for business metrics. Create materialized views for common aggregations. Schedule periodic reports.

## Evidence Pointers

**Enrollment Automation**: Database triggers for automatic status updates (`database/triggers/auto_update_enrollment_status.sql`). Timeline events provide audit trail (see `verification-checklist.md`).

**Payment Integration**: Asaas checkout integration (`backend/src/services/checkoutService.ts`). Voucher system with database constraints (`database/supabase/supabase_schema_vouchers_campaigns_final.sql`).

**Operational Efficiency**: Admin Kanban interface and voucher/campaign management (`deep-dives/admin/architecture.md`). Database triggers for automatic protocol generation.

**Data Consistency**: RLS policies enforce access control (`database/supabase/supabase_schema_collaborators.sql`). Database triggers ensure status consistency (`database/triggers/auto_update_enrollment_status.sql`).

**Reliability Mechanisms**: Agent retry settings (BullMQ 2 attempts, 5s backoff) documented in `reliability-and-ops.md`. Voucher uniqueness constraints prevent duplicate usage (`verification-checklist.md`).

**Evidence**: `backend/src/middleware/errorHandler.ts`, `src/utils/sanitizers.ts`, `database/schema.sql`
