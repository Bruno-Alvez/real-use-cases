# Key Flows

## Enrollment Creation Flow

Form submission → validation → student creation → enrollment creation:

```typescript
async createEnrollment(enrollmentData: EnrollmentData) {
    const student = await studentRepository.findByCpf(enrollmentData.cpf);
    
    if (!student) {
        student = await studentRepository.create(enrollmentData);
    }
    
    const enrollment = await enrollmentRepository.create({
        student_id: student.id,
        course_id: enrollmentData.courseId,
        entry_modality_id: enrollmentData.modalityId,
        protocol_number: generateProtocol(),
        status: 'pending',
    });
    
    return enrollment;
}
```

Flow: Form → validation → student lookup/create → enrollment creation → protocol generation → response.

## Smart Pricing Flow

Course selection → pricing calculation → discount application:

```typescript
async getPricing(context: PricingContext): Promise<PricingResult> {
    const timezoneInfo = this.getTimezoneInfo(context.referenceDate);
    const coursePricing = await this.getCoursePricing(context.courseId, context.shift);
    const preEnrollmentWindow = this.getPreEnrollmentWindow(timezoneInfo, coursePricing);
    const campaignInfo = await this.getCampaignInfo(context.courseId);
    
    if (preEnrollmentWindow.isActive) {
        return { amount: coursePricing.pre_enrollment_fee, feeType: 'pre_enrollment' };
    }
    
    if (campaignInfo.hasActiveCampaign) {
        return { amount: campaignInfo.campaignValues[context.shift], feeType: 'campaign' };
    }
    
    return { amount: coursePricing.full_fee, feeType: 'full' };
}
```

Date-based calculation. Pre-enrollment window check. Campaign priority. Dynamic pricing.

## Checkout Flow

Enrollment → pricing → voucher → Asaas session:

```typescript
async createCheckoutSession(data: CheckoutData) {
    const pricing = await pricingService.getPricing({
        courseId: data.courseId,
        shift: data.shift,
        referenceDate: data.referenceDate,
    });
    
    const voucher = await this.fetchApplicableVoucher(data.studentCpf, data.selectedVoucher);
    const finalAmount = this.applyVoucherDiscount(pricing.amount, voucher);
    
    const session = await asaas.post('/checkout', {
        value: finalAmount,
        customer: { name: data.studentName, email: data.studentEmail, cpf: data.studentCpf },
        callbackUrl: CALLBACK_URLS.success,
    });
    
    await this.saveCheckoutSession(enrollmentId, session.id, finalAmount);
    
    return { checkoutUrl: session.checkoutUrl, checkoutId: session.id };
}
```

Pricing calculation. Voucher lookup. Discount application. Asaas session creation. Session tracking.

## Document Upload Flow

File selection → validation → upload → database record:

```typescript
async uploadDocument(file: File, enrollmentId: string, documentType: string) {
    const enrollment = await enrollmentRepository.findById(enrollmentId);
    const student = await studentRepository.findById(enrollment.student_id);
    
    const fileName = `${student.cpf}/${documentType}-${Date.now()}.${file.name.split('.').pop()}`;
    
    const { data, error } = await supabase.storage
        .from('documents')
        .upload(fileName, file, { contentType: file.type });
    
    if (error) throw error;
    
    await documentRepository.create({
        enrollment_id: enrollmentId,
        document_type: documentType,
        file_url: data.path,
        status: 'pending',
    });
}
```

File validation. CPF-based organization. Storage upload. Database record. Status tracking.

## Payment Webhook Flow

Asaas webhook → payment verification → enrollment update:

```typescript
app.post('/api/payments/webhook', async (req, res) => {
    const { event, payment } = req.body;
    
    if (event === 'PAYMENT_CONFIRMED') {
        const checkoutSession = await this.findCheckoutSessionByPaymentId(payment.id);
        if (checkoutSession) {
            await enrollmentRepository.updateStatus(checkoutSession.enrollment_id, 'payment_confirmed');
            await this.createTimelineEvent(checkoutSession.enrollment_id, 'payment', 'Pagamento confirmado');
        }
    }
    
    res.status(200).send('OK');
});
```

Webhook reception. Payment verification. Enrollment status update. Timeline event creation.

