# Technical Decisions

## Decision 1: Layered Architecture (Controller → Service → Repository)

**Alternatives Considered:**
- Direct database access from controllers
- Fat models with business logic in database functions only
- GraphQL API instead of REST

**Why Chosen:**
Separation of concerns. Controllers handle HTTP, services contain business rules, repositories abstract data access. Makes testing easier and allows swapping data sources without changing business logic.

**Consequences:**
- More files and indirection for simple operations
- Business logic split between services and database functions (some validation in PostgreSQL, some in TypeScript)
- Future improvement: Consolidate validation in one layer (prefer services for testability)

**Evidence:**
- `backend/src/controllers/VoucherController.ts`
- `backend/src/services/VoucherService.ts`
- `backend/src/repositories/VoucherRepository.ts`

## Decision 2: Supabase Client for Database Access

**Alternatives Considered:**
- Raw PostgreSQL driver (pg)
- Prisma ORM
- TypeORM

**Why Chosen:**
Supabase provides built-in RLS, real-time subscriptions (not used yet), and simplified connection management. The client library handles connection pooling and provides a query builder API.

**Consequences:**
- Vendor lock-in to Supabase
- Limited transaction support (Supabase client doesn't expose explicit transactions)
- Future improvement: Abstract repository interface to allow swapping implementations

**Evidence:**
- `backend/src/config/supabase.ts`
- `backend/src/repositories/VoucherRepository.ts` - Uses supabase client throughout

## Decision 3: Database Triggers for Automatic Status Updates

**Alternatives Considered:**
- Application-level status checks on every read
- Scheduled background jobs in application code
- Event-driven updates via message queue

**Why Chosen:**
Triggers ensure status consistency at the database level. Voucher status updates to 'utilizado' when voucher_usage record is inserted, and campaign utilizados counter increments automatically. Reduces risk of application bugs causing inconsistent state.

**Consequences:**
- Business logic in two places (application and database) makes debugging harder
- Triggers are harder to test than application code
- Future improvement: Document all trigger behavior and add integration tests

**Evidence:**
- `database/supabase/supabase_schema_vouchers_campaigns_final.sql` lines 304-340

## Decision 4: No Database Transactions for Multi-Step Updates

**Alternatives Considered:**
- Wrapping campaign updates (campaign + courses + shifts) in a transaction
- Using Supabase RPC functions that execute in a single transaction
- Application-level compensation logic

**Why Chosen:**
Supabase JS client doesn't expose explicit transaction APIs easily. The current approach deletes and re-inserts related records, which is simpler to implement.

**Consequences:**
- Partial failures leave inconsistent state (e.g., campaign updated but courses not updated)
- No rollback on errors
- Future improvement: Use Supabase RPC functions or pg-promise for explicit transactions

**Evidence:**
- `backend/src/repositories/CampaignRepository.ts` lines 293-319 - Separate delete/insert operations without transaction

## Decision 5: Code Generation with Retry Loop

**Alternatives Considered:**
- UUID-based codes
- Sequential number generation with database sequence
- External code generation service

**Why Chosen:**
Human-readable codes (DESC20241234) are easier for users to reference. Retry loop with timestamp-based generation avoids collisions in low-concurrency scenarios.

**Consequences:**
- Under high concurrency, retry loop may fail after 10 attempts
- No distributed locking (multiple instances could generate same code)
- Future improvement: Use database sequence or distributed lock (Redis) for code generation

**Evidence:**
- `backend/src/services/VoucherService.ts` lines 253-271

## Decision 6: CHECK Constraints for Shift Values

**Alternatives Considered:**
- Enum type in PostgreSQL
- Application-level validation only
- Separate lookup table for valid shifts

**Why Chosen:**
CHECK constraint enforces data integrity at the database level, preventing invalid data even if application code has bugs. Simpler than a lookup table for a small, static set of values.

**Consequences:**
- Constraint values ('MANHA', 'TARDE', 'NOITE', 'SABADOS') don't match frontend labels ('Manhã', 'Tarde', 'Noite', 'Sábados', 'Noturno', 'Integral')
- Requires mapping layer in application code
- Future improvement: Standardize shift values across frontend and backend, or use enum type

**Evidence:**
- `database/supabase/supabase_schema_vouchers_campaigns_final.sql` line 147 - CHECK constraint
- `backend/src/repositories/CampaignRepository.ts` lines 469-479 - normalizeShift mapping
- `frontend/src/services/supabase/campaigns.ts` - Frontend shift mapping

## Decision 7: Winston for Logging

**Alternatives Considered:**
- console.log only
- Pino (faster JSON logger)
- Structured logging to external service (Datadog, CloudWatch)

**Why Chosen:**
Winston provides structured logging with multiple transports. Console output for development, JSON format for production parsing. Easy to add file or external service transports later.

**Consequences:**
- No log aggregation or search currently (logs only to console)
- No correlation IDs for request tracing
- Future improvement: Add file transport and integrate with log aggregation service

**Evidence:**
- `backend/src/utils/logger.ts`
- `backend/src/middlewares/requestLogger.ts` - Uses Winston logger

## Decision 8: Rate Limiting at API Level

**Alternatives Considered:**
- Rate limiting at reverse proxy (nginx)
- Per-user rate limits
- No rate limiting

**Why Chosen:**
express-rate-limit is simple to configure and provides basic protection against abuse. Global rate limit (100 requests per 15 minutes) is sufficient for initial deployment.

**Consequences:**
- Global limit doesn't distinguish between users (one abusive user can block others)
- No per-endpoint rate limits (except authLimiter and createLimiter)
- Future improvement: Implement per-user or per-IP rate limits with Redis backend

**Evidence:**
- `backend/src/middlewares/rateLimiter.ts`
- `backend/src/app.ts` line 34 - Applied to all API routes

## Decision 9: Monorepo with npm Workspaces

**Alternatives Considered:**
- Separate repositories for frontend and backend
- Single package.json with all dependencies
- Lerna or Nx for monorepo management

**Why Chosen:**
npm workspaces provide simple dependency management and allow sharing TypeScript types via symlink. No additional tooling required.

**Consequences:**
- Shared types require symlink setup (`backend/src/shared` → `../../shared`)
- Build processes are separate (no unified build pipeline)
- Future improvement: Consider Nx for better build orchestration and dependency graph

**Evidence:**
- `package.json` (root) - workspaces configuration
- `backend/src/shared` - Symlink to shared types

## Decision 10: No Idempotency Keys

**Alternatives Considered:**
- Idempotency keys in request headers
- Database table to track processed requests
- Idempotency via natural keys (e.g., codigo for vouchers)

**Why Chosen:**
Current operations are mostly idempotent by nature (GET, PUT with same data). Create operations generate new resources, so duplicates are acceptable for now.

**Consequences:**
- Duplicate POST requests create duplicate resources
- No protection against retry storms
- Future improvement: Add Idempotency-Key header support for POST/PATCH operations

**Evidence:**
- No idempotency handling in controllers or services
- `backend/src/controllers/VoucherController.ts` - create method has no idempotency check
