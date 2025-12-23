# Key Flows

## Flow 1: Enrollment Creation and Protocol Generation

**Trigger:** Student submits enrollment form via frontend.

**Steps:**
1. Frontend calls `POST /api/enrollments` with student and course data
2. Controller validates input (`enrollmentController.ts`)
3. `enrollmentService.createEnrollment()` calls `enrollmentRepository.create()`
4. Repository inserts into `enrollments` table
5. Database trigger `set_enrollment_protocol` fires on INSERT
6. Trigger calls `generate_protocol_number()` function
7. Function generates sequential protocol: `YYYY-NNNN` format
8. Protocol number assigned to enrollment record
9. Timeline event created: `enrollment_created`
10. Response returned with enrollment ID and protocol number

**Failure Points:**
- Database connection failure during insert
- Protocol number generation collision (handled by UNIQUE constraint)
- Timeline event creation failure (logged, doesn't block enrollment)

**Handling:**
- Database errors propagate to controller, return 500 with error message
- UNIQUE constraint on `protocol_number` prevents duplicates
- Timeline errors caught and logged, enrollment still created
- No retry logic implemented

**Evidence:**
- `backend/src/services/enrollmentService.ts` - Lines 10-44
- `database/triggers/auto_create_student_profile.sql` - Protocol generation trigger
- `database/schema.sql` - `enrollments` table with `protocol_number` UNIQUE constraint (line 73)

---

## Flow 2: Checkout Session Creation with Dynamic Pricing

**Trigger:** Student clicks "Finalizar Pagamento" on checkout page.

**Steps:**
1. Frontend calls `POST /api/checkout/pix` or `/api/checkout/cartao` with enrollment data
2. `checkoutController.createPixCheckout()` validates required fields
3. `checkoutService.createPixCheckoutSession()` called
4. Service validates checkout data (CPF format, email, required fields)
5. `pricingService.getPricing()` calculates price:
   - Gets timezone info (America/Fortaleza)
   - Fetches `course_pricing` for course/shift
   - Checks pre-enrollment window (deadline comparison)
   - Checks active campaigns
   - Determines pricing window (D1-5, D6-10, D11-20, NOMINAL)
   - Returns `PricingResult` with amount and options
6. Service fetches enrollment with student data from database
7. Validates student address fields (required by Asaas)
8. Applies voucher discount if voucher code provided:
   - Queries `vouchers` table (active, not expired, not used)
   - Calculates discount (percentage or fixed, not both)
9. Truncates customer name to 30 characters (Asaas limit)
10. Builds Asaas checkout payload with customer data, items, callbacks
11. HTTP POST to `https://www.asaas.com/api/v3/checkouts`
12. Asaas returns checkout session (id, link)
13. Service saves session to `checkout_sessions` table
14. Updates enrollment status to "Aguardando Pagamento"
15. Returns checkout URL to frontend

**Failure Points:**
- Pricing calculation failure (course pricing not found)
- Missing student address fields
- Asaas API error (rate limit, invalid data, network failure)
- Database save failure after Asaas success
- Voucher validation failure (expired, already used)

**Handling:**
- Pricing errors throw exception, caught by controller, return 400 with message
- Missing address fields: explicit error message listing missing fields
- Asaas errors: error response parsed, message extracted, returned to frontend
- Database save errors: logged, but checkout session already created in Asaas (inconsistency risk)
- Voucher errors: silently ignored, checkout proceeds without discount
- No retry on Asaas API failures
- No idempotency key to prevent duplicate checkouts

**Evidence:**
- `backend/src/services/checkoutService.ts` - `createPixCheckoutSession()` (lines 543-744)
- `backend/src/services/pricingService.ts` - `getPricing()` (lines 48-93)
- `backend/src/config/payment.ts` - Asaas configuration (lines 11-19)

---

## Flow 3: Automatic Enrollment Status Update via Database Trigger

**Trigger:** Document status updated in `documents` table (INSERT or UPDATE).

**Steps:**
1. Document uploaded or status changed (e.g., `pending` → `approved`)
2. `auto_update_enrollment_status_trigger` fires AFTER UPDATE
3. Trigger function `auto_update_enrollment_status()` executes
4. Function queries enrollment and entry modality
5. Gets required documents list from `entry_modalities.required_documents`
6. Queries all documents for enrollment (pending, approved, rejected)
7. Checks if all required documents sent (array comparison)
8. Checks if all required documents approved
9. If all sent and status is "Aguardando Documentos":
   - Updates enrollment: `current_step = 'Documentos em Análise'`, `status = 'in_progress'`
   - Inserts timeline event: `status_auto_updated`
10. If all approved and status is "Documentos em Análise":
   - Updates enrollment: `current_step = 'Aguardando Pagamento'`, `status = 'approved_pending_payment'`
   - Inserts timeline event: `status_auto_updated`
11. Returns NEW record (trigger completion)

**Failure Points:**
- Trigger execution failure (database error, constraint violation)
- Missing entry modality configuration
- Race condition if multiple documents updated simultaneously

**Handling:**
- Database errors logged by PostgreSQL, trigger aborts, document update rolls back
- Missing modality: trigger returns early, no status update
- Race conditions: PostgreSQL row-level locking prevents concurrent updates
- No retry mechanism (trigger is synchronous)

**Evidence:**
- `database/triggers/auto_update_enrollment_status.sql` - Complete trigger implementation (lines 6-143)

---

## Flow 4: Voucher Application and Usage Tracking

**Trigger:** Student enters voucher code on checkout page.

**Steps:**
1. Frontend calls voucher validation endpoint (or includes code in checkout request)
2. `checkoutService.createPixCheckoutSession()` calls `applyVoucherIfAvailable()`
3. `fetchApplicableVoucher()` queries `vouchers` table:
   - Filters by CPF, status='ativo', optional code match
   - Checks expiration date (if `data_expiracao` < today, returns null)
   - Queries `voucher_usage` to check if already used
   - Returns first active, non-expired, unused voucher
4. `calculateVoucherDiscount()` applies discount:
   - If `porcentagem_desconto` exists: applies percentage
   - Else if `valor_desconto_fixo` exists: applies fixed amount
   - Only one discount type applied (not both)
5. Discounted amount calculated and returned
6. Checkout proceeds with discounted amount
7. After successful payment, `markVoucherAsUsed()` called:
   - Inserts record into `voucher_usage` table
   - Updates voucher `status` to 'usado'

**Failure Points:**
- Voucher query failure
- Expired voucher not filtered correctly
- Race condition: two checkouts use same voucher simultaneously
- Voucher marked as used but payment fails

**Handling:**
- Query errors: logged, voucher ignored, checkout proceeds without discount
- Expiration check: explicit date comparison in code
- Race condition: no database-level lock, two checkouts could both see voucher as available
- Usage marking: happens after payment success, but if payment fails voucher remains available (inconsistency)

**Evidence:**
- `backend/src/services/checkoutService.ts` - `fetchApplicableVoucher()` (lines 65-118)
- `backend/src/services/checkoutService.ts` - `calculateVoucherDiscount()` (lines 120-144)
- `backend/src/services/checkoutService.ts` - `markVoucherAsUsed()` (lines 503-538)

---

## Flow 5: Dynamic Pricing Calculation with Timezone Awareness

**Trigger:** Pricing endpoint called or checkout initiated.

**Steps:**
1. `pricingService.getPricing()` receives context (courseId, shift, referenceDate, studentCpf)
2. `getTimezoneInfo()` converts reference date to America/Fortaleza timezone
3. Extracts day of month, year, month from timezone-adjusted date
4. `getCoursePricing()` queries `course_pricing` table:
   - Filters by course_id, shift (normalized), is_active=true
   - Filters by validity dates (valid_from <= today, valid_until >= today or null)
   - Returns most recent pricing record
5. `getPreEnrollmentWindow()` checks if today <= `pre_enrollment_deadline`
6. `getCampaignInfo()` queries `campaigns` table:
   - Filters by status='ativa', date range includes today
   - Returns campaign discount values
7. `getPricingWindow()` determines window based on day of month:
   - Days 1-5: D1-5 window
   - Days 6-10: D6-10 window
   - Days 11-20: D11-20 window
   - Days 21+: NOMINAL (no discount)
8. `calculatePricing()` applies rules:
   - If pre-enrollment window active: returns `pre_enrollment_fee` and `nominal_value` as options
   - Else: returns `nominal_value` only
   - Campaign discounts not currently applied (campaign info fetched but unused)
9. Returns `PricingResult` with amount, options, metadata

**Failure Points:**
- Timezone conversion error
- Course pricing not found (no active pricing for course/shift)
- Invalid date format in referenceDate
- Database query timeout

**Handling:**
- Timezone errors: uses system default, may cause incorrect day calculation
- Missing pricing: throws error, caught by caller, returns 400
- Invalid dates: JavaScript Date parsing may return invalid date, no explicit validation
- Query timeouts: Supabase client timeout, error propagated
- No caching of pricing results (recalculated on every request)

**Evidence:**
- `backend/src/services/pricingService.ts` - `getPricing()` (lines 48-93)
- `backend/src/services/pricingService.ts` - `getTimezoneInfo()` (lines 98-119)
- `backend/src/services/pricingService.ts` - `calculatePricing()` (lines 275-351)
