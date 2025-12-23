# Key Flows

## Flow 1: WhatsApp Agent Enrollment

**Trigger**: Z-API webhook POST `/webhook/whatsapp` receives message.

**Steps**:
1. Webhook validates payload with Zod, ignores bot messages and empty text
2. Enqueues job in BullMQ with `jobId: {phone}-{timestamp}`, responds 200
3. Worker processes job: loads conversation history from Redis
4. Sends prompt to OpenAI with context, LLM produces a response and structured fields under a constrained schema; the state machine drives control flow
5. If data complete: validates CPF format and checksum
6. Queries ViaCEP for address (retry 2x with backoff: 1s, 2s)
7. Creates student and enrollment in Supabase
8. Generates protocol and password
9. Sends credentials via WhatsApp
10. Saves message to history

**Failure Points**: Invalid payload (400), Redis unavailable (enqueue fails), OpenAI timeout (30s) or API error, ViaCEP unavailable (fallback to manual address), Supabase write failure, Z-API send failure.

**Handling**: Zod validation returns structured error. BullMQ retries failed jobs (2 attempts, 5s backoff). OpenAI timeout falls back to state machine mode. ViaCEP retry with progressive backoff, manual address if fails. Supabase errors logged, user receives friendly message. Z-API 20s timeout, error logged, no retry.

**Evidence**: `deep-dives/agent/key-flows.md`, `src/index.ts`, `src/queue.ts`, `src/agents/conversationalAgent.ts`

## Flow 2: Portal Enrollment with Documents and Payment

**Trigger**: Student submits enrollment form and clicks "Finalizar Pagamento".

**Steps**:
1. Frontend calls `POST /api/checkout/pix` with enrollment data
2. Controller validates required fields
3. Service calculates dynamic pricing: gets timezone (America/Fortaleza), fetches course_pricing, checks pre-enrollment window, determines pricing window (D1-5, D6-10, D11-20, NOMINAL)
4. Applies voucher discount if code provided: queries vouchers table (active, not expired, not used), calculates discount
5. Validates student address fields
6. Truncates name to 30 characters
7. HTTP POST to Asaas API creates checkout session
8. Saves session to database
9. Updates enrollment status to "Aguardando Pagamento"
10. Returns checkout URL
11. On payment completion (polled), enrollment status updated to "approved"

**Failure Points**: Pricing calculation failure, missing address fields, Asaas API error, database save failure after Asaas success, voucher validation failure.

**Handling**: Pricing errors return 400 with message. Missing address fields: explicit error listing fields. Asaas errors: response parsed, message returned. Database save errors: logged, inconsistency risk. Voucher errors: silently ignored, checkout proceeds. No retry on Asaas failures. No idempotency key.

**Evidence**: `deep-dives/portal/key-flows.md`, `backend/src/services/checkoutService.ts`, `backend/src/services/pricingService.ts`

## Flow 3: Admin Operational Workflow

**Trigger**: Admin validates documents, manages enrollment pipeline, or creates vouchers/campaigns.

**Steps**:
1. Admin views enrollment in Kanban board
2. Reviews uploaded documents in Supabase Storage
3. Approves or rejects documents via API endpoint
4. Database trigger checks if all required documents approved
5. If approved: trigger automatically updates enrollment status
6. Admin creates voucher via `POST /api/v1/vouchers`: service validates CPF format and discount values, generates unique code (retry 10 attempts, 100ms delay), calculates expiration date, repository inserts record, database trigger populates student name from CPF
7. Admin creates campaign via `POST /api/v1/campaigns`: service validates date ranges and limits, repository inserts campaign with course/shift associations, database trigger updates utilizados count on usage
8. RLS policies enforce department-based access: marketing can manage campaigns/vouchers, secretaries can manage enrollments/documents

**Failure Points**: Invalid CPF format, code generation collision, database constraint violation, campaign has usage but admin tries to change code, RLS policy blocks access.

**Handling**: Validation errors return 400. UNIQUE constraint prevents duplicate codes. Business rule violations return 400 with message. RLS policies block unauthorized access at database level.

**Evidence**: `deep-dives/admin/key-flows.md`, `backend/src/services/VoucherService.ts`, `backend/src/services/CampaignService.ts`, `database/supabase/supabase_schema_collaborators.sql`

## Flow 4: Legacy SQL Server Sync

**Trigger**: Portal creates enrollment or updates enrollment status.

**Steps**:
1. After enrollment creation in Supabase, portal service calls sync function (Planned: not implemented)
2. Function builds SQL Server payload from enrollment and student data
3. HTTP POST to SQL Server API endpoint or direct database connection
4. Sync includes: student data (CPF, name, email, address), enrollment data (protocol, course, modality, shift, status), payment data if available
5. On success, sync status logged
6. On failure, error logged, enrollment continues in Supabase system

**Failure Points**: SQL Server unavailable, network timeout, authentication failure, payload validation error, partial sync success.

**Handling**: Failures logged for manual reconciliation. No retry mechanism implemented. Enrollment proceeds in Supabase regardless of sync status. Manual reconciliation required for failed syncs.

**Evidence**: Legacy sync mentioned as integration requirement. Implementation status: Planned (sync function not found in portal codebase)
