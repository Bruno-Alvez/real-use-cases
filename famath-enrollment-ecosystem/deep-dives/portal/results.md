# Results and Measurement

## Outcomes (Qualitative)

**Enrollment Process Automation:**
- Students can complete enrollment without manual intervention (document upload, payment processing)
- Automatic status updates reduce administrative workload
- Protocol number generation ensures unique, sequential enrollment IDs

**Payment Integration:**
- Seamless checkout experience via Asaas (PIX and credit card)
- Dynamic pricing adapts to enrollment windows and campaigns
- Voucher system enables targeted discounts

**Data Consistency:**
- Database triggers ensure enrollment status matches document approval state
- Foreign key constraints prevent orphaned records
- Timeline events provide audit trail for enrollment lifecycle

**Operational Simplicity:**
- Single codebase (monorepo) simplifies deployment
- Supabase eliminates database management overhead
- Render/Vercel hosting reduces infrastructure maintenance

## What is Measured Today

**Application Logs:**
- HTTP request logs (method, path, status, duration, IP) via Morgan middleware
- Error logs with stack traces (development mode)
- Console logs for checkout flow, pricing calculation, voucher application

**Evidence:**
- `backend/src/middleware/logger.ts` - Request logging (lines 21-56)
- `backend/src/middleware/errorHandler.ts` - Error logging (lines 12-31)

**Database Metrics (via Supabase Dashboard):**
- Query performance (if Supabase dashboard provides)
- Table row counts
- Storage usage

**No Application-Level Metrics:**
- No request rate tracking
- No error rate monitoring
- No latency percentiles (p50, p95, p99)
- No business metrics (enrollments per day, checkout conversion rate)

## What to Measure Next

### Reliability Metrics

1. **Checkout Success Rate**
   - Definition: Percentage of checkout session creations that succeed
   - Calculation: `(successful_checkouts / total_checkout_attempts) * 100`
   - Where to instrument: `checkoutService.createPixCheckoutSession()`, `createCartaoCheckoutSession()`
   - Target: > 95%

2. **Asaas API Error Rate**
   - Definition: Percentage of Asaas API calls that fail
   - Calculation: `(failed_asaas_calls / total_asaas_calls) * 100`
   - Where to instrument: All `fetch()` calls to Asaas API
   - Target: < 1%

3. **Checkout Creation Latency (p95)**
   - Definition: 95th percentile time to create checkout session
   - Calculation: Measure time from request to Asaas response
   - Where to instrument: `checkoutService.createPixCheckoutSession()` - start/end timestamps
   - Target: < 2 seconds

4. **Database Query Latency (p95)**
   - Definition: 95th percentile time for database queries
   - Calculation: Measure Supabase query execution time
   - Where to instrument: Repository layer, wrap Supabase calls
   - Target: < 500ms

### Business Metrics

5. **Enrollment Conversion Rate**
   - Definition: Percentage of started enrollments that complete payment
   - Calculation: `(enrollments_with_payment / total_enrollments) * 100`
   - Where to instrument: Query `enrollments` and `payments` tables
   - Query: `SELECT COUNT(DISTINCT e.id) as total, COUNT(DISTINCT p.enrollment_id) as paid FROM enrollments e LEFT JOIN payments p ON e.id = p.enrollment_id AND p.status = 'paid'`

6. **Average Time to Payment**
   - Definition: Average time from enrollment creation to payment completion
   - Calculation: `AVG(payments.paid_at - enrollments.created_at)`
   - Where to instrument: Query `enrollments` and `payments` tables
   - Target: < 7 days

7. **Voucher Usage Rate**
   - Definition: Percentage of checkouts that use a voucher
   - Calculation: `(checkouts_with_voucher / total_checkouts) * 100`
   - Where to instrument: `checkout_sessions` table (add `voucher_id` column if not exists)
   - Query: `SELECT COUNT(*) FILTER (WHERE voucher_id IS NOT NULL) / COUNT(*) * 100 FROM checkout_sessions`

8. **Document Approval Time (Average)**
   - Definition: Average time from document upload to approval
   - Calculation: `AVG(documents.reviewed_at - documents.uploaded_at) WHERE status = 'approved'`
   - Where to instrument: Query `documents` table
   - Target: < 2 business days

### Operational Metrics

9. **API Request Rate (per minute)**
   - Definition: Number of API requests per minute
   - Calculation: Count requests in time window
   - Where to instrument: Request logger middleware
   - Use case: Capacity planning, rate limit tuning

10. **Error Rate by Endpoint**
    - Definition: Percentage of requests that return 4xx/5xx per endpoint
    - Calculation: `(error_responses / total_requests) * 100` per route
    - Where to instrument: Error handler middleware, route identification
    - Target: < 5% per endpoint

11. **Active Checkout Sessions**
    - Definition: Number of checkout sessions in 'ACTIVE' status
    - Calculation: `SELECT COUNT(*) FROM checkout_sessions WHERE status = 'ACTIVE' AND expires_at > NOW()`
    - Where to instrument: Query `checkout_sessions` table
    - Use case: Monitor abandoned checkouts

12. **Database Connection Pool Usage**
    - Definition: Percentage of available connections in use
    - Calculation: `(active_connections / max_connections) * 100`
    - Where to instrument: Supabase dashboard or connection pool metrics
    - Target: < 80%

## How to Measure

### Instrumentation Points

**Request Metrics:**
- Add metrics middleware before routes in `backend/src/app.ts`
- Use library like `prom-client` for Prometheus metrics
- Export metrics endpoint: `GET /api/metrics`

**Example:**
```typescript
// backend/src/middleware/metrics.ts
import { Counter, Histogram } from 'prom-client';

const httpRequestsTotal = new Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status']
});

const httpRequestDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration in seconds',
  labelNames: ['method', 'route']
});
```

**Service-Level Metrics:**
- Wrap Asaas API calls with timing and error tracking
- Add metrics to `checkoutService.ts`, `pricingService.ts`

**Example:**
```typescript
// In checkoutService.ts
const checkoutDuration = new Histogram({
  name: 'checkout_creation_duration_seconds',
  help: 'Time to create checkout session'
});

const asaasApiErrors = new Counter({
  name: 'asaas_api_errors_total',
  help: 'Total Asaas API errors',
  labelNames: ['endpoint', 'status_code']
});
```

**Database Metrics:**
- Wrap Supabase queries with timing
- Add to repository layer or create database client wrapper

**Example:**
```typescript
// In repository wrapper
const dbQueryDuration = new Histogram({
  name: 'database_query_duration_seconds',
  help: 'Database query duration',
  labelNames: ['table', 'operation']
});
```

### Dashboards

**Grafana Dashboard (if using Prometheus):**
- Panels: Request rate, error rate, latency (p50/p95/p99), checkout success rate
- Alerts: Error rate > 5%, checkout success rate < 90%, p95 latency > 2s

**Supabase Dashboard:**
- Query performance: Slow queries, table sizes, connection pool usage
- Custom queries: Enrollment conversion rate, voucher usage rate

**Render Dashboard:**
- Application logs: Filter by error level, search by enrollment ID
- Metrics: CPU, memory, request rate (if available)

### Queries for Business Metrics

**Enrollment Conversion Rate:**
```sql
SELECT 
  COUNT(DISTINCT e.id) as total_enrollments,
  COUNT(DISTINCT p.enrollment_id) as enrollments_with_payment,
  ROUND(COUNT(DISTINCT p.enrollment_id)::numeric / COUNT(DISTINCT e.id)::numeric * 100, 2) as conversion_rate
FROM enrollments e
LEFT JOIN payments p ON e.id = p.enrollment_id AND p.status = 'paid'
WHERE e.created_at >= NOW() - INTERVAL '30 days';
```

**Average Time to Payment:**
```sql
SELECT 
  AVG(EXTRACT(EPOCH FROM (p.paid_at - e.created_at)) / 86400) as avg_days_to_payment
FROM enrollments e
JOIN payments p ON e.id = p.enrollment_id
WHERE p.status = 'paid'
  AND p.paid_at >= NOW() - INTERVAL '30 days';
```

**Voucher Usage Rate:**
```sql
SELECT 
  COUNT(*) FILTER (WHERE voucher_id IS NOT NULL) as checkouts_with_voucher,
  COUNT(*) as total_checkouts,
  ROUND(COUNT(*) FILTER (WHERE voucher_id IS NOT NULL)::numeric / COUNT(*)::numeric * 100, 2) as voucher_usage_rate
FROM checkout_sessions
WHERE created_at >= NOW() - INTERVAL '30 days';
```

**Document Approval Time:**
```sql
SELECT 
  AVG(EXTRACT(EPOCH FROM (reviewed_at - uploaded_at)) / 3600) as avg_hours_to_approval
FROM documents
WHERE status = 'approved'
  AND reviewed_at IS NOT NULL
  AND reviewed_at >= NOW() - INTERVAL '30 days';
```
