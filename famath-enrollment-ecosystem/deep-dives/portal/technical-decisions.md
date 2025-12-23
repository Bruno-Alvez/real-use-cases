# Technical Decisions

## Decision 1: Supabase over Direct PostgreSQL Connection

**Decision:** Use Supabase JS client instead of direct PostgreSQL connection (pg library).

**Alternatives Considered:**
- Direct `pg` library connection with connection pooling
- Prisma ORM with PostgreSQL
- TypeORM with PostgreSQL

**Why Chosen:**
- Supabase provides built-in authentication, storage, and real-time capabilities
- Client-side can query database directly (reduces backend load)
- Automatic connection management and retries
- Free tier sufficient for initial scale

**Consequences:**
- Vendor lock-in to Supabase
- Limited control over connection pooling configuration
- Client-side queries bypass backend validation (security risk if RLS not configured)
- Supabase client timeout behavior not customizable

**Future Improvements:**
- Implement backend API layer for all database operations (remove direct client queries)
- Add connection pooling configuration if moving to direct PostgreSQL
- Consider Prisma for type-safe queries and migrations

**Evidence:**
- `backend/src/config/database.ts` - Supabase client initialization (lines 12-22)

---

## Decision 2: Database Triggers for Protocol Generation

**Decision:** Generate enrollment protocol numbers (`YYYY-NNNN`) via PostgreSQL trigger and function.

**Alternatives Considered:**
- Application-level sequential number generation (Redis counter, database sequence)
- UUID-based protocol numbers
- External service for ID generation

**Why Chosen:**
- Guarantees sequential, year-based numbering without race conditions
- Database-level atomicity prevents duplicates
- No external dependencies (Redis, etc.)
- Simple implementation: trigger fires on INSERT

**Consequences:**
- Protocol format tied to database function (hard to change)
- Year-based reset requires function update each year
- No distributed ID generation (single database instance)
- Difficult to test trigger logic in isolation

**Future Improvements:**
- Extract protocol generation to application service for testability
- Use database sequences for better performance
- Consider distributed ID generation if scaling horizontally

**Evidence:**
- `database/triggers/auto_create_student_profile.sql` - Protocol generation trigger
- `database/schema.sql` - `enrollments.protocol_number` UNIQUE constraint (line 73)

---

## Decision 3: Synchronous Asaas API Calls (No Retry/Backoff)

**Decision:** Make direct HTTP `fetch()` calls to Asaas API without retry logic or exponential backoff.

**Alternatives Considered:**
- Implement retry with exponential backoff
- Use queue system (Bull, BullMQ) for async processing
- Circuit breaker pattern for API failures

**Why Chosen:**
- Simplicity: immediate feedback to user
- Asaas API generally reliable (low failure rate assumed)
- No additional infrastructure (queues, workers) needed
- Faster implementation

**Consequences:**
- Transient network failures cause checkout failures (user must retry)
- No automatic recovery from temporary Asaas outages
- Rate limiting from Asaas not handled gracefully
- No idempotency protection (duplicate checkouts possible on retry)

**Future Improvements:**
- Implement retry with exponential backoff (3 attempts, 1s/2s/4s delays)
- Add idempotency keys to prevent duplicate checkout sessions
- Implement circuit breaker to fail fast during Asaas outages
- Queue checkout creation for async processing with retry

**Evidence:**
- `backend/src/services/checkoutService.ts` - Direct `fetch()` calls (lines 326-330, 683-687, 865-869)

---

## Decision 4: Database Triggers for Automatic Status Updates

**Decision:** Use PostgreSQL trigger to automatically update enrollment status when documents are approved.

**Alternatives Considered:**
- Application-level status check on document update
- Scheduled job to check document status periodically
- Event-driven architecture (message queue, webhooks)

**Why Chosen:**
- Immediate status update without application code
- Database-level consistency (ACID guarantees)
- No additional infrastructure (queues, workers)
- Reduces application complexity

**Consequences:**
- Business logic in database (harder to test, version, and debug)
- Trigger execution not visible in application logs
- Difficult to modify trigger logic without database migration
- No way to disable automatic updates for specific cases

**Future Improvements:**
- Move status update logic to application service
- Use database events or change data capture (CDC) to trigger application logic
- Add configuration flag to enable/disable automatic updates

**Evidence:**
- `database/triggers/auto_update_enrollment_status.sql` - Complete trigger implementation

---

## Decision 5: Voucher Discount Application (Percentage OR Fixed, Not Both)

**Decision:** Apply only one discount type per voucher: percentage discount OR fixed value discount, prioritizing percentage.

**Alternatives Considered:**
- Apply both discounts sequentially (percentage then fixed)
- Apply both discounts additively
- Allow only one discount type per voucher (database constraint)

**Why Chosen:**
- Prevents over-discounting (voucher with both types would apply too much discount)
- Clear precedence: percentage takes priority
- Matches business requirement (vouchers should have single discount type)

**Consequences:**
- If voucher has both fields populated, fixed discount ignored
- No validation to prevent vouchers with both discount types
- Business logic in application code (not enforced at database level)

**Future Improvements:**
- Add database constraint: CHECK that only one discount type is non-null
- Validate voucher creation to prevent both discount types
- Document discount precedence clearly

**Evidence:**
- `backend/src/services/checkoutService.ts` - `calculateVoucherDiscount()` (lines 120-144)
- `backend/src/services/pricingService.ts` - `applyVoucherDiscount()` - Similar logic

---

## Decision 6: Timezone-Aware Pricing (America/Fortaleza)

**Decision:** Calculate pricing based on America/Fortaleza timezone, not server timezone.

**Alternatives Considered:**
- Use server timezone (UTC or server local time)
- Use client timezone (browser timezone)
- Store all dates in UTC and convert on display only

**Why Chosen:**
- Business rules depend on local date (day of month determines pricing window)
- Consistent pricing regardless of server location
- Prevents timezone-related pricing errors

**Consequences:**
- Requires timezone conversion logic in application
- JavaScript Date timezone handling can be error-prone
- Day-of-month calculation depends on correct timezone conversion
- No timezone validation (invalid timezone string would use default)

**Future Improvements:**
- Use library like `date-fns-tz` for reliable timezone conversion
- Validate timezone string in configuration
- Add unit tests for timezone edge cases (day boundaries)

**Evidence:**
- `backend/src/services/pricingService.ts` - `getTimezoneInfo()` (lines 98-119)

---

## Decision 7: Name Truncation for Asaas (30 Character Limit)

**Decision:** Truncate customer names to 30 characters when sending to Asaas API.

**Alternatives Considered:**
- Reject enrollments with names > 30 characters
- Split name into multiple fields
- Use abbreviation logic

**Why Chosen:**
- Asaas API enforces 30-character limit (causes errors if exceeded)
- Truncation is simplest solution
- Preserves enrollment flow (doesn't block students)

**Consequences:**
- Names may be truncated mid-word
- No notification to user that name was truncated
- Potential confusion if truncated name appears in Asaas receipts

**Future Improvements:**
- Validate name length on enrollment form, show warning
- Implement smart truncation (truncate at word boundary)
- Store full name in database, truncated name only for Asaas

**Evidence:**
- `backend/src/services/checkoutService.ts` - `customerName.substring(0, 30)` (lines 296, 656, 838)
- `backend/src/services/paymentService.ts` - `student.full_name.substring(0, 30)` (line 50)

---

## Decision 8: Payment Values in Reais (Not Cents) for Asaas Checkout

**Decision:** Send payment amounts in reais (decimal format) to Asaas checkout API, not cents.

**Alternatives Considered:**
- Send amounts in cents (integer, multiply by 100)
- Always send as decimal with 2 decimal places

**Why Chosen:**
- Asaas checkout API expects decimal values in reais
- Avoids "value exceeds limit" errors (150,000 reais limit, not 15,000,000 cents)
- Matches Asaas API documentation for checkout sessions

**Consequences:**
- Inconsistency with legacy payment API (may expect cents)
- Floating-point precision concerns (rounding handled via `roundCurrency()`)
- Must ensure consistent currency formatting across system

**Future Improvements:**
- Standardize on single currency format (reais or cents) across all APIs
- Use decimal library for precise currency calculations
- Document currency format requirements clearly

**Evidence:**
- `backend/src/services/checkoutService.ts` - `value: finalAmount` (lines 313, 675, 857) - Values in reais
- Historical: Previous implementation multiplied by 100 (removed in recent fix)

---

## Decision 9: Single Installment Limit for Credit Card Payments

**Decision:** Enforce `maxInstallmentCount: 1` for credit card checkout sessions.

**Alternatives Considered:**
- Allow multiple installments (2-12 months)
- Make installments configurable per course
- Remove installment option entirely

**Why Chosen:**
- Business requirement: enrollment fees must be paid in full
- Prevents confusion (students might expect to pay installments for full course)
- Simplifies payment tracking (single payment per enrollment)

**Consequences:**
- Students cannot split enrollment fee into installments
- May reduce conversion if students cannot afford full payment
- Asaas checkout UI still shows installment option (though limited to 1)

**Future Improvements:**
- Make installment limit configurable per course or pricing tier
- Add business logic to allow installments for full course payment (not enrollment fee)
- Communicate installment policy clearly to students

**Evidence:**
- `backend/src/services/checkoutService.ts` - `installment: { maxInstallmentCount: 1 }` (lines 316, 829)

---

## Decision 10: No Idempotency Keys for Checkout Creation

**Decision:** Do not implement idempotency keys when creating Asaas checkout sessions.

**Alternatives Considered:**
- Generate idempotency key (UUID) per checkout request
- Use enrollment ID + payment option as idempotency key
- Store idempotency key in database, check before creating checkout

**Why Chosen:**
- Simplicity: no additional storage or logic needed
- Assumed low retry rate (users don't double-click frequently)
- Asaas may handle duplicates on their side (not verified)

**Consequences:**
- Duplicate checkout sessions possible if user retries or network retries
- Multiple charges possible (though Asaas may prevent this)
- No way to safely retry failed checkout creation
- Database may have multiple `checkout_sessions` records for same enrollment

**Future Improvements:**
- Implement idempotency key generation and storage
- Check for existing active checkout session before creating new one
- Add unique constraint on (enrollment_id, status) to prevent duplicates
- Use idempotency key in Asaas request if API supports it

**Evidence:**
- `backend/src/services/checkoutService.ts` - No idempotency check before `fetch()` calls
- `database/schema.sql` - `checkout_sessions` table has no unique constraint on enrollment_id
