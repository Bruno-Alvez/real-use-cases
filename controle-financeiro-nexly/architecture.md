# Architecture

## Components

### API Layer

FastAPI application with REST endpoints. Handles HTTP requests, validates JWT tokens, extracts company context, and returns JSON responses.

**Files:**
- `backend/app/main.py` - FastAPI app initialization, CORS middleware, exception handlers
- `backend/app/api/v1/__init__.py` - Router aggregation
- `backend/app/api/v1/routers/*.py` - Individual endpoint routers

**Key routers:**
- `transactions.py` - CRUD for financial transactions
- `recurring_rules.py` - Recurring rule management and processing
- `import_csv.py` - CSV file upload and processing
- `forecasts.py` - Forecast generation and retrieval
- `dashboard.py` - Aggregated KPI endpoints
- `auth.py` - Authentication endpoints

### Service Layer

Business logic and orchestration. Services coordinate between repositories and enforce business rules.

**Files:**
- `backend/app/services/transaction_service.py` - Transaction validation and creation
- `backend/app/services/recurring_service.py` - Recurring rule processing, versioning, deduplication
- `backend/app/services/csv_import_service.py` - CSV parsing and transaction creation
- `backend/app/services/forecast_service.py` - Forecast algorithms (EWMA, trend, seasonality)
- `backend/app/services/plan_limits_service.py` - Plan-based feature gating
- `backend/app/services/storage_service.py` - File upload wrapper for Supabase Storage
- `backend/app/services/kpi_service.py` - Dashboard aggregations

### Repository Layer

Data access abstraction. All queries filter by `company_id` for multi-tenant isolation. Uses Supabase Python client.

**Files:**
- `backend/app/repositories/transaction_repository.py` - Transaction CRUD
- `backend/app/repositories/recurring_repository.py` - Recurring rule CRUD and versioning
- `backend/app/repositories/csv_imports.py` - CSV import tracking
- `backend/app/repositories/category_repository.py` - Category management
- `backend/app/repositories/account_repository.py` - Account management
- `backend/app/repositories/forecast_repository.py` - Forecast persistence

### Database

PostgreSQL via Supabase with Row Level Security (RLS) enabled on all tenant tables.

**Schema files:**
- `database/init/01_schema.sql` - Base tables (categories, accounts, transactions, recurring_rules, forecasts)
- `database/migrations/01_companies_and_users.sql` - Multi-tenant foundation
- `database/migrations/02_plans_and_subscriptions.sql` - Subscription system
- `database/migrations/03_migrate_tables_to_company_id.sql` - Migration to company_id
- `database/migrations/05_rls_policies.sql` - RLS policy definitions

**Key tables:**
- `companies` - Tenant organizations
- `company_users` - User-company associations
- `transactions` - Financial transactions
- `recurring_rules` - Recurring transaction templates
- `forecasts` - Revenue/expense forecasts
- `csv_imports` - Import audit log
- `subscriptions` - Active subscriptions
- `plans` - Plan definitions with JSONB features

### External Services

**Supabase Storage:**
- Buckets: `avatars`, `csv-imports`, `documents`
- Path structure: `{company_id}/{filename}` or `{user_id}/{filename}`
- Upload via Supabase Python client or direct REST API

**Evidence:**
- `backend/app/services/storage_service.py` - Storage wrapper implementation
- Uses `client.storage.from_(BUCKET).upload()` and direct REST API for avatars

**Supabase Auth:**
- JWT token generation and validation
- User management (handled by Supabase, not backend)

**Evidence:**
- `backend/app/core/security.py` - JWT token decoding and validation
- Manual base64 decoding of JWT payload (no signature verification in current implementation)

## Data Boundaries

### Multi-Tenancy

Isolation via `company_id` column on all tenant tables plus RLS policies.

**Flow:**
1. Request includes JWT token
2. Backend extracts `user_id` from token
3. `get_current_company_id()` queries `company_users` table
4. All repository queries filter by `company_id`
5. RLS policies enforce additional isolation at database level

**Evidence:**
- `backend/app/core/security.py:143` - `get_current_company_id()` function
- `backend/app/repositories/transaction_repository.py:17` - Repository initialized with `company_id`
- `database/migrations/05_rls_policies.sql` - RLS policies using `auth.uid()` and `company_users` join

**RLS Policy Example:**
```sql
CREATE POLICY "Users can view own transactions"
    ON transactions FOR SELECT
    USING (
        company_id IN (
            SELECT company_id FROM company_users 
            WHERE user_id = auth.uid()
        )
    );
```

## Async Processing

No async job queues or workers. All processing is synchronous within HTTP request handlers.

**Recurring Rule Processing:**
- Triggered via `POST /recurring-rules/process-all` endpoint
- Processes all active rules in a single request
- Creates transactions synchronously
- No background jobs or scheduling

**Evidence:**
- `backend/app/api/v1/routers/recurring_rules.py:101` - `process_all_recurring_rules()` endpoint
- `backend/app/services/recurring_service.py:337` - Synchronous processing loop

**CSV Import:**
- Upload and processing happen in the same HTTP request
- File read, parse, validate, and transaction creation all synchronous
- Large files could block the request

**Evidence:**
- `backend/app/api/v1/routers/import_csv.py:15` - CSV import endpoint
- `backend/app/services/csv_import_service.py:41` - Synchronous processing

## Storage

**Database Tables:**
- `transactions` - Core financial data (date, amount, type, category_id, account_id, company_id)
- `recurring_rules` - Template rules with versioning (original_rule_id, effective_from_date, superseded_by)
- `forecasts` - Monthly forecasts with optional advanced algorithm fields (ewma_revenue, trend_revenue, seasonality_factor)
- `csv_imports` - Audit log (file_name, file_path, total_records, success_count, failed_count, status)

**Object Storage:**
- Supabase Storage buckets for file uploads
- CSV files stored in `csv-imports/{company_id}/{filename}`
- Avatars in `avatars/{user_id}/{filename}`
- Documents in `documents/{company_id}/{filename}`

**Evidence:**
- `backend/app/services/storage_service.py:17-20` - Bucket constants
- `backend/app/services/storage_service.py:249` - CSV upload path structure

## Indexes

Indexes on frequently queried columns for performance.

**Evidence:**
- `database/migrations/04_update_indexes_for_company_id.sql` - Composite indexes on `(company_id, date)`, `(company_id, status)`
- `database/init/01_schema.sql:93-102` - Base indexes on `user_id`, `date`, foreign keys
