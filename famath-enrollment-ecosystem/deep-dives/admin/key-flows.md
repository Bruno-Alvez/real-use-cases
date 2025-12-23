# Key Flows

## Flow 1: Create Voucher

**Trigger:** POST `/api/v1/vouchers`

**Steps:**
1. Controller extracts request body and user ID (`VoucherController.create`)
2. Service validates CPF format, discount type, and values (`VoucherService.createVoucher`)
3. Service generates unique code with retry logic (max 10 attempts, 100ms delay) (`VoucherService.generateUniqueCode`)
4. Service calculates expiration date from validade_dias (`VoucherService.calculateExpirationDate`)
5. Repository inserts voucher record (`VoucherRepository.create`)
6. Database trigger populates nome_aluno from students table based on CPF (`supabase_schema_vouchers_campaigns_final.sql` line 384-388)
7. Controller returns created voucher

**Failure Points:**
- Invalid CPF format → 400 error from service validation
- Invalid discount values → 400 error from service validation
- Code generation fails after 10 attempts → 500 error
- Database constraint violation (duplicate code, invalid CPF) → 500 error from repository
- Database trigger fails to find student → nome_aluno remains null (non-fatal)

**Handling:**
- No retries for code generation failures (throws error after max attempts)
- No idempotency keys (duplicate requests create duplicate vouchers)
- Database UNIQUE constraint on codigo prevents duplicates at DB level
- Errors logged via Winston logger (`backend/src/utils/logger.ts`)

**Evidence:**
- `backend/src/controllers/VoucherController.ts` lines 53-71
- `backend/src/services/VoucherService.ts` lines 49-80, 253-271
- `backend/src/repositories/VoucherRepository.ts` lines 116-137
- `database/supabase/supabase_schema_vouchers_campaigns_final.sql` lines 384-388

## Flow 2: Update Campaign with Course/Shift Changes

**Trigger:** PUT `/api/v1/campaigns/:id`

**Steps:**
1. Controller extracts campaign ID, update data, courseIds, and shifts (`CampaignController.update`)
2. Service fetches existing campaign (`CampaignService.getCampaignById`)
3. Service queries campaign_courses, campaign_shifts, and campaign_usage tables (`CampaignRepository.getCampaignCourseIds`, `getCampaignShifts`, `getCampaignUsage`)
4. Service validates business rules:
   - If campaign has usage, prevent codigo_promocional changes
   - If campaign has usage, prevent reducing limite_total below usage count
   - Validate date ranges
5. Repository updates campaigns table (`CampaignRepository.update`)
6. Repository deletes existing campaign_courses and campaign_shifts records
7. Repository inserts new campaign_courses and campaign_shifts records
8. Database trigger updates campaign.utilizados count on campaign_usage changes (`supabase_schema_vouchers_campaigns_final.sql` lines 320-340)
9. Controller returns updated campaign

**Failure Points:**
- Campaign not found → 404 error
- Business rule violation (e.g., changing code on used campaign) → 400 error
- Database constraint violation (e.g., invalid shift value) → 400/500 error
- Partial update failure (campaign updated but courses/shifts fail) → Inconsistent state (no transaction rollback)

**Handling:**
- No database transactions (each operation commits independently)
- No rollback on partial failures
- Shift values validated against CHECK constraint ('MANHA', 'TARDE', 'NOITE', 'SABADOS') at database level
- Errors logged with context (`CampaignRepository.update` lines 229-241)

**Evidence:**
- `backend/src/controllers/CampaignController.ts` lines 91-135
- `backend/src/services/CampaignService.ts` lines 94-184
- `backend/src/repositories/CampaignRepository.ts` lines 220-332
- `database/supabase/supabase_schema_vouchers_campaigns_final.sql` lines 144-157 (campaign_shifts CHECK constraint)

## Flow 3: Apply Campaign Discount (Database Function)

**Trigger:** Database function call `apply_campaign_discount(campaign_id, student_id, valor_original)`

**Steps:**
1. Function checks if campaign is active (`is_campaign_active`)
2. Function checks if student can use campaign (`can_student_use_campaign`)
3. Function checks if campaign has available slots (`campaign_has_available_slots`)
4. Function calculates discount using `calculate_discount` function
5. Returns valor_desconto, valor_final, is_valid

**Failure Points:**
- Campaign not active → returns is_valid=false
- Student exceeded per-student limit → returns is_valid=false
- Campaign exceeded total limit → returns is_valid=false
- Invalid discount configuration → returns zero discount

**Handling:**
- All validations return boolean flags (no exceptions)
- Function is idempotent (same inputs produce same outputs)
- No retries (called synchronously from application code)

**Evidence:**
- `database/supabase/supabase_schema_vouchers_campaigns_final.sql` lines 602-644

## Flow 4: Daily Status Updates (Cron Job)

**Trigger:** Scheduled SQL script execution (daily at 1 AM)

**Steps:**
1. Update vouchers with status='ativo' and data_expiracao < CURRENT_DATE to status='expirado'
2. Update campaigns with status='agendada' and data_inicio <= CURRENT_DATE to status='ativa'
3. Update campaigns with status IN ('ativa', 'pausada') and data_fim < CURRENT_DATE to status='finalizada'
4. Log each operation to audit_logs table
5. Generate summary report with counts of active vouchers/campaigns

**Failure Points:**
- Database connection failure → Script fails, no partial updates
- Concurrent modification during execution → Last write wins (no locking)
- Audit log insertion failure → Operation succeeds but no audit trail

**Handling:**
- No retry mechanism (relies on cron to retry next day)
- No idempotency (re-running script multiple times produces same results)
- Errors logged to audit_logs if insertion succeeds
- No dead-letter queue (failures require manual investigation)

**Evidence:**
- `database/supabase/supabase_cron_daily_updates.sql` - Full script

## Flow 5: Frontend Campaign Form Submission

**Trigger:** User submits campaign edit form in React UI

**Steps:**
1. Frontend validates form inputs (`CampaignForm.tsx`)
2. Frontend calls `updateCampaign` service (`frontend/src/services/supabase/campaigns.ts`)
3. Service maps frontend shift labels to database values (e.g., 'NOTURNO' → 'NOITE')
4. Service sends PUT request to backend API
5. Backend processes request (Flow 2 above)
6. Frontend shows loading state ("Salvando...")
7. On success: show toast, wait 1.5s, close modal
8. On error: show error toast, keep modal open, re-enable form

**Failure Points:**
- Infinite request loop (fixed with useRef guard) → ERR_INSUFFICIENT_RESOURCES
- Invalid shift mapping → 400 error from database CHECK constraint
- Network failure → TypeError: Failed to fetch
- State desynchronization → Checkboxes "flicker" (fixed with controlled initialization)

**Handling:**
- useRef prevents re-initialization loops (`CampaignForm.tsx` initializationRef)
- Shift mapping validates against allowed values before sending
- Error boundaries catch and display errors
- Loading state prevents duplicate submissions

**Evidence:**
- `frontend/src/components/Voucher/CampaignForm.tsx` - Form component
- `frontend/src/services/supabase/campaigns.ts` - Service with shift mapping
- `backend/src/repositories/CampaignRepository.ts` lines 469-479 (normalizeShift)
