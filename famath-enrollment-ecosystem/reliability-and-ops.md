# Reliability and Operations

## Implemented Mechanisms

### Timeouts

**Agent**: 20s timeout for Z-API calls, 10s for ViaCEP, 30s for OpenAI. Prevents indefinite blocking on external API calls.

**Portal**: No explicit timeout for Asaas API calls (relies on Node.js default ~2 minutes). Risk of long-running requests during Asaas outages.

**Admin**: No explicit timeouts configured.

### Retries

**Agent**: BullMQ automatic retry for failed jobs (2 attempts, 5s exponential backoff). ViaCEP has manual retry (2 attempts, 1s/2s backoff). OpenAI timeout falls back to state machine mode.

**Portal**: No retry logic for Asaas API calls. Single attempt, failure returns error to user.

**Admin**: No retry logic.

### Idempotency

**Payment confirmation**: Checks if already processed (idempotent). Prevents duplicate enrollment status updates.

**Voucher usage**: Tracked in database (prevents reuse). UNIQUE constraint on voucher_usage table.

**Checkout creation**: No idempotency keys. Duplicate checkout sessions possible if user retries.

**Enrollment creation**: No idempotency keys. Duplicate enrollments possible if user retries.

### Rate Limiting

**Portal**: Global rate limit (100 requests per 15 minutes per IP). Prevents abuse.

**Admin**: Rate limiting (100 requests per 15 minutes). Prevents abuse.

**Agent**: No rate limiting on webhook endpoint. Risk of webhook spam.

### Deduplication

**Agent**: Message deduplication via job ID (`{phone}-{timestamp}`). Prevents duplicate processing from webhook retries.

**Portal**: No deduplication for checkout creation or enrollment creation.

**Admin**: No deduplication.

### Logging

**Agent**: Pino structured logging with sanitization (CPF, phone, email masked). Error logs include stack traces.

**Portal**: Morgan for HTTP logs (method, path, status, duration). console.error for errors. No sanitization.

**Admin**: Winston structured logging. No sanitization.

**Correlation IDs**: Not implemented in any component.

### Automatic Status Updates

**Database triggers**: Update enrollment status when all required documents approved. Automatic protocol generation.

**Scheduled jobs**: Daily cron job updates voucher and campaign expiration status (Admin component).

### Graceful Shutdown

**Portal**: Graceful shutdown handler (10s timeout). Allows in-flight requests to complete.

**Agent**: No explicit graceful shutdown. Jobs may be interrupted on restart.

**Admin**: No explicit graceful shutdown.

## Known Gaps

**Portal**: No retry logic for Asaas API calls. No idempotency keys for checkout creation. No timeout configuration for Asaas API. No structured logging with correlation IDs.

**Admin**: No retry logic. No timeout configuration. No structured logging with correlation IDs.

**Agent**: No webhook signature verification. No dead-letter queue for failed jobs after retries exhausted. No structured logging with correlation IDs.

**Ecosystem**: No metrics collection (Prometheus/StatsD). No distributed tracing. No centralized error tracking. Legacy SQL Server sync has no retry mechanism.

## Operational Playbook

### Missing Payment

**Symptoms**: Enrollment status shows "Aguardando Pagamento" but payment completed in Asaas.

**Diagnosis**:
1. Check `payments` table for enrollment_id, verify `status` field
2. Query Asaas API with `payment_reference` to get current status
3. Check application logs for Asaas API errors (Portal logs)
4. Check if polling job ran (if implemented)

**Resolution**: Manually trigger `processPaymentConfirmation()` if endpoint exists. Update enrollment status directly in database if payment confirmed in Asaas.

### Agent Stuck

**Symptoms**: WhatsApp messages not being processed, no responses from agent.

**Diagnosis**:
1. Check Redis connection (Agent logs show "Redis connected")
2. Check BullMQ queue for pending jobs: `KEYS bull:messages:*` in Redis
3. Check worker logs for errors (Agent logs)
4. Verify Z-API credentials and API status
5. Check OpenAI API status and rate limits
6. Check if worker process is running

**Resolution**: Restart worker process. Clear stuck jobs from queue. Verify external API credentials.

### Legacy Sync Missing

**Symptoms**: Enrollment exists in Supabase but not in SQL Server.

**Diagnosis**:
1. Check application logs for SQL Server sync errors (Portal logs)
2. Query `enrollments` table for records without sync confirmation (if sync_status column exists)
3. Check SQL Server connection credentials in environment variables
4. Verify SQL Server API endpoint is accessible

**Resolution**: Manually trigger sync for failed enrollments (if endpoint exists). Reconcile data between Supabase and SQL Server manually. Check sync function implementation status (Planned: not implemented).

### Document Status Not Updating

**Symptoms**: All required documents approved but enrollment status not updated.

**Diagnosis**:
1. Verify trigger exists in database: `SELECT * FROM pg_trigger WHERE tgname = 'auto_update_enrollment_status'`
2. Check `documents` table for all required documents with `status = 'approved'`
3. Check `entry_modalities.required_documents` array matches document types
4. Check PostgreSQL logs for trigger execution errors
5. Verify enrollment status manually: `SELECT status FROM enrollments WHERE id = ?`

**Resolution**: Manually update enrollment status if trigger failed. Check trigger logic for edge cases. Review required_documents configuration.

### Voucher Not Applying

**Symptoms**: Voucher code valid but discount not applied during checkout.

**Diagnosis**:
1. Check voucher status (must be 'ativo'): `SELECT * FROM vouchers WHERE code = ?`
2. Check expiration date: `SELECT expiration_date FROM vouchers WHERE code = ?`
3. Check voucher_usage table for existing usage: `SELECT * FROM voucher_usage WHERE voucher_id = ?`
4. Verify CPF matches voucher.cpf_aluno
5. Check discount calculation logic (percentage OR fixed, not both)

**Resolution**: Verify voucher is active and not expired. Check if voucher already used. Verify CPF matches. Review discount calculation in checkout service.

## Evidence

**Agent**: `deep-dives/agent/reliability-and-ops.md`, `src/services/redis.ts`, `src/queue.ts`, `src/utils/sanitizers.ts`

**Portal**: `deep-dives/portal/challenges.md`, `backend/src/middleware/errorHandler.ts`, `backend/src/services/checkoutService.ts`

**Admin**: `deep-dives/admin/challenges.md`, `backend/src/utils/logger.ts`, `backend/src/services/VoucherService.ts`
