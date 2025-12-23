# Reliability and Operations

## Timeouts

**HTTP Client Timeouts:**
- Supabase Storage upload: 30 seconds via httpx
- No timeout configured for Supabase Python client database operations

**Evidence:**
- `backend/app/services/storage_service.py:219` - `httpx.Client(timeout=30.0)`
- `backend/app/core/database.py` - No timeout configuration in Supabase client initialization

**Request Timeouts:**
- Uvicorn default (no explicit configuration)
- No timeout middleware for long-running endpoints

**Docker Health Check:**
- Health check timeout: 10 seconds
- Health check interval: 30 seconds
- Start period: 5 seconds
- Retries: 3

**Evidence:**
- `backend/Dockerfile:49` - `HEALTHCHECK --timeout=10s --interval=30s --start-period=5s --retries=3`

## Retries and Backoff

**No retry mechanism implemented.** Failures return errors immediately.

**Places where retries would help:**
- Supabase Storage upload failures
- Database connection errors
- CSV import row failures

**Evidence:**
- No retry decorators or retry logic found in codebase
- `backend/app/services/storage_service.py:234` - Exception raised directly on upload failure

## Idempotency Strategy

**Recurring Rule Processing:**
- Transaction key: `f"{date_str}-{amount}-{description.strip().lower()}-{category_id}-{account_id}"`
- Checked in-memory cache first, then database query
- Prevents duplicate transactions from same rule processing

**Evidence:**
- `backend/app/services/recurring_service.py:36` - Key generation
- `backend/app/services/recurring_service.py:47` - Duplicate check logic

**CSV Import:**
- No idempotency mechanism
- Same file imported twice creates duplicate transactions
- File name stored in `csv_imports` table but not used for deduplication

**Transaction Creation:**
- No unique constraint prevents duplicates
- Same transaction can be created multiple times

**Future Improvements:**
- Add unique constraint on transactions table
- Use file hash for CSV import deduplication
- Add idempotency key to transaction creation endpoint

## Rate Limiting

**Not implemented.** `slowapi>=0.1.9` listed in requirements but not configured.

**Evidence:**
- `backend/requirements.txt:37` - `slowapi>=0.1.9` dependency
- No usage of slowapi in codebase
- `backend/app/api/v1/routers/auth.py:228` - Comment mentions rate limiting should be applied

**Plan-Based Rate Limiting:**
- CSV imports limited per month via `PlanLimitsService`
- Not HTTP rate limiting, but business logic limit
- Enforced before processing starts

**Evidence:**
- `backend/app/services/plan_limits_service.py:102` - `ensure_can_import_csv()` method
- `backend/app/services/csv_import_service.py:46` - Limit check before processing

## Dead-Letter or Reprocessing Strategy

**Not implemented.** Failed operations return errors with no retry or reprocessing mechanism.

**CSV Import:**
- Failed rows returned in response with error messages
- No mechanism to retry failed rows
- Must re-upload entire file

**Recurring Rule Processing:**
- Errors logged but processing continues for other rules
- No retry queue for failed rules
- Must manually reprocess via endpoint

**Evidence:**
- `backend/app/services/csv_import_service.py:123` - Failed rows added to result, no retry
- `backend/app/services/recurring_service.py:450` - Errors logged, processing continues

## Logging and Observability

**Logging:**
- Python standard logging module
- Log levels: INFO, ERROR
- Logs include user_id, company_id, operation details

**Evidence:**
- `backend/app/core/security.py:35` - Logger initialization
- `backend/app/core/security.py:39` - Info logs for token verification
- `backend/app/main.py:71` - Error logging in exception handler

**Health Check:**
- `/health` endpoint tests database connection
- Returns status: "healthy" or "degraded"
- Includes database connection status and timestamp

**Evidence:**
- `backend/app/main.py:94` - Health check endpoint
- `backend/app/main.py:110` - Database connection test

**Metrics:**
- No structured metrics collection
- No APM or tracing
- No request duration tracking
- No error rate tracking

**Future Improvements:**
- Add structured logging (JSON format)
- Add request ID for tracing
- Add metrics endpoint (Prometheus format)
- Add APM integration (e.g., Sentry, Datadog)

## CI/CD

**Docker:**
- Multi-stage Dockerfile for production
- Docker Compose for local development
- Health check configured in Dockerfile

**Evidence:**
- `backend/Dockerfile` - Production multi-stage build
- `backend/Dockerfile.dev` - Development Dockerfile (referenced in docker-compose)
- `backend/docker-compose.yml` - Local development setup

**No CI/CD Pipeline Found:**
- No GitHub Actions, GitLab CI, or similar
- No automated tests in pipeline
- No deployment automation

**Tests:**
- pytest configured (`backend/pytest.ini`)
- Test files in `backend/tests/`
- Not integrated into CI/CD

**Evidence:**
- `backend/pytest.ini` - Test configuration
- `backend/tests/` - Test directory exists
- No `.github/workflows/` or CI config found

## Runbook Basics

### Troubleshooting Database Connection Issues

**Symptoms:** Health check returns "degraded", database errors in logs

**Steps:**
1. Check `/health` endpoint: `curl http://localhost:8000/health`
2. Verify Supabase credentials in environment variables
3. Check Supabase dashboard for service status
4. Review logs for connection errors: `docker compose logs backend`

**Evidence:**
- `backend/app/main.py:94` - Health check implementation
- `backend/app/core/config.py:16` - SUPABASE_URL and SUPABASE_SERVICE_ROLE_KEY required

### Troubleshooting CSV Import Failures

**Symptoms:** Import returns errors, some rows fail

**Steps:**
1. Check import history: `GET /api/v1/tools/csv/history`
2. Review error messages in `ImportResult.rows[].errors`
3. Verify CSV format matches template: `GET /api/v1/tools/csv/template`
4. Check plan limits: ensure not exceeding monthly import limit
5. Verify categories and accounts exist for failed rows

**Evidence:**
- `backend/app/api/v1/routers/import_csv.py:66` - History endpoint
- `backend/app/services/csv_import_service.py:122` - Error collection per row

### Troubleshooting Recurring Rule Processing

**Symptoms:** Rules not processing, transactions not created

**Steps:**
1. Check rule status: `GET /api/v1/recurring-rules?active_only=true`
2. Verify `last_processed_date` vs current month
3. Check for processing locks: `processing_lock = true` blocks processing
4. Check for duplicate transactions (may be silently skipped)
5. Review logs for processing errors

**Evidence:**
- `backend/app/services/recurring_service.py:86` - `_should_reprocess_rule()` logic
- `backend/app/services/recurring_service.py:245` - Processing lock check

### Troubleshooting Authentication Issues

**Symptoms:** 401 Unauthorized errors

**Steps:**
1. Verify token format (should be JWT with 3 parts)
2. Check token expiration: `exp` claim vs current time
3. Verify `sub` claim exists (user_id)
4. Check user has associated company: query `company_users` table
5. Review security logs for token validation errors

**Evidence:**
- `backend/app/core/security.py:49` - Token format validation
- `backend/app/core/security.py:79` - Expiration check
- `backend/app/core/security.py:67` - User ID extraction

### Where to Look for Issues

**Application Logs:**
- Docker: `docker compose logs backend`
- Local: stdout/stderr from uvicorn process

**Database:**
- Supabase dashboard: table data, query performance
- RLS policies: verify policies are active
- Indexes: check query performance

**Storage:**
- Supabase Storage dashboard: verify buckets exist
- File paths: check `csv_imports.file_path` for uploaded files

**Health Endpoint:**
- `GET /health` - Database connection status
- `GET /` - Basic app status

**Evidence:**
- `backend/app/main.py:84` - Root endpoint
- `backend/app/main.py:94` - Health endpoint
