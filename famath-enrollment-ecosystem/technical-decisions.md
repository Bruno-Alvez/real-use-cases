# Technical Decisions

## Decision 1: Component Separation

**Context**: Need to support three distinct user interfaces (admin backoffice, student portal, WhatsApp agent) with different operational requirements and deployment needs.

**Decision**: Three separate components (Admin, Portal, Agent) sharing a common Supabase database.

**Alternatives Considered**: Single monolithic application, microservices with API gateway, shared library with different entry points.

**Trade-offs**: Separation allows independent deployment and scaling, but requires coordination on database schema changes and shared business logic. Business logic duplicated in some areas (voucher validation, pricing calculation).

**Consequence**: Database schema changes require coordination across components. Shared database becomes single point of failure. Independent deployment enables faster iteration per component.

**Evidence**: `deep-dives/admin/architecture.md`, `deep-dives/portal/architecture.md`, `deep-dives/agent/architecture.md`

## Decision 2: Enrollment State Model

**Context**: Enrollment status must reflect document approval state and payment status. Status updates happen from multiple sources (document approval, payment confirmation, manual admin updates).

**Decision**: Database triggers automatically update enrollment status when all required documents are approved. Status field stored in enrollments table.

**Alternatives Considered**: Application-level status checks, scheduled jobs, event-driven architecture with message queue.

**Trade-offs**: Triggers ensure consistency at database level but move business logic to database, making testing and debugging harder.

**Consequence**: Status updates happen automatically without application code, but trigger logic is harder to modify and test. No way to disable automatic updates for specific cases.

**Evidence**: `database/triggers/auto_update_enrollment_status.sql`, `deep-dives/portal/technical-decisions.md`

## Decision 3: Payment Integration Strategy

**Context**: Payment processing requires integration with Asaas API. Need to handle checkout session creation and payment confirmation.

**Decision**: Portal creates checkout sessions synchronously. Payment status checked via polling. No webhook handler for Asaas payment confirmations.

**Alternatives Considered**: Webhook endpoint for Asaas callbacks, event-driven payment processing, database polling with scheduled jobs.

**Trade-offs**: Polling is simpler to implement but delays payment confirmation. Webhooks provide immediate updates but require reliable endpoint and signature validation.

**Consequence**: Payment status updates delayed until next poll. No automatic enrollment approval on payment. Manual intervention may be required for failed syncs.

**Evidence**: `deep-dives/portal/overview.md` (Planned section), `backend/src/services/checkoutService.ts`

## Decision 4: Legacy SQL Server Sync Strategy

**Context**: Enrollment data must sync to legacy SQL Server system for downstream processes. Sync must not block enrollment flow.

**Decision**: Portal syncs enrollment data asynchronously. Failures logged for manual reconciliation. No retry mechanism.

**Alternatives Considered**: Synchronous sync (block until success), retry with backoff, message queue for reliable delivery, no sync (deprecate legacy system).

**Trade-offs**: Fire-and-forget is simplest but risks data inconsistency. Retry adds reliability but requires queue infrastructure.

**Consequence**: Enrollment proceeds in Supabase even if sync fails. Manual reconciliation required for failed syncs. No automatic retry or dead-letter queue. Implementation status: Planned (sync function not found in portal codebase).

**Evidence**: Legacy sync mentioned as integration requirement. `deep-dives/portal/architecture.md`

## Decision 5: Permission Model

**Context**: Admin component needs granular access control by department and role. Marketing manages campaigns/vouchers, secretaries manage enrollments/documents.

**Decision**: Row Level Security policies enforce department-based and role-based access control at database level.

**Alternatives Considered**: Application-level permission checks only, role-based access without department isolation, coarse-grained permissions.

**Trade-offs**: RLS enforces permissions at database level (defense in depth) but policies must be maintained. Application-level checks are easier to test but can be bypassed.

**Consequence**: Permissions enforced even if application code has bugs, but RLS policies are complex and harder to debug. RLS is the primary enforcement layer. Direct database access from the frontend is only safe under audited, least-privilege RLS policies; sensitive operations should remain behind backend endpoints.

**Evidence**: `database/supabase/supabase_schema_collaborators.sql` (RLS policies), `deep-dives/admin/architecture.md`

## Decision 6: WhatsApp Reliability Strategy

**Context**: WhatsApp Agent receives webhooks that may timeout. Message processing requires LLM calls and external API calls that may fail.

**Decision**: Agent uses BullMQ for async job processing. Webhook responds 200 immediately, job enqueued. Worker processes with retry (2 attempts, 5s exponential backoff). Redis stores conversation state.

**Alternatives Considered**: Synchronous webhook processing, different queue system (RabbitMQ, SQS), no retry mechanism.

**Trade-offs**: Async processing prevents webhook timeouts and enables retry, but adds complexity (Redis, worker process). Synchronous is simpler but blocks on external APIs.

**Consequence**: Agent can handle webhook timeouts gracefully. Jobs retry automatically. Conversation state persists in Redis. Portal and Admin have no retry logic.

**Evidence**: `deep-dives/agent/architecture.md`, `src/queue.ts`, `src/services/redis.ts`

## Decision 7: Data Validation Strategy

**Context**: CPF, addresses, and enrollment data must be validated before database writes. Validation occurs in multiple components.

**Decision**: Portal validates CPF format and checksum, address fields required. Agent validates CPF format and checksum, queries ViaCEP for address validation with retry. Database constraints enforce referential integrity.

**Alternatives Considered**: Single validation service, database-only validation, no validation.

**Trade-offs**: Component-level validation is simpler but duplicates logic. Centralized validation requires shared service.

**Consequence**: Validation logic duplicated across components. CPF validation consistent (format + checksum). Address validation differs (Portal requires fields, Agent queries ViaCEP).

**Evidence**: `deep-dives/portal/key-flows.md`, `deep-dives/agent/key-flows.md`, `src/tools/viacep.ts`

## Decision 8: Observability Approach

**Context**: Need to monitor system health, debug issues, and track business metrics.

**Decision**: Agent uses Pino structured logging with sanitization (CPF, phone, email masked). Portal uses Morgan for HTTP logs, console.error for errors. Admin uses Winston structured logging. No metrics collection, no distributed tracing, no correlation IDs.

**Alternatives Considered**: Unified logging framework, metrics collection (Prometheus), distributed tracing (OpenTelemetry), correlation IDs.

**Trade-offs**: Component-specific logging is simpler but inconsistent. Unified approach requires shared infrastructure.

**Consequence**: Logs available for debugging but format differs per component. No metrics for monitoring. No correlation IDs for request tracing. Sanitization only in Agent.

**Evidence**: `deep-dives/agent/reliability-and-ops.md`, `src/utils/sanitizers.ts`, `backend/src/middleware/errorHandler.ts`
