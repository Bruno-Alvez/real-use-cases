# Architecture

## Components and Boundaries

**Admin Backoffice**: Express.js API with React frontend. REST endpoints for vouchers, campaigns, enrollment management. Direct Supabase queries from frontend for some operations. Department-based permissions via RLS policies. Synchronous request handlers for all operations. No background job queues.

**Student Portal**: Express.js API with React frontend. Enrollment creation, document upload, pricing calculation, Asaas checkout integration. Synchronous payment processing. Legacy SQL Server sync planned (not implemented). No background job queues.

**WhatsApp Agent**: Fastify webhook server with BullMQ worker in same process. Receives Z-API webhooks, enqueues jobs, processes asynchronously. Redis for state and job queue. LLM integration for conversational mode. Async processing for all message handling.

**Shared Database**: PostgreSQL via Supabase. Tables: students, enrollments, documents, payments, vouchers, campaigns, enrollment_timeline, checkout_sessions. RLS policies enforce access control. Triggers for automatic status updates and protocol generation.

## Data Flow

**Enrollment Creation**: Portal or Agent creates enrollment in shared database. Database trigger generates protocol number. Timeline event created. Portal syncs to SQL Server (Planned: not implemented).

**Document Upload**: Portal uploads to Supabase Storage, creates document record. Database trigger checks if all required documents approved, updates enrollment status automatically.

**Payment Processing**: Portal creates Asaas checkout session synchronously, saves session record. Payment status checked via polling (no webhook handler). On payment confirmation, enrollment status updated.

**Voucher Application**: Portal queries vouchers table, validates expiration and usage, applies discount to checkout amount. Admin creates vouchers via API, database trigger populates student name from CPF.

**Campaign Management**: Admin creates campaigns with course/shift associations. Portal checks active campaigns during pricing calculation. Campaign usage tracked in campaign_usage table.

**WhatsApp Enrollment**: Agent receives message, enqueues job, processes asynchronously. LLM extracts data from conversation, validates CPF and address, creates student and enrollment in shared database, sends credentials via WhatsApp.

## Synchronous vs Asynchronous Processing

**Portal**: All operations synchronous. Enrollment creation, checkout session creation, document upload, pricing calculation block on external API calls. No retry logic. No background jobs.

**Admin**: All operations synchronous. Voucher creation, campaign management, enrollment updates, document validation execute within request handler. Daily cron job updates voucher/campaign expiration status (scheduled SQL job).

**Agent**: All message processing asynchronous. Webhook receives message, responds 200 immediately, enqueues job in BullMQ. Worker processes job asynchronously with retry (2 attempts, 5s backoff). Redis stores conversation state.

## Data Stores

**PostgreSQL (Supabase)**: Primary database. All components read/write. RLS policies isolate data by user role and department. Single institution, no multi-tenancy.

**Redis**: Agent only. Job queue (BullMQ), conversation state, temporary data, message history. 24h TTL for state, 48h for handoff.

**Supabase Storage**: Document files. Portal uploads, admin views. Path structure: `{enrollment_id}/{document_type}`.

**SQL Server**: Legacy system. Portal syncs enrollment data (Planned: not implemented). No read operations from ecosystem. Failures logged, no retry mechanism.

## Isolation Model

No multi-tenancy. Single institution. Data isolation via RLS policies based on user roles (secretary, marketing, admin) and departments. Agent uses service role key for writes (bypasses RLS), anon key for reads.

## Legacy SQL Server Sync Boundary

Portal component responsible for syncing enrollment data to SQL Server. Sync includes student data (CPF, name, email, address), enrollment data (protocol, course, modality, shift, status), payment data if available. Implementation status: Planned (sync function not found in portal codebase). Failure handling: errors logged for manual reconciliation. No retry mechanism. Enrollment proceeds in Supabase regardless of sync status.

## Evidence

**Admin**: `deep-dives/admin/architecture.md`, `backend/src/config/supabase.ts`, `database/supabase/supabase_schema_collaborators.sql`

**Portal**: `deep-dives/portal/architecture.md`, `backend/src/config/database.ts`, `database/schema.sql`

**Agent**: `deep-dives/agent/architecture.md`, `src/services/redis.ts`, `src/tools/supabase.ts`
