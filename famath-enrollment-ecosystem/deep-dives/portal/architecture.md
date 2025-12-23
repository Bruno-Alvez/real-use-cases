# Architecture

## Components

### API Server
Express.js application handling HTTP requests. Entry point: `backend/src/server.ts`. Application setup: `backend/src/app.ts`.

**Evidence:**
- `backend/src/server.ts` - Server initialization, graceful shutdown, health checks
- `backend/src/app.ts` - Express app configuration, middleware setup, route registration

### Services Layer
Business logic organized by domain:
- `checkoutService.ts` - Asaas checkout session creation and management
- `pricingService.ts` - Dynamic pricing calculation based on dates, campaigns, pre-enrollment windows
- `paymentService.ts` - Legacy payment creation (direct Asaas charges)
- `enrollmentService.ts` - Enrollment lifecycle management
- `timelineService.ts` - Audit trail event creation

**Evidence:**
- `backend/src/services/checkoutService.ts` - Lines 193-917
- `backend/src/services/pricingService.ts` - Lines 44-352
- `backend/src/services/paymentService.ts` - Lines 30-257

### Repository Layer
Data access abstraction over Supabase:
- `enrollmentRepository.ts` - Enrollment CRUD operations
- `studentRepository.ts` - Student data access
- `paymentRepository.ts` - Payment records
- `courseRepository.ts` - Course information
- `documentRepository.ts` - Document metadata

**Evidence:**
- `backend/src/repositories/` directory

### Controllers
HTTP request handlers that validate input and call services:
- `checkoutController.ts` - Checkout endpoints (create, PIX, credit card)
- `pricingController.ts` - Pricing calculation endpoint
- `enrollmentController.ts` - Enrollment management endpoints
- `paymentController.ts` - Payment status endpoints
- `courseController.ts` - Course listing endpoints

**Evidence:**
- `backend/src/controllers/checkoutController.ts` - Lines 9-337
- `backend/src/routes/checkout.routes.ts` - Route definitions

### Database
PostgreSQL via Supabase with:
- Tables: `students`, `enrollments`, `documents`, `payments`, `enrollment_timeline`, `course_pricing`, `vouchers`, `checkout_sessions`, `campaigns`
- Triggers: `auto_update_enrollment_status`, `set_enrollment_protocol`
- Functions: `generate_protocol_number()`, `get_documents_with_resubmission_info()`

**Evidence:**
- `database/schema.sql` - Complete schema definition
- `database/triggers/auto_update_enrollment_status.sql` - Status update trigger
- `backend/src/config/database.ts` - Supabase client configuration

### External Services
- **Asaas API**: Payment gateway integration
  - Base URL: `https://www.asaas.com/api/v3`
  - Endpoints: `/checkouts`, `/customers`, `/payments`
  - Authentication: API key in `access_token` header

**Evidence:**
- `backend/src/config/payment.ts` - Asaas configuration and types
- `backend/src/services/checkoutService.ts` - Lines 326-330, 683-687, 865-869

## Data Boundaries

No multi-tenancy. Single institution (FAMATH). Data isolation by:
- `enrollment_id` foreign keys for related records
- `student_id` for student-scoped queries
- `protocol_number` for enrollment lookup

**Evidence:**
- `database/schema.sql` - Foreign key constraints (lines 76-78, 112, 140, 162)

## Async Processing

No background job queue. Synchronous processing only:
- Checkout session creation is synchronous HTTP call to Asaas
- Payment status checked via polling (no webhook handler implemented)
- Database triggers handle automatic status updates

**Evidence:**
- `backend/src/services/checkoutService.ts` - Synchronous `fetch()` calls (lines 326-330)
- `database/triggers/auto_update_enrollment_status.sql` - Database-side async via triggers

## Storage

### Database Tables

**Core entities:**
- `students` - Student personal data, authentication credentials
- `enrollments` - Enrollment records with protocol numbers, status, current step
- `documents` - Document metadata (file URLs point to Supabase Storage)
- `payments` - Payment records linked to enrollments
- `enrollment_timeline` - Event log for audit trail

**Configuration:**
- `courses` - Course definitions with available shifts
- `entry_modalities` - Entry method definitions (ENEM, Vestibular, etc.) with required documents
- `course_pricing` - Pricing rules per course/shift with validity dates
- `campaigns` - Active discount campaigns
- `vouchers` - Discount vouchers with expiration and usage tracking
- `checkout_sessions` - Asaas checkout session metadata

**Evidence:**
- `database/schema.sql` - Complete table definitions (lines 11-180)

### File Storage
Documents stored in Supabase Storage. Database stores metadata only (file_url, file_name, mime_type).

**Evidence:**
- `database/schema.sql` - `documents` table (line 117: `file_url TEXT NOT NULL`)

## Component Evidence

| Component | Files |
|-----------|-------|
| API Server | `backend/src/server.ts`, `backend/src/app.ts` |
| Checkout Service | `backend/src/services/checkoutService.ts` |
| Pricing Engine | `backend/src/services/pricingService.ts` |
| Database Config | `backend/src/config/database.ts` |
| Asaas Integration | `backend/src/config/payment.ts`, `backend/src/services/checkoutService.ts` |
| Security Middleware | `backend/src/config/security.ts`, `backend/src/middleware/errorHandler.ts` |
| Database Schema | `database/schema.sql` |
| Triggers | `database/triggers/auto_update_enrollment_status.sql` |
