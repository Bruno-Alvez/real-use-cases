# Technical Decisions

## Decision 1: Manual JWT Token Decoding vs Signature Verification

**Decision:** Decode JWT payload manually via base64 for development context resolution. Signature verification required for production.

**Alternatives Considered:**
- Use `jose.jwt.decode()` with Supabase JWT secret for signature verification
- Call Supabase Auth API to validate token
- Use Supabase client's built-in auth methods

**Why Chosen:**
- Faster context resolution during development
- Simpler MVP implementation
- Token structure and expiration validation sufficient for development
- Enables rapid iteration on multi-tenant context resolution

**Consequences:**
- Tokens with invalid signatures accepted if structure is valid
- Cannot detect token tampering
- Not suitable for production

**Production Requirements:**
- Verify signature using provider JWT secret or JWKS endpoint
- Validate issuer and audience claims
- Reject tokens without valid signature
- Implement token refresh mechanism

**Evidence:**
- `backend/app/core/security.py:42` - Manual base64url_decode implementation
- `backend/app/core/security.py:44` - Comment notes signature verification should be added

## Decision 2: Synchronous Processing vs Async Queues

**Decision:** Process recurring rules and CSV imports synchronously within HTTP request handlers. Architecture designed to migrate heavy operations to background queues when volume requires.

**Alternatives Considered:**
- Background job queue (Celery, RQ, BullMQ)
- Async task processing with FastAPI BackgroundTasks
- Scheduled cron jobs for recurring rules

**Why Chosen:**
- Simpler initial architecture (no queue infrastructure)
- Faster MVP implementation
- No additional infrastructure dependencies
- Immediate feedback to user
- Service layer separation enables queue migration without API changes

**Consequences:**
- Long-running operations block HTTP requests
- No retry mechanism for failed processing
- Cannot scale processing independently
- Timeout risk for large CSV files

**Design Intent:**
- Service layer abstraction allows queue insertion without changing API contracts
- CSV import and forecast generation are candidates for background processing
- Recurring rule processing can move to scheduled jobs when needed

**Future Improvements:**
- Move CSV import to background task with job status endpoint
- Use queue system for recurring rule processing
- Add job status tracking table
- Implement retry logic with exponential backoff

**Evidence:**
- `backend/app/services/recurring_service.py:337` - Synchronous loop processing
- `backend/app/services/csv_import_service.py:41` - Synchronous CSV processing
- No queue/worker code found in codebase

## Decision 3: In-Memory Deduplication Cache vs Database-Only

**Decision:** Use in-memory Set cache for transaction deduplication during recurring rule processing, with database fallback.

**Alternatives Considered:**
- Database-only duplicate check (query before each insert)
- Unique constraint on transaction table
- External cache (Redis)

**Why Chosen:**
- Reduces database queries during batch processing
- Fast lookup for same-request duplicates
- Database check still validates against existing data
- No external dependencies

**Consequences:**
- Cache is request-scoped (not shared across requests)
- Multiple concurrent requests could create duplicates
- Memory usage scales with number of rules processed

**Future Improvements:**
- Add unique constraint: `UNIQUE(company_id, date, amount, description, category_id, account_id)`
- Use database-level deduplication
- Consider distributed cache if scaling horizontally

**Evidence:**
- `backend/app/services/recurring_service.py:287` - `processed_cache: Set[str]` parameter
- `backend/app/services/recurring_service.py:60` - Cache check before database query
- `backend/app/services/recurring_service.py:36` - Transaction key generation

## Decision 4: Company ID Resolution via Database Query vs Token Claims

**Decision:** Query `company_users` table on each request to resolve `company_id` from `user_id`.

**Alternatives Considered:**
- Include `company_id` in JWT token claims
- Cache company_id in session or token
- Single company per user assumption

**Why Chosen:**
- Supports future multi-company per user feature
- Single source of truth (database)
- Handles company membership changes immediately
- No token refresh needed when company changes

**Consequences:**
- Extra database query on every authenticated request
- Slight latency overhead
- Database load scales with request volume

**Future Improvements:**
- Cache company_id in request context after first lookup
- Consider including in token if single-company assumption holds
- Add database index on `(user_id, company_id)` if not exists

**Evidence:**
- `backend/app/core/security.py:143` - `get_current_company_id()` queries database
- `backend/app/core/security.py:170` - Queries `company_users` table

## Decision 5: Plan Limits as Service vs Middleware

**Decision:** Centralize plan limit checks in `PlanLimitsService` called explicitly from service layer.

**Alternatives Considered:**
- Middleware that intercepts requests
- Database triggers
- Feature flags in token claims

**Why Chosen:**
- Explicit and visible in code flow
- Easy to test in isolation
- Flexible (can check multiple limits in one call)
- No magic behavior

**Consequences:**
- Must remember to call limit checks in each feature
- Risk of forgetting to add checks
- Duplicate code if checks needed in multiple places

**Future Improvements:**
- Decorator pattern for automatic limit checking
- Middleware for common endpoints
- Database-level constraints where possible

**Evidence:**
- `backend/app/services/plan_limits_service.py:38` - Service implementation
- `backend/app/services/csv_import_service.py:46` - Explicit call to `ensure_can_import_csv()`

## Decision 6: Supabase Client Singleton vs Per-Request

**Decision:** Global singleton Supabase client instance with service role key.

**Alternatives Considered:**
- Create new client per request
- Connection pooling via SQLAlchemy
- Direct PostgreSQL connection

**Why Chosen:**
- Supabase Python client handles connection management
- Service role key bypasses RLS (needed for server-side operations)
- Singleton reduces initialization overhead
- Simpler than managing connection pools

**Consequences:**
- All operations use service role (bypasses RLS)
- Must manually filter by `company_id` in all queries
- No per-request connection isolation
- Potential security risk if `company_id` filtering is missed

**Future Improvements:**
- Add repository-level validation that `company_id` is always filtered
- Consider connection pooling for high concurrency
- Add query logging to detect missing filters

**Evidence:**
- `backend/app/core/database.py:15` - Global `_supabase_client` variable
- `backend/app/core/database.py:32` - Singleton pattern with service role key
- `backend/app/core/database.py:21` - Comment notes RLS bypass

## Decision 7: Recurring Rule Versioning vs In-Place Updates

**Decision:** Create new rule version when updating future transactions only, update existing rule when updating all transactions.

**Alternatives Considered:**
- Always create new version
- Always update in place
- Soft delete old rule, create new

**Why Chosen:**
- Supports "update all" vs "update future only" use cases
- Maintains history via `original_rule_id` and `effective_from_date`
- Allows querying active version via `superseded_by IS NULL`
- Preserves audit trail

**Consequences:**
- More complex update logic
- Multiple versions in database
- Must filter out superseded versions in queries
- Transaction linking to correct version

**Future Improvements:**
- Add version history endpoint
- Consider time-travel queries for historical forecasts
- Add version comparison UI

**Evidence:**
- `backend/app/services/recurring_service.py:123` - `update_smart()` method
- `backend/app/services/recurring_service.py:203` - `create_rule_version()` call
- `backend/app/repositories/recurring_repository.py` - Versioning logic

## Decision 8: CSV Import Row-by-Row vs Batch Insert

**Decision:** Process CSV rows sequentially, creating transactions one at a time.

**Alternatives Considered:**
- Batch insert all valid rows in single transaction
- Validate all rows first, then insert batch
- Stream processing for large files

**Why Chosen:**
- Simpler error handling (per-row status)
- Immediate validation feedback
- Can return partial results if some rows fail
- No transaction rollback complexity

**Consequences:**
- Slower for large files (N database writes)
- No atomicity (partial success possible)
- Higher database load
- No transaction-level rollback

**Future Improvements:**
- Batch insert valid rows in chunks (e.g., 100 at a time)
- Use database transaction for all-or-nothing import option
- Add progress tracking for large files
- Consider async processing with job status

**Evidence:**
- `backend/app/services/csv_import_service.py:80` - Sequential row processing loop
- `backend/app/services/csv_import_service.py:148` - Individual `transaction_service.create()` calls

## Decision 9: Forecast Algorithm Storage vs On-Demand Calculation

**Decision:** Store forecast results with algorithm metadata (EWMA, trend, seasonality) in database.

**Alternatives Considered:**
- Calculate on-demand from historical data
- Cache in memory/Redis
- Store only final values

**Why Chosen:**
- Fast retrieval for dashboard
- Can compare algorithm results over time
- Supports debugging and algorithm tuning
- Historical forecast accuracy tracking

**Consequences:**
- More database storage
- Must recalculate when historical data changes
- Schema includes algorithm-specific fields

**Future Improvements:**
- Add recalculation trigger on transaction updates
- Version algorithm parameters
- Store algorithm version for comparison

**Evidence:**
- `database/migrations/03_migrate_tables_to_company_id.sql` - Forecast table includes `ewma_revenue`, `trend_revenue`, `seasonality_factor`
- `backend/app/services/forecast_service.py:141` - Stores algorithm results

## Decision 10: Storage Service Direct REST API vs Python Client

**Decision:** Use direct REST API with httpx for avatar uploads, Supabase Python client for other uploads.

**Alternatives Considered:**
- Use Python client for all uploads
- Use only REST API
- Use only Python client

**Why Chosen:**
- Python client had content-type issues for avatars
- REST API allows explicit header control
- Other uploads work fine with Python client
- Fallback to REST when client fails

**Consequences:**
- Inconsistent upload methods
- More code to maintain
- Direct API calls require manual URL construction

**Future Improvements:**
- Standardize on one method
- Fix Python client content-type handling
- Add retry logic for upload failures

**Evidence:**
- `backend/app/services/storage_service.py:195` - Direct REST API for avatars
- `backend/app/services/storage_service.py:260` - Python client for CSV uploads
- `backend/app/services/storage_service.py:188` - Comment about content-type issues
