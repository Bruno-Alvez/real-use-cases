# Overview

The FAMATH Enrollment Ecosystem manages the complete student enrollment lifecycle from initial candidate contact through payment confirmation. The system addresses the need to automate enrollment workflows, reduce manual processing, and provide multiple enrollment channels while maintaining data consistency across operational boundaries.

The solution consists of three integrated components sharing a unified Supabase PostgreSQL database. The Admin Backoffice provides internal tools for secretaries and marketing staff to manage candidate records, validate documents, administer vouchers and campaigns, and track enrollment status via Kanban interface with granular department-based permissions. The Student Portal is a public-facing enrollment system where candidates register, select courses, upload required documents, calculate dynamic pricing, apply vouchers, complete payment via Asaas (PIX or credit card), and view enrollment timeline. The WhatsApp Agent automates enrollment through conversational interactions, receiving messages via Z-API webhooks, processing asynchronously via BullMQ, using LLM for natural conversation, validating data, creating enrollments in the shared database, and sending credentials via WhatsApp.

A legacy boundary exists: enrollment data must sync to a legacy SQL Server system. The Portal component is responsible for this sync, but the implementation is planned and not yet present in the codebase. Failures are logged for manual reconciliation, and enrollment proceeds in Supabase regardless of sync status.

## Implemented vs Planned

### Portal

**Implemented**: Enrollment creation, document upload to Supabase Storage, dynamic pricing engine with timezone awareness, voucher and campaign discount application, Asaas checkout integration (PIX and credit card), payment status polling, enrollment timeline events, database triggers for automatic status updates.

**Planned**: Legacy SQL Server sync implementation, Asaas webhook endpoint for payment confirmations, retry logic with exponential backoff for Asaas API calls, idempotency keys for checkout creation, structured logging with correlation IDs.

### Admin

**Implemented**: Voucher creation and management, campaign creation with course/shift associations, enrollment Kanban board, document validation workflow, granular permissions via RLS policies, department-based access control, daily cron job for voucher/campaign expiration updates.

**Planned**: Analytics dashboard, advanced reporting, scheduled SQL jobs for bulk operations, metrics collection.

### Agent

**Implemented**: Z-API webhook reception, async job processing via BullMQ with retry (2 attempts, 5s backoff), Redis state management, LLM conversational enrollment flow, CPF and address validation, ViaCEP integration with retry, enrollment creation in shared database, credential generation and delivery via WhatsApp, log sanitization (CPF, phone, email).

**Planned**: Webhook signature verification, dead-letter queue for failed jobs, structured logging with correlation IDs, metrics collection, distributed tracing.

## Evidence

**Admin**: `deep-dives/admin/architecture.md`, `deep-dives/admin/key-flows.md`, `backend/src/services/VoucherService.ts`, `backend/src/services/CampaignService.ts`, `database/supabase/supabase_schema_vouchers_campaigns_final.sql`

**Portal**: `deep-dives/portal/architecture.md`, `deep-dives/portal/key-flows.md`, `backend/src/services/checkoutService.ts`, `backend/src/services/pricingService.ts`, `database/triggers/auto_update_enrollment_status.sql`

**Agent**: `deep-dives/agent/architecture.md`, `deep-dives/agent/key-flows.md`, `src/agents/conversationalAgent.ts`, `src/queue.ts`, `src/services/openai.ts`
