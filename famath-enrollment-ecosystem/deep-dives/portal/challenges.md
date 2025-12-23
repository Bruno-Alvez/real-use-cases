# Reliability and Operations

## Timeouts

**HTTP Client Timeouts:**
- No explicit timeout configured for `fetch()` calls to Asaas API
- Relies on Node.js default timeout (no timeout by default, may hang indefinitely)
- Supabase client uses default timeout (not configured)

**Evidence:**
- `backend/src/services/checkoutService.ts` - `fetch()` calls without timeout option (lines 326, 683, 865)

**Database Query Timeouts:**
- Supabase client default timeout (not explicitly configured)
- No query-level timeout settings

**Evidence:**
- `backend/src/config/database.ts` - Supabase client initialization without timeout config

**Server Timeouts:**
- Express server no explicit request timeout
- Graceful shutdown timeout: 10 seconds (forces exit if shutdown not completed)

**Evidence:**
- `backend/src/server.ts` - Graceful shutdown timeout (line 59: `setTimeout(..., 10000)`)

## Retries and Backoff

**No retry logic implemented for:**
- Asaas API calls (checkout creation, customer creation)
- Database queries (Supabase client may retry internally, not verified)
- External service calls

**Evidence:**
- `backend/src/services/checkoutService.ts` - Single `fetch()` call, no retry wrapper
- `backend/src/services/paymentService.ts` - Single `fetch()` calls, no retry

**Backoff Strategy:**
- Not implemented

## Idempotency Strategy

**Checkout Session Creation:**
- No idempotency keys used
- No duplicate prevention (multiple checkout sessions possible for same enrollment)
- No check for existing active checkout session before creating new one

**Evidence:**
- `backend/src/services/checkoutService.ts` - `createPixCheckoutSession()` - No idempotency check (lines 543-744)
- `database/schema.sql` - `checkout_sessions` table has no unique constraint preventing duplicates

**Payment Processing:**
- `processPaymentConfirmation()` checks if payment already processed:
  - Queries payment by `asaasChargeId`
  - If `status === 'paid'`, returns early (idempotent)

**Evidence:**
- `backend/src/services/paymentService.ts` - `processPaymentConfirmation()` (lines 187-191)

**Voucher Usage:**
- `voucher_usage` table tracks voucher usage (prevents reuse)
- Query checks for existing usage before applying voucher
- Race condition: two concurrent checkouts may both see voucher as available

**Evidence:**
- `backend/src/services/checkoutService.ts` - `fetchApplicableVoucher()` checks `voucher_usage` (lines 102-115)

**Deduplication Keys:**
- Enrollment protocol number: UNIQUE constraint prevents duplicates
- Student CPF: UNIQUE constraint prevents duplicate students
- Checkout session ID: Asaas-generated, unique per session

**Evidence:**
- `database/schema.sql` - UNIQUE constraints (lines 17: `cpf`, 73: `protocol_number`)

## Rate Limiting

**API Rate Limiting:**
- Express rate limiter configured: 100 requests per 15 minutes per IP
- Applied to all `/api/` routes
- Uses `express-rate-limit` middleware

**Evidence:**
- `backend/src/config/security.ts` - Rate limit config (lines 36-42)
- `backend/src/app.ts` - Rate limiter applied (line 32)

**Asaas API Rate Limits:**
- Not handled (no rate limit detection or backoff)
- Asaas may return 429 (Too Many Requests), not handled gracefully

**Evidence:**
- `backend/src/services/checkoutService.ts` - Error handling doesn't check for 429 status

## Dead-Letter or Reprocessing Strategy

**No dead-letter queue implemented:**
- Failed checkout creations are not retried
- Failed payment confirmations are not queued for retry
- No background job system for reprocessing

**Failed Payment Processing:**
- Webhook handler not implemented (payment status checked via polling)
- No mechanism to reprocess failed payment confirmations

**Evidence:**
- No queue system in codebase (no Bull, BullMQ, or similar)
- `backend/src/services/paymentService.ts` - `processPaymentConfirmation()` is synchronous, no retry

## Logging and Observability

**Request Logging:**
- Morgan middleware for HTTP request logging
- Development: custom format with method, URL, status, response time, body (sanitized)
- Production: combined format (Apache common log format)
- Custom request logger: logs method, path, status, duration, IP, timestamp

**Evidence:**
- `backend/src/middleware/logger.ts` - Morgan configuration (lines 21-31)
- `backend/src/middleware/logger.ts` - Custom request logger (lines 34-56)

**Error Logging:**
- Console.error for errors (not structured logging)
- Error handler middleware logs errors with stack trace in development
- No correlation IDs or request tracing

**Evidence:**
- `backend/src/middleware/errorHandler.ts` - Error handler (lines 12-31)
- `backend/src/services/checkoutService.ts` - Console.error calls throughout

**Application Logging:**
- Console.log/console.error used throughout (not structured)
- No log levels (INFO, WARN, ERROR)
- No centralized logging service (CloudWatch, Datadog, etc.)

**Evidence:**
- `backend/src/services/checkoutService.ts` - Multiple `console.log()` and `console.error()` calls

**Database Query Logging:**
- Supabase client doesn't log queries by default
- No query logging middleware

**Observability:**
- No metrics collection (Prometheus, StatsD)
- No distributed tracing (OpenTelemetry, Jaeger)
- No APM (Application Performance Monitoring)

## CI/CD

**Deployment Configuration:**
- Render.com blueprint: `backend/render.yaml`
- Docker multi-stage build: `backend/Dockerfile`
- Frontend: Vercel deployment (configured via `frontend/vercel.json`)

**Evidence:**
- `backend/render.yaml` - Render deployment config
- `backend/Dockerfile` - Multi-stage Docker build (lines 1-89)
- `frontend/vercel.json` - Vercel config

**Build Process:**
- Backend: TypeScript compilation (`npm run build`)
- Frontend: Vite build (`npm run build`)
- Docker: Multi-stage (dependencies → builder → runner)

**Evidence:**
- `backend/package.json` - Build script (line 9: `"build": "tsc"`)
- `backend/Dockerfile` - Build stages (lines 9-40)

**No CI Pipeline:**
- No GitHub Actions, GitLab CI, or similar
- No automated testing in CI
- No automated deployment on merge to main (Render auto-deploy enabled)

**Evidence:**
- No `.github/workflows/` or `.gitlab-ci.yml` files

## Runbook Basics

### Troubleshooting Checkout Failures

**Symptoms:** Student cannot complete checkout, receives error message.

**Steps:**
1. Check application logs for error message (Render dashboard or logs)
2. Verify Asaas API key is configured (`ASAAS_PRODUCTION_API_KEY` env var)
3. Check Asaas API status (external service)
4. Verify student data (address fields required by Asaas):
   - Query `students` table for enrollment's `student_id`
   - Check: `address`, `address_number`, `postal_code`, `province` are not null
5. Check `checkout_sessions` table for existing session:
   - Query by `enrollment_id`, `status = 'ACTIVE'`
   - If exists, return existing checkout URL instead of creating new
6. Verify pricing calculation:
   - Check `course_pricing` table for course/shift
   - Verify `is_active = true` and validity dates include today
7. Check voucher if applied:
   - Query `vouchers` table by CPF and code
   - Verify `status = 'ativo'`, `data_expiracao >= today`
   - Check `voucher_usage` for existing usage

**Where to Look:**
- Application logs: Render dashboard → Service → Logs
- Database: Supabase dashboard → Table Editor
- Asaas dashboard: Check for failed API calls

**Evidence:**
- `backend/src/services/checkoutService.ts` - Error handling and logging (lines 363-377, 720-743)

### Troubleshooting Payment Status Not Updating

**Symptoms:** Payment completed in Asaas but enrollment status not updated.

**Steps:**
1. Check `payments` table for enrollment:
   - Query by `enrollment_id`
   - Check `status` field (should be 'paid' after payment)
2. Verify payment reference matches Asaas charge ID:
   - Query Asaas API: `GET /payments/{chargeId}`
   - Compare with `payments.payment_reference`
3. Check if webhook received (if webhook handler implemented):
   - Check application logs for webhook requests
   - Verify webhook signature validation
4. Manually trigger payment confirmation:
   - Call `paymentService.processPaymentConfirmation(asaasChargeId)`
   - Check logs for errors

**Where to Look:**
- `payments` table: Supabase dashboard
- Asaas dashboard: Payment status
- Application logs: Webhook requests

**Evidence:**
- `backend/src/services/paymentService.ts` - `processPaymentConfirmation()` (lines 175-230)

### Troubleshooting Enrollment Status Not Updating Automatically

**Symptoms:** All documents approved but enrollment status still "Documentos em Análise".

**Steps:**
1. Verify trigger exists:
   - Query PostgreSQL: `SELECT * FROM pg_trigger WHERE tgname = 'auto_update_enrollment_status_trigger'`
2. Check document statuses:
   - Query `documents` table for enrollment
   - Verify all required documents have `status = 'approved'`
3. Check entry modality configuration:
   - Query `entry_modalities` table for enrollment's `entry_modality_id`
   - Verify `required_documents` array is populated
4. Check trigger execution logs:
   - PostgreSQL logs (if accessible) for trigger RAISE LOG statements
5. Manually update status if trigger failed:
   - Update `enrollments` table: `current_step = 'Aguardando Pagamento'`

**Where to Look:**
- `documents` table: Supabase dashboard
- `entry_modalities` table: Required documents configuration
- PostgreSQL logs: Render dashboard (if accessible)

**Evidence:**
- `database/triggers/auto_update_enrollment_status.sql` - Trigger implementation (lines 6-143)

### Health Check Endpoint

**Endpoint:** `GET /api/health`

**Response:**
```json
{
  "success": true,
  "message": "API FAMATH está funcionando!",
  "timestamp": "2025-01-15T10:30:00.000Z"
}
```

**Use Cases:**
- Render health check (configured in `render.yaml`)
- Load balancer health checks
- Monitoring service pings

**Evidence:**
- `backend/src/routes/index.ts` - Health check route (lines 16-22)
- `backend/render.yaml` - Health check path (line 16)
