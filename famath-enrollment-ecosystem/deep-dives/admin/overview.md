# System Overview

## What the System Does

The FAMATH Enrollment Management System manages student enrollment workflows for a higher education institution. It handles candidate registration, document management, voucher and promotional campaign administration, and enrollment tracking through a Kanban-style interface. The system enforces business rules around discount eligibility, campaign limits, and voucher validity periods.

## Integrations

- **Supabase (PostgreSQL)**: Primary database for all entities (students, enrollments, vouchers, campaigns, collaborators, documents)
- **Supabase Auth**: User authentication and session management (via frontend direct connection)
- **Supabase Storage**: Document file storage (referenced in frontend services)
- **PostgreSQL Functions**: Database-level business logic for discount calculations, validation, and status updates

Evidence:
- `backend/src/config/supabase.ts` - Supabase client configuration
- `frontend/src/lib/supabase.ts` - Frontend Supabase client
- `database/supabase/supabase_schema_vouchers_campaigns_final.sql` - Database functions and triggers

## Technology Stack

**Backend:**
- Node.js 18+ with Express 4.18.2
- TypeScript 5.3.3
- Winston 3.11.0 for logging
- express-rate-limit 7.1.5 for rate limiting
- express-validator 7.0.1 for input validation
- Zod 3.22.4 for schema validation

**Frontend:**
- React 18.3.1
- TypeScript 5.5.3
- Vite 5.4.11
- Tailwind CSS 3.4.1
- Supabase JS client 2.57.4

**Database:**
- PostgreSQL (via Supabase)
- Row Level Security (RLS) policies
- Database triggers for automatic status updates
- Scheduled functions for daily maintenance

**Infrastructure:**
- Docker multi-stage builds (`backend/Dockerfile`)
- Monorepo structure with npm workspaces

Evidence:
- `package.json` (root, backend, frontend)
- `backend/Dockerfile`
- `backend/src/app.ts`, `backend/src/server.ts`

## Implemented

- REST API for vouchers (CRUD operations)
- REST API for campaigns (CRUD operations)
- Business logic layer (Services) with validation rules
- Repository pattern for database access
- Rate limiting middleware
- Error handling middleware with structured logging
- Request logging middleware
- Database triggers for automatic status updates
- Daily cron job SQL script for voucher/campaign status maintenance
- Frontend campaign and voucher management UI
- Campaign editing with course and shift association
- Voucher editing with campaign rule validation
- Database constraints and CHECK constraints for data integrity

Evidence:
- `backend/src/controllers/VoucherController.ts`
- `backend/src/controllers/CampaignController.ts`
- `backend/src/services/VoucherService.ts`
- `backend/src/services/CampaignService.ts`
- `backend/src/repositories/VoucherRepository.ts`
- `backend/src/repositories/CampaignRepository.ts`
- `database/supabase/supabase_cron_daily_updates.sql`

## Planned

- Additional API endpoints for candidates, documents, analytics (commented in `backend/src/routes/index.ts`)
- Authentication middleware integration (referenced but not fully implemented)
- Comprehensive test coverage (Jest configured but no tests present)
- Dead-letter queue or reprocessing strategy for failed operations
- Metrics collection and observability dashboards
- Retry mechanisms with exponential backoff
- Idempotency keys for critical operations

Evidence:
- `backend/src/routes/index.ts` lines 25-29 (commented routes)
- `backend/package.json` includes Jest but no test files found
- No retry logic in repository or service layers
- No idempotency handling in controllers
