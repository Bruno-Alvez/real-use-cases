# Security and Data Boundaries

## Authentication and Authorization

### Admin

**Authentication**: Supabase Auth for user authentication. Frontend connects directly to Supabase. JWT tokens for session management.

**Authorization**: RLS policies enforce department-based and role-based access. Backend API uses JWT validation (if implemented). Marketing can manage campaigns and vouchers. Secretaries can manage enrollments and documents.

**Evidence**: `database/supabase/supabase_schema_collaborators.sql` (RLS policies), `backend/src/config/supabase.ts`

### Portal

**Authentication**: Student authentication via Supabase Auth. Password hashing with PBKDF2 (compatible with admin system). JWT tokens for session management.

**Authorization**: Students can only access their own enrollment data. No cross-student data access. RLS policies enforce student data isolation.

**Evidence**: `backend/src/services/enrollmentService.ts`, `database/schema.sql` (password_hash column)

### Agent

**Authentication**: No user authentication. Webhook endpoint validates Z-API payload structure but does not verify webhook signature.

**Authorization**: Service role key used for database writes (bypasses RLS). Anon key used for reads. Agent can read/write any enrollment data. No user context.

**Risk**: Webhook endpoint does not verify Z-API signature. Malicious requests could enqueue jobs if webhook URL is exposed. Service role key bypasses RLS, allowing agent to access any data. This is acceptable only if the webhook endpoint is treated as a private integration boundary and protected by network controls plus signature verification (recommended next step).

**Evidence**: `src/tools/supabase.ts` (service role key usage), `src/index.ts` (webhook endpoint)

## Permission Boundaries

**Admin RLS Policies**: Users can only access data for their department. Marketing can manage campaigns and vouchers. Secretaries can manage enrollments and documents. Policies defined in `database/supabase/supabase_schema_collaborators.sql`.

**Portal**: Students can only access their own enrollment data. No cross-student data access. RLS policies enforce student data isolation.

**Agent**: Service role key bypasses RLS. Agent can read/write any enrollment data. No user context. No permission checks.

## Sensitive Data Handling

**Log Sanitization**: Agent sanitizes CPF, phone, and email in logs (last digits masked). Portal and Admin do not sanitize logs (sensitive data may appear in error messages).

**Password Storage**: Passwords hashed with PBKDF2 (SHA-256, 100k iterations, 16-byte salt). Compatible across Portal and Admin systems.

**CPF Validation**: Format validation (11 digits) and checksum validation (Brazilian algorithm) in Agent and Portal.

**Evidence**: `src/utils/sanitizers.ts`, `backend/src/services/enrollmentService.ts`

## Legacy SQL Server Boundary

**Sync Approach**: Portal syncs enrollment data to SQL Server. Connection credentials stored in environment variables. Failures logged, no retry mechanism.

**Secret Handling**: SQL Server credentials in environment variables. No secrets in code or logs. Connection string not exposed in error messages.

**Data Scope**: Only enrollment and student data synced. Payment data may be included if configured. No reverse sync from SQL Server.

**Implementation Status**: Planned (sync function not found in portal codebase).

## Recommended Next Steps

**Webhook Signature Verification**: Implement Z-API webhook signature verification in Agent webhook endpoint. Validate HMAC signature before processing requests.

**Service Role Key Restriction**: Create dedicated database user for Agent with minimal required permissions. Use RLS policies even for service role operations where possible.

**Log Sanitization**: Extend sanitization to Portal and Admin components. Mask CPF, phone, and email in all error logs.

**Correlation IDs**: Add correlation IDs to all requests for tracing. Include correlation ID in logs and error messages.

**Secrets Management**: Use secrets management service (AWS Secrets Manager, HashiCorp Vault) instead of environment variables for SQL Server credentials.

**Evidence**: `src/index.ts` (webhook endpoint), `src/tools/supabase.ts` (service role key usage), `backend/src/middleware/errorHandler.ts` (error logging)
