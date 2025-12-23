# Results and Measurement

## Outcomes

**Qualitative Outcomes:**
- Campaign and voucher management moved from manual database operations to UI-based workflows
- Business rules (e.g., preventing code changes on used campaigns) are enforced automatically
- Status updates (voucher expiration, campaign activation) happen automatically via triggers and cron jobs
- Frontend form state management improved (fixed infinite request loops and checkbox persistence issues)

**Evidence:**
- `backend/src/services/CampaignService.ts` lines 121-124 - Business rule enforcement
- `database/supabase/supabase_schema_vouchers_campaigns_final.sql` lines 304-340 - Automatic status updates
- `frontend/src/components/Voucher/CampaignForm.tsx` - Fixed state management

## What is Measured Today

**Request Logging:**
- HTTP method, path, status code, duration (milliseconds)
- Logged for every request via `requestLogger` middleware

**Error Logging:**
- Error messages, stack traces, request path, HTTP method
- Logged via Winston logger in error handler

**Database Audit Trail:**
- All table changes logged to `audit_logs` table (via triggers)
- Includes old_values, new_values, action type, user ID, timestamp

**Cron Job Execution:**
- Daily cron job logs execution results to `audit_logs`
- Includes counts of updated vouchers and campaigns

**Evidence:**
- `backend/src/middlewares/requestLogger.ts` - Request duration logging
- `backend/src/middlewares/errorHandler.ts` - Error logging
- `database/supabase/supabase_schema_collaborators.sql` lines 78-91 - audit_logs table
- `database/supabase/supabase_cron_daily_updates.sql` lines 29-54, 70-94 - Cron job logging

## What to Measure Next

### API Performance Metrics
1. **Request latency (p50, p95, p99)** - Instrument in `requestLogger` middleware, export to Prometheus/StatsD
2. **Request rate (requests per second)** - Count requests in rate limiter, export as counter metric
3. **Error rate by endpoint** - Count errors by path in error handler, export as counter with path label
4. **Database query latency** - Add timing around Supabase client calls in repositories

### Business Metrics
5. **Voucher creation rate** - Count POST /vouchers requests, export as counter
6. **Campaign usage rate** - Count campaign_usage inserts, export as counter with campaign_id label
7. **Voucher expiration rate** - Count vouchers updated to 'expirado' status, export as counter
8. **Campaign activation rate** - Count campaigns updated to 'ativa' status, export as counter

### Reliability Metrics
9. **Failed request rate** - Count 5xx errors, export as counter
10. **Rate limit hits** - Count 429 responses, export as counter
11. **Database connection pool usage** - Monitor Supabase connection pool (if exposed)
12. **Cron job execution success rate** - Track cron job completion vs failures, export as counter

### Where to Instrument

**Request Logger Middleware:**
```typescript
// backend/src/middlewares/requestLogger.ts
// Add: metrics.histogram('http_request_duration_ms', duration, { method, path, status })
// Add: metrics.counter('http_requests_total', { method, path, status })
```

**Error Handler:**
```typescript
// backend/src/middlewares/errorHandler.ts
// Add: metrics.counter('http_errors_total', { statusCode, path, errorType })
```

**Rate Limiter:**
```typescript
// backend/src/middlewares/rateLimiter.ts
// Add: metrics.counter('rate_limit_hits_total', { endpoint })
```

**Repository Methods:**
```typescript
// backend/src/repositories/VoucherRepository.ts
// Add: const startTime = Date.now() before Supabase call
// Add: metrics.histogram('db_query_duration_ms', Date.now() - startTime, { table: 'vouchers', operation })
```

**Cron Job:**
```typescript
// database/supabase/supabase_cron_daily_updates.sql
// Add: INSERT INTO metrics (name, value, labels) VALUES ('cron_execution_success', 1, 'job=daily_updates')
```

### Dashboard Queries

**Prometheus/Grafana Examples:**

```
# Request rate by endpoint
rate(http_requests_total[5m])

# Error rate
rate(http_errors_total[5m])

# P95 latency by endpoint
histogram_quantile(0.95, rate(http_request_duration_ms_bucket[5m]))

# Voucher creation rate
rate(voucher_creations_total[5m])

# Campaign usage by campaign
sum by (campaign_id) (rate(campaign_usage_total[5m]))
```

### Database Queries for Metrics

**Voucher metrics:**
```sql
-- Vouchers created today
SELECT COUNT(*) FROM vouchers WHERE DATE(created_at) = CURRENT_DATE;

-- Vouchers by status
SELECT status, COUNT(*) FROM vouchers GROUP BY status;

-- Voucher expiration rate (last 7 days)
SELECT DATE(updated_at), COUNT(*) 
FROM vouchers 
WHERE status = 'expirado' AND updated_at >= NOW() - INTERVAL '7 days'
GROUP BY DATE(updated_at);
```

**Campaign metrics:**
```sql
-- Campaign usage by campaign
SELECT c.nome, COUNT(cu.id) as usos
FROM campaigns c
LEFT JOIN campaign_usage cu ON c.id = cu.campaign_id
GROUP BY c.id, c.nome;

-- Campaigns by status
SELECT status, COUNT(*) FROM campaigns GROUP BY status;
```

**API performance (from audit_logs):**
```sql
-- Request patterns (if logged to audit_logs)
SELECT resource_type, action, COUNT(*) 
FROM audit_logs 
WHERE created_at >= NOW() - INTERVAL '24 hours'
GROUP BY resource_type, action;
```

**Evidence:**
- `backend/src/middlewares/requestLogger.ts` - Where to add latency metrics
- `backend/src/middlewares/errorHandler.ts` - Where to add error metrics
- `database/supabase/supabase_cron_daily_updates.sql` - Where to add cron metrics
