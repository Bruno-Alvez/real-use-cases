# Representative Snippets

## Express App Setup

Security middleware configuration:

```typescript
const app = express();

app.use(helmet(SECURITY_CONFIG.helmet));
app.use(cors(SECURITY_CONFIG.cors));
app.use(rateLimit(SECURITY_CONFIG.rateLimit));
app.use(express.json({ limit: '10mb' }));

app.use('/api', routes);
```

## Smart Pricing Calculation

Date-based pricing with timezone:

```typescript
async getPricing(context: PricingContext): Promise<PricingResult> {
    const timezoneInfo = this.getTimezoneInfo(context.referenceDate);
    const coursePricing = await this.getCoursePricing(context.courseId, context.shift);
    const preEnrollmentWindow = this.getPreEnrollmentWindow(timezoneInfo, coursePricing);
    
    if (preEnrollmentWindow.isActive) {
        return { amount: coursePricing.pre_enrollment_fee, feeType: 'pre_enrollment' };
    }
    
    return { amount: coursePricing.full_fee, feeType: 'full' };
}
```

## Checkout Session Creation

Pricing + voucher + Asaas:

```typescript
async createCheckoutSession(data: CheckoutData) {
    const pricing = await pricingService.getPricing({ courseId: data.courseId, shift: data.shift });
    const voucher = await this.fetchApplicableVoucher(data.studentCpf, data.selectedVoucher);
    const finalAmount = this.applyVoucherDiscount(pricing.amount, voucher);
    
    const session = await asaas.post('/checkout', {
        value: finalAmount,
        customer: { name: data.studentName, email: data.studentEmail, cpf: data.studentCpf },
        callbackUrl: CALLBACK_URLS.success,
    });
    
    return { checkoutUrl: session.checkoutUrl, checkoutId: session.id };
}
```

## Document Upload

Supabase Storage with organization:

```typescript
async uploadDocument(file: File, enrollmentId: string, documentType: string) {
    const student = await getStudentByEnrollmentId(enrollmentId);
    const fileName = `${student.cpf}/${documentType}-${Date.now()}.${file.name.split('.').pop()}`;
    
    const { data } = await supabase.storage
        .from('documents')
        .upload(fileName, file, { contentType: file.type });
    
    await supabase.from('documents').insert({
        enrollment_id: enrollmentId,
        document_type: documentType,
        file_url: data.path,
        status: 'pending',
    });
}
```

## Voucher Lookup

CPF-based voucher validation:

```typescript
async fetchApplicableVoucher(cpf: string, voucherCode?: string) {
    let query = supabase.from('vouchers')
        .eq('cpf_aluno', cpf)
        .eq('status', 'ativo');
    
    if (voucherCode) query = query.eq('codigo', voucherCode);
    
    const { data } = await query.limit(1);
    const voucher = data?.[0];
    
    if (voucher?.data_expiracao && new Date(voucher.data_expiracao) < new Date()) {
        return null;
    }
    
    return voucher;
}
```

## Payment Webhook

Asaas webhook handling:

```typescript
app.post('/api/payments/webhook', async (req, res) => {
    const { event, payment } = req.body;
    
    if (event === 'PAYMENT_CONFIRMED') {
        const checkoutSession = await findCheckoutSessionByPaymentId(payment.id);
        if (checkoutSession) {
            await enrollmentRepository.updateStatus(
                checkoutSession.enrollment_id,
                'payment_confirmed'
            );
        }
    }
    
    res.status(200).send('OK');
});
```

