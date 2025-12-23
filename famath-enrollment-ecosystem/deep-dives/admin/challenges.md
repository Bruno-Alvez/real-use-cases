# Reliability and Operations

## Timeouts

No explicit timeouts configured for Supabase client requests. Relies on default Node.js HTTP timeout (typically 2 minutes) and Supabase client defaults.

**Evidence:**
- `backend/src/config/supabase.ts` - No timeout configuration
- Supabase client uses default fetch timeout

## Retries and Backoff

No retry logic implemented in application code. Only retry mechanism is in code generation (10 attempts with 100ms fixed delay).

**Evidence:**
- `backend/src/services/VoucherService.ts` lines 253-271 - Code generation retry only
- No retry wrappers in repositories or services

## Idempotency Strategy

No idempotency keys or deduplication implemented. Operations rely on natural uniqueness constraints:
- Voucher codes: UNIQUE constraint on `vouchers.codigo`
- Campaign codes: UNIQUE constraint on `campaigns.codigo_promocional`
- Campaign usage: UNIQUE constraint on `(campaign_id, student_id, enrollment_id)`
- Voucher usage: UNIQUE constraint on `(voucher_id, enrollment_id)`

Duplicate requests create duplicate resources (except where database constraints prevent it).

**Evidence:**
- `database/supabase/supabase_schema_vouchers_campaigns_final.sql` lines 28, 82, 185, 223 - UNIQUE constraints
- No idempotency key handling in controllers

## Rate Limiting

Global rate limit: 100 requests per 15 minutes per IP address. Additional limits:
- Auth endpoints: 5 requests per 15 minutes (skips successful requests)
- Create endpoints: 10 requests per minute

Rate limiter uses in-memory store (no Redis), so limits are per-instance, not shared across multiple API servers.

**Evidence:**
- `backend/src/middlewares/rateLimiter.ts`
- `backend/src/app.ts` line 34 - Applied to API routes

## Dead-Letter or Reprocessing Strategy

No dead-letter queue or reprocessing mechanism. Failed operations return errors to clients immediately. No background retry or manual reprocessing workflow.

**Evidence:**
- No queue infrastructure
- No dead-letter handling
- Errors logged but not queued for retry

## Logging and Observability

**Implemented:**
- Winston logger with JSON format in production, colored console in development
- Request/response logging middleware (method, path, status code, duration)
- Error logging with stack traces
- Structured logging with metadata (user ID, request path, etc.)

**Not Implemented:**
- Distributed tracing (no correlation IDs)
- Metrics collection (no Prometheus/StatsD)
- Log aggregation (logs only to console)
- Performance monitoring (no APM integration)

**Evidence:**
- `backend/src/utils/logger.ts` - Winston configuration
- `backend/src/middlewares/requestLogger.ts` - Request logging
- `backend/src/middlewares/errorHandler.ts` - Error logging

## CI/CD

No CI/CD pipelines found in repository. Dockerfile exists for containerization but no GitHub Actions, GitLab CI, or similar configuration files.

**Evidence:**
- `backend/Dockerfile` - Multi-stage build
- No `.github/workflows/` or `.gitlab-ci.yml` files

## Runbook Basics

### Troubleshooting Failed Voucher Creation

1. Check logs for error message (`backend/src/utils/logger.ts` outputs to console)
2. Verify CPF format (must be 11 digits after normalization)
3. Check database constraint violations (duplicate codigo, invalid CPF)
4. Verify Supabase connection (`SUPABASE_URL`, `SUPABASE_SERVICE_KEY` env vars)
5. Check rate limit status (429 errors indicate limit exceeded)

**Where to look:**
- Application logs: Console output (Winston)
- Database logs: Supabase dashboard → Logs
- Error responses: HTTP status code and error message in response body

### Troubleshooting Campaign Update Failures

1. Check logs for business rule violations (e.g., "Não é possível alterar o código promocional")
2. Verify campaign has no usage if changing code (`campaign_usage` table)
3. Check shift value mapping (must be 'MANHA', 'TARDE', 'NOITE', 'SABADOS')
4. Verify database CHECK constraint errors (invalid shift value)
5. Check for partial update failures (campaign updated but courses/shifts not updated)

**Where to look:**
- `backend/src/services/CampaignService.ts` lines 116-130 - Business rule validation
- `backend/src/repositories/CampaignRepository.ts` lines 229-241 - Logging of campaign data
- Database: `campaign_shifts` table CHECK constraint

### Troubleshooting Infinite Request Loops (Frontend)

1. Check browser console for `ERR_INSUFFICIENT_RESOURCES` errors
2. Verify `initializationRef` is preventing re-initialization (`CampaignForm.tsx`)
3. Check useEffect dependencies (should not include functions that change on every render)
4. Verify API is returning data (not causing re-renders due to error handling)

**Where to look:**
- Browser DevTools → Network tab (repeated requests)
- Browser DevTools → Console (error messages)
- `frontend/src/components/Voucher/CampaignForm.tsx` - useEffect dependencies

### Daily Cron Job Monitoring

1. Check `audit_logs` table for `cron_daily_report` entries
2. Verify voucher expiration updates (count of vouchers with status='expirado')
3. Verify campaign status transitions (agendada → ativa, ativa → finalizada)
4. Check for cron job execution errors in database logs

**Where to look:**
- Database: `audit_logs` table filtered by `action LIKE 'cron_%'`
- Database: `vouchers` table status distribution
- Database: `campaigns` table status distribution
- Supabase dashboard → Database → Logs

**Evidence:**
- `database/supabase/supabase_cron_daily_updates.sql` - Cron job script
- `database/supabase/supabase_schema_collaborators.sql` lines 78-91 - audit_logs table
