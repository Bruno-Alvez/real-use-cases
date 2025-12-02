# Challenges

## Timezone-Aware Pricing

**Problem**: Pricing depends on enrollment date. Need timezone-aware calculations. Edge cases at midnight boundaries.

**Solution**: Timezone info extraction with explicit timezone handling. Date comparison with timezone. Pre-enrollment window detection with boundary handling.

```typescript
const timezoneInfo = this.getTimezoneInfo(context.referenceDate);
const preEnrollmentWindow = this.getPreEnrollmentWindow(timezoneInfo, coursePricing);

// Handle edge case: enrollment at 23:59 vs 00:01
if (preEnrollmentWindow.isActive) {
    const isWithinWindow = this.isWithinWindow(
        timezoneInfo.today,
        preEnrollmentWindow.startDate,
        preEnrollmentWindow.endDate
    );
    if (isWithinWindow) {
        return { amount: coursePricing.pre_enrollment_fee, feeType: 'pre_enrollment' };
    }
}
```

**Trade-off**: Server-side timezone handling adds complexity but ensures consistency. Client-side would be simpler but unreliable (timezone spoofing, clock skew).

## Voucher Application

**Problem**: Vouchers must be validated (CPF match, expiration, status) before applying discount.

**Solution**: Voucher lookup with validation. Expiration check. Discount calculation.

```typescript
async fetchApplicableVoucher(cpf: string, voucherCode?: string) {
    const voucher = await supabase.from('vouchers')
        .eq('cpf_aluno', cpf)
        .eq('status', 'ativo')
        .eq('codigo', voucherCode || '')
        .single();
    
    if (voucher?.data_expiracao && new Date(voucher.data_expiracao) < new Date()) {
        return null;
    }
    
    return voucher;
}
```

CPF validation. Expiration check. Status validation. Discount application.

## Document Organization

**Problem**: Documents need organized storage. CPF-based folder structure prevents conflicts.

**Solution**: CPF-based folder structure. Timestamped filenames. Database tracking.

```typescript
const fileName = `${student.cpf}/${documentType}-${Date.now()}.${extension}`;
await supabase.storage.from('documents').upload(fileName, file);
```

Organized storage. Easy retrieval. Conflict prevention.

## Payment Webhook Reliability

**Problem**: Asaas webhooks must be processed reliably. Need idempotency, ordering, and duplicate detection.

**Solution**: Idempotency keys, webhook signature verification, event deduplication. Database transaction ensures atomicity.

```typescript
app.post('/api/payments/webhook', async (req, res) => {
    const signature = req.headers['asaas-signature'];
    if (!verifySignature(req.body, signature)) {
        return res.status(401).send('Invalid signature');
    }
    
    const { event, payment } = req.body;
    const idempotencyKey = `${payment.id}-${event}`;
    
    // Idempotent processing with database lock
    const result = await db.transaction(async (tx) => {
        const existing = await tx.payment_webhooks.findUnique({
            where: { idempotency_key: idempotencyKey }
        });
        if (existing) return { processed: false };
        
        await tx.payment_webhooks.create({ data: { idempotency_key: idempotencyKey } });
        await processPaymentConfirmation(payment, tx);
        return { processed: true };
    });
    
    res.status(200).send('OK');
});
```

Idempotent processing. Signature verification. Atomic transactions. Handles duplicate webhooks and network retries.

## Checkout Session Management

**Problem**: Checkout sessions must be tracked. Need expiration and status management.

**Solution**: Database session storage. Expiration tracking. Status updates.

```typescript
async createCheckoutSession(data: CheckoutData) {
    const session = await asaas.post('/checkout', { ... });
    
    await supabase.from('checkout_sessions').insert({
        enrollment_id: data.enrollmentId,
        asaas_session_id: session.id,
        amount: finalAmount,
        expires_at: session.expiresAt,
        status: 'pending',
    });
    
    return session;
}
```

Session tracking. Expiration management. Status updates.

