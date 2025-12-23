# Architecture

## Components

### API Server
Express.js application serving REST endpoints. Handles HTTP requests, applies middleware (rate limiting, logging, error handling), and routes to controllers.

Evidence:
- `backend/src/server.ts` - Server entry point
- `backend/src/app.ts` - Express app configuration
- `backend/src/routes/index.ts` - Route definitions

### Controllers
HTTP request handlers that extract request data, call services, and format responses. Use asyncHandler wrapper for error propagation.

Evidence:
- `backend/src/controllers/VoucherController.ts` - Voucher endpoints
- `backend/src/controllers/CampaignController.ts` - Campaign endpoints
- `backend/src/middlewares/errorHandler.ts` - asyncHandler implementation

### Services
Business logic layer. Validates business rules, orchestrates repository calls, generates unique codes, and enforces constraints (e.g., preventing edits to used vouchers).

Evidence:
- `backend/src/services/VoucherService.ts` - Voucher business logic
- `backend/src/services/CampaignService.ts` - Campaign business logic

### Repositories
Data access layer. Abstracts Supabase client calls, maps database records to domain objects, and handles query construction.

Evidence:
- `backend/src/repositories/VoucherRepository.ts` - Voucher data access
- `backend/src/repositories/CampaignRepository.ts` - Campaign data access
- `backend/src/config/supabase.ts` - Supabase client initialization

### Database (PostgreSQL via Supabase)
Stores all entities. Includes triggers for automatic status updates, functions for business logic validation, and RLS policies for access control.

Evidence:
- `database/supabase/supabase_schema_vouchers_campaigns_final.sql` - Schema, triggers, functions
- `database/supabase/supabase_schema_collaborators.sql` - User and permission tables

### Frontend Application
React SPA that communicates with backend API and directly with Supabase for some operations. Manages UI state and user interactions.

Evidence:
- `frontend/src/App.tsx` - Main application component
- `frontend/src/services/supabase/campaigns.ts` - Frontend campaign service
- `frontend/src/components/Voucher/CampaignForm.tsx` - Campaign management UI

### Scheduled Jobs
SQL scripts executed daily via cron to update voucher and campaign statuses based on expiration dates.

Evidence:
- `database/supabase/supabase_cron_daily_updates.sql` - Daily maintenance script

## Data Boundaries

No multi-tenancy. Single database instance serves one institution. Row Level Security (RLS) policies control access based on user roles and departments, but all data shares the same schema.

Evidence:
- `database/supabase/supabase_schema_collaborators.sql` lines 383-413 - RLS policies
- `database/fixes/fix_rls_policies.sql` - RLS policy corrections

## Async Processing

No message queues or background workers. All processing is synchronous within HTTP request handlers. The only asynchronous operation is the daily SQL cron job that runs outside the application process.

Scheduled operations:
- Daily voucher expiration updates
- Daily campaign status transitions (agendada → ativa, ativa → finalizada)

Evidence:
- `database/supabase/supabase_cron_daily_updates.sql` - Cron job implementation
- No queue or worker code found in repository

## Storage

### Core Tables

**vouchers**: Individual discount vouchers
- Fields: codigo (unique), cpf_aluno, tipo_desconto, porcentagem_desconto/valor_desconto_fixo, validade_dias, data_expiracao, status
- Indexes: codigo, cpf_aluno, status, data_expiracao

**campaigns**: Promotional campaigns
- Fields: nome, codigo_promocional (unique), tipo_desconto, data_inicio, data_fim, limite_por_aluno, limite_total, utilizados, status
- Indexes: codigo_promocional, status, data_inicio, data_fim

**campaign_courses**: Many-to-many relationship between campaigns and eligible courses
- Fields: campaign_id, course_id
- Unique constraint on (campaign_id, course_id)

**campaign_shifts**: Many-to-many relationship between campaigns and eligible shifts
- Fields: campaign_id, shift (CHECK constraint: 'MANHA', 'TARDE', 'NOITE', 'SABADOS')
- Unique constraint on (campaign_id, shift)

**campaign_usage**: Records of campaign applications to enrollments
- Fields: campaign_id, student_id, enrollment_id, codigo_promocional_usado, valor_original, valor_desconto_aplicado, valor_final
- Unique constraint on (campaign_id, student_id, enrollment_id)

**voucher_usage**: Records of voucher applications to enrollments
- Fields: voucher_id, enrollment_id, student_id, valor_original, valor_desconto_aplicado, valor_final
- Unique constraint on (voucher_id, enrollment_id)

**collaborators**: System users
- Fields: id, name, email (case-insensitive), password_hash, department_id

**departments**: Organizational units
- Fields: id, name, description

**audit_logs**: Audit trail for all operations
- Fields: collaborator_id, action, resource_type, resource_id, old_values (JSONB), new_values (JSONB)

Evidence:
- `database/supabase/supabase_schema_vouchers_campaigns_final.sql` lines 26-230 - Table definitions
- `database/supabase/supabase_schema_collaborators.sql` lines 15-91 - User tables
