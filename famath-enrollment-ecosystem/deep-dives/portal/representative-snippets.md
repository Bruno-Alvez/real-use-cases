# Representative Code Snippets

## 1. Checkout Session Creation

**File:** `backend/src/services/checkoutService.ts`

```typescript
const response = await fetch(`${ASAAS_CONFIG.baseUrl}/checkouts`, {
  method: 'POST',
  headers: ASAAS_CONFIG.headers,
  body: JSON.stringify(checkoutPayload)
});

if (!response.ok) {
  const errorData = await response.json().catch(() => ({ 
    message: 'Unknown error' 
  })) as any;
  console.error('Asaas API error:', errorData);
  throw new Error(
    `Asaas error: ${errorData.message || errorData.errors?.[0]?.description || response.statusText}`
  );
}

const checkoutSession = await response.json() as { id: string; link: string };
```

**Context:** Direct HTTP call to Asaas API without retry logic or timeout configuration. Error handling extracts message from Asaas error response.

## 2. Voucher Validation and Deduplication Check

**File:** `backend/src/services/checkoutService.ts`

```typescript
async function fetchApplicableVoucher(
  cpfDigits: string, 
  voucherCode?: string
): Promise<VoucherRecord | null> {
  if (!cpfDigits) return null;

  let query = supabase
    .from('vouchers')
    .select('*')
    .eq('cpf_aluno', cpfDigits)
    .eq('status', 'ativo');

  if (voucherCode && voucherCode.trim().length > 0) {
    query = query.eq('codigo', voucherCode.trim());
  }

  const { data, error } = await query
    .order('created_at', { ascending: false })
    .limit(1);

  if (error || !data?.[0]) return null;

  const voucher = data[0] as VoucherRecord;
  
  if (voucher.data_expiracao) {
    const expiration = new Date(voucher.data_expiracao);
    if (expiration < new Date()) return null;
  }

  const { data: usageData } = await supabase
    .from('voucher_usage')
    .select('id')
    .eq('voucher_id', voucher.id)
    .limit(1);

  if (usageData && usageData.length > 0) return null;

  return voucher;
}
```

**Context:** Voucher deduplication via separate `voucher_usage` table. Checks expiration date and usage status before applying discount. Race condition possible if two checkouts query simultaneously.

## 3. Payment Confirmation Processing (Idempotency Check)

**File:** `backend/src/services/paymentService.ts`

```typescript
async processPaymentConfirmation(asaasChargeId: string): Promise<boolean> {
  try {
    const asaasCharge = await this.getAsaasCharge(asaasChargeId);
    const payment = await paymentRepository.findByReference(asaasChargeId);
    
    if (!payment) {
      console.error('Payment not found:', asaasChargeId);
      return false;
    }

    if (payment.status === 'paid') {
      console.log('Payment already processed');
      return true;
    }

    const newStatus = AsaasUtils.convertAsaasStatus(asaasCharge.status);
    
    await paymentRepository.update(payment.id, {
      status: newStatus,
      paid_at: asaasCharge.paymentDate || new Date().toISOString()
    });

    if (newStatus === 'paid') {
      await enrollmentRepository.updateStatus(
        payment.enrollment_id,
        'approved',
        'Enrollment Confirmed',
        'Payment confirmed. Enrollment approved successfully!'
      );
    }

    return true;
  } catch (error) {
    console.error('Error processing payment confirmation:', error);
    return false;
  }
}
```

**Context:** Idempotency check: if payment already processed (`status === 'paid'`), returns early. Updates payment status and enrollment status if payment confirmed.

## 4. Database Trigger for Automatic Status Update

**File:** `database/triggers/auto_update_enrollment_status.sql`

```sql
CREATE OR REPLACE FUNCTION auto_update_enrollment_status()
RETURNS TRIGGER AS $$
DECLARE
    enrollment_record RECORD;
    required_docs TEXT[];
    sent_docs TEXT[];
    approved_docs TEXT[];
    all_docs_sent BOOLEAN;
    all_docs_approved BOOLEAN;
BEGIN
    IF OLD.status IS DISTINCT FROM NEW.status THEN
        SELECT e.id, e.current_step, e.status, em.required_documents
        INTO enrollment_record
        FROM enrollments e
        JOIN entry_modalities em ON e.entry_modality_id = em.id
        WHERE e.id = NEW.enrollment_id;
        
        IF NOT FOUND THEN RETURN NEW; END IF;
        
        required_docs := enrollment_record.required_documents;
        IF required_docs IS NULL OR array_length(required_docs, 1) IS NULL THEN
            RETURN NEW;
        END IF;
        
        SELECT array_agg(DISTINCT document_type) INTO sent_docs
        FROM documents
        WHERE enrollment_id = NEW.enrollment_id
        AND status IN ('pending', 'approved', 'rejected');
        
        SELECT array_agg(DISTINCT document_type) INTO approved_docs
        FROM documents
        WHERE enrollment_id = NEW.enrollment_id AND status = 'approved';
        
        all_docs_sent := (
            SELECT bool_and(doc_type = ANY(COALESCE(sent_docs, ARRAY[]::TEXT[])))
            FROM unnest(required_docs) AS doc_type
        );
        
        all_docs_approved := (
            SELECT bool_and(doc_type = ANY(COALESCE(approved_docs, ARRAY[]::TEXT[])))
            FROM unnest(required_docs) AS doc_type
        );
        
        IF all_docs_sent AND enrollment_record.current_step = 'Aguardando Documentos' THEN
            UPDATE enrollments 
            SET current_step = 'Documentos em Análise',
                status = 'in_progress',
                last_update = NOW()
            WHERE id = NEW.enrollment_id;
        END IF;
        
        IF all_docs_approved AND enrollment_record.current_step = 'Documentos em Análise' THEN
            UPDATE enrollments 
            SET current_step = 'Aguardando Pagamento',
                status = 'approved_pending_payment',
                last_update = NOW()
            WHERE id = NEW.enrollment_id;
        END IF;
    END IF;
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

**Context:** Database trigger automatically updates enrollment status when documents are approved. Uses array comparison to check if all required documents are sent/approved.

## 5. Dynamic Pricing Calculation with Timezone

**File:** `backend/src/services/pricingService.ts`

```typescript
getTimezoneInfo(referenceDate?: string): TimezoneInfo {
  const timezone = 'America/Fortaleza';
  
  let today: Date;
  if (referenceDate) {
    today = new Date(referenceDate);
  } else {
    today = new Date();
  }
  
  const todayInTimezone = new Date(
    today.toLocaleString('en-US', { timeZone: timezone })
  );
  
  return {
    timezone,
    today: todayInTimezone,
    todayISO: todayInTimezone.toISOString().split('T')[0],
    dayOfMonth: todayInTimezone.getDate(),
    year: todayInTimezone.getFullYear(),
    month: todayInTimezone.getMonth() + 1
  };
}
```

**Context:** Timezone conversion to America/Fortaleza for pricing calculation. Day of month determines pricing window (D1-5, D6-10, D11-20, NOMINAL).

## 6. Voucher Discount Calculation

**File:** `backend/src/services/checkoutService.ts`

```typescript
function calculateVoucherDiscount(
  amount: number, 
  voucher: VoucherRecord
): { amount: number; discountApplied: number } {
  let discountedAmount = amount;
  let totalDiscount = 0;

  if (voucher.porcentagem_desconto && voucher.porcentagem_desconto > 0) {
    const percentDiscount = roundCurrency(
      amount * (voucher.porcentagem_desconto / 100)
    );
    discountedAmount -= percentDiscount;
    totalDiscount += percentDiscount;
  } else if (voucher.valor_desconto_fixo && voucher.valor_desconto_fixo > 0) {
    discountedAmount -= voucher.valor_desconto_fixo;
    totalDiscount += voucher.valor_desconto_fixo;
  }

  if (discountedAmount < 0) {
    discountedAmount = 0;
  }

  return {
    amount: roundCurrency(discountedAmount),
    discountApplied: roundCurrency(totalDiscount)
  };
}
```

**Context:** Applies only one discount type (percentage OR fixed), prioritizing percentage. Prevents over-discounting. Uses `roundCurrency()` for precision.

## 7. Error Handler Middleware

**File:** `backend/src/middleware/errorHandler.ts`

```typescript
export function errorHandler(
  err: ApiError,
  req: Request,
  res: Response,
  next: NextFunction
) {
  console.error('Error caught:', err);

  const statusCode = err.statusCode || 500;
  const message = err.message || 'Internal server error';

  res.status(statusCode).json({
    success: false,
    error: message,
    ...(process.env.NODE_ENV === 'development' && {
      stack: err.stack,
      details: err
    })
  });
}
```

**Context:** Global error handler. Returns error message to client. Stack trace only in development mode. No structured logging or error tracking service integration.

## 8. Graceful Shutdown Handler

**File:** `backend/src/server.ts`

```typescript
const gracefulShutdown = (signal: string) => {
  console.log(`Received signal ${signal}. Shutting down server...`);
  
  server.close(() => {
    console.log('Server closed successfully.');
    process.exit(0);
  });

  setTimeout(() => {
    console.error('Forcing server shutdown...');
    process.exit(1);
  }, 10000);
};
```

**Context:** Graceful shutdown on SIGTERM/SIGINT. Waits for active connections to close. Forces exit after 10 seconds if shutdown not completed.
