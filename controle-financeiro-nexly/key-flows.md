# Key Flows

## Flow 1: CSV Import

**Trigger:** `POST /api/v1/tools/csv` with multipart file upload

**Steps:**
1. Router validates file extension (`.csv`)
2. Reads file bytes (`await file.read()`)
3. Uploads to Supabase Storage via `StorageService.upload_csv_import_from_bytes()`
4. `PlanLimitsService.ensure_can_import_csv()` checks monthly limit
5. `CSVImportService.import_csv()` decodes UTF-8, parses CSV with `csv.DictReader`
6. Validates required columns: `date`, `type`, `amount`, `category`, `account`
7. Pre-loads categories and accounts into memory maps
8. For each row:
   - Validates date format (DD/MM/YYYY)
   - Validates amount (positive number)
   - Resolves category and account by name (case-insensitive)
   - Creates transaction via `TransactionService.create()`
9. Records import in `csv_imports` table with success/failed counts
10. Returns `ImportResult` with per-row status

**Failure Points:**
- Invalid file encoding (UnicodeDecodeError)
- Missing required columns
- Invalid date format
- Category or account not found
- Plan limit exceeded (403 Forbidden)
- Storage upload failure
- Transaction creation failure

**Handling:**
- File encoding errors return 400 with message
- Missing columns return 400 with list of missing columns
- Invalid rows marked as failed, processing continues
- Plan limit check happens before processing (early failure)
- Storage errors propagate as exceptions
- Individual row failures don't stop batch processing

**Evidence:**
- `backend/app/api/v1/routers/import_csv.py:15` - Endpoint definition
- `backend/app/services/csv_import_service.py:41` - Import logic
- `backend/app/services/plan_limits_service.py:102` - Limit check
- `backend/app/services/storage_service.py:275` - Storage upload

## Flow 2: Recurring Rule Processing

**Trigger:** `POST /api/v1/recurring-rules/process-all` or `POST /api/v1/recurring-rules/{rule_id}/process`

**Steps:**
1. `RecurringService.process_all_rules()` or `process_rule()` called
2. Resets monthly processed flags (if processing all)
3. Fetches active rules filtered by `company_id`
4. For each rule:
   - Checks if already processed this month (`_should_reprocess_rule()`)
   - Checks processing lock
   - Calculates target date: `start_date + repetitions_completed` months
   - Handles year rollover (month > 12)
   - Generates transaction key: `{date}-{amount}-{description}-{category_id}-{account_id}`
   - Checks for duplicate transaction (`_transaction_exists()`)
   - Creates transaction if not duplicate
   - Updates rule: `repetitions_completed++`, `last_processed_date`, `processed` flag
5. Returns summary: `{processed, skipped, errors, total}`

**Failure Points:**
- Rule already processed this month
- Processing lock active (423 Locked)
- Transaction already exists (deduplication check)
- Invalid date calculation (year rollover edge cases)
- Transaction creation failure
- Rule update failure

**Handling:**
- Already processed rules are skipped (not an error)
- Locked rules return 423 HTTP status
- Duplicate transactions detected via key matching (in-memory cache + database query)
- Year rollover handled with while loop
- Invalid dates use `min(day_of_month, 28)` to avoid Feb 30 errors
- Individual rule failures logged, processing continues
- No retry mechanism (failures require manual reprocessing)

**Idempotency:**
- Transaction key prevents duplicates: `f"{date_str}-{amount}-{description.strip().lower()}-{category_id}-{account_id}"`
- Checked in memory cache first, then database query
- Same rule processed twice creates no duplicate transactions

**Evidence:**
- `backend/app/services/recurring_service.py:223` - `process_rule()` method
- `backend/app/services/recurring_service.py:36` - Transaction key generation
- `backend/app/services/recurring_service.py:47` - Duplicate check
- `backend/app/api/v1/routers/recurring_rules.py:89` - Endpoint

## Flow 3: Forecast Generation (Advanced)

**Trigger:** `POST /api/v1/forecasts/generate?months=3&advanced=true`

**Steps:**
1. `PlanLimitsService.ensure_advanced_forecast_enabled()` checks Pro plan
2. `ForecastService.generate_advanced()` fetches historical transactions
3. Groups transactions by month (YYYY-MM)
4. Calculates EWMA (Exponential Weighted Moving Average) for revenue and expense
5. Detects trend using linear regression slope
6. Calculates seasonality factors per month (1-12)
7. For each forecast month:
   - Applies EWMA baseline
   - Applies trend coefficient
   - Applies seasonal factor
   - Stores forecast with algorithm metadata
8. Returns list of `ForecastResponse`

**Failure Points:**
- Plan limit check fails (403 Forbidden)
- Insufficient historical data (< 2 months)
- Division by zero in trend calculation
- Invalid date parsing

**Handling:**
- Plan check happens first (early failure)
- Insufficient data returns empty forecasts or uses fallback
- Trend calculation checks `denominator == 0` before division
- Date parsing uses ISO format (YYYY-MM-DD)

**Evidence:**
- `backend/app/services/forecast_service.py:141` - `generate_advanced()` method
- `backend/app/services/forecast_service.py:51` - EWMA calculation
- `backend/app/services/forecast_service.py:75` - Trend detection
- `backend/app/services/forecast_service.py:104` - Seasonality calculation

## Flow 4: Transaction Creation with Validation

**Trigger:** `POST /api/v1/transactions` with `TransactionCreate` payload

**Steps:**
1. Router extracts `company_id` from JWT token
2. `TransactionService.create()` validates:
   - Category exists (queries `categories` table)
   - Account exists (queries `accounts` table)
   - Amount > 0
3. `TransactionRepository.create()` inserts with `company_id`
4. RLS policy enforces `company_id` matches user's company
5. Returns `TransactionResponse`

**Failure Points:**
- Category not found (404)
- Account not found (404)
- Amount <= 0 (400)
- RLS policy violation (database-level rejection)
- Foreign key constraint violation

**Handling:**
- Validation happens in service layer before database write
- Foreign key constraints in database schema prevent orphaned records
- RLS policies prevent cross-tenant access
- Errors return appropriate HTTP status codes

**Evidence:**
- `backend/app/services/transaction_service.py:72` - Create method
- `backend/app/repositories/transaction_repository.py:98` - Repository insert
- `database/migrations/05_rls_policies.sql:152` - RLS policy

## Flow 5: JWT Authentication and Company Resolution

**Trigger:** Any authenticated endpoint (uses `Depends(get_current_company_id)`)

**Steps:**
1. Request includes `Authorization: Bearer <token>` header
2. `verify_token()` extracts JWT payload via base64 decoding
3. Validates token expiration (`exp` claim vs current UTC time)
4. Extracts `user_id` from `sub` claim
5. `get_current_company_id()` queries `company_users` table
6. Returns `company_id` for repository filtering

**Failure Points:**
- Missing or invalid token format
- Token expired
- Missing `sub` claim
- User has no associated company
- Database query failure

**Handling:**
- Invalid format returns 401 with "Invalid token format"
- Expired token returns 401 with "Token expired"
- Missing user_id returns 401
- No company returns 403 with "User has no associated company"
- Database errors return 500

**Note:** Current implementation decodes JWT payload for development context resolution. Signature verification required for production.

**Evidence:**
- `backend/app/core/security.py:16` - `verify_token()` function
- `backend/app/core/security.py:143` - `get_current_company_id()` function
- `backend/app/core/security.py:42` - Manual base64 decoding (no signature verification)
