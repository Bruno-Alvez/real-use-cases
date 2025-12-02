# Architecture

## System Overview

Monorepo with React frontend and Express backend. Layered architecture: Routes → Controllers → Services → Repositories → Supabase. Asaas integration for payments. Supabase Storage for documents.

## Backend Architecture

### Express Application

Express with security middleware:

```typescript
const app = express();

app.use(helmet(SECURITY_CONFIG.helmet));
app.use(cors(SECURITY_CONFIG.cors));
app.use(rateLimit(SECURITY_CONFIG.rateLimit));
app.use(express.json({ limit: '10mb' }));

app.use('/api', routes);
```

Security headers, CORS, rate limiting, request logging.

### Layered Architecture

**Routes**: API endpoints  
**Controllers**: HTTP handling  
**Services**: Business logic (pricing, checkout, payments)  
**Repositories**: Data access

```typescript
// Route
router.post('/checkout/create-session', checkoutController.createSession);

// Controller
async createSession(req: Request, res: Response) {
    const session = await checkoutService.createCheckoutSession(req.body);
    res.json({ success: true, data: session });
}

// Service
async createCheckoutSession(data: CheckoutData) {
    const pricing = await pricingService.getPricing({ courseId, shift });
    const voucher = await this.fetchApplicableVoucher(cpf, voucherCode);
    const finalAmount = this.calculateFinalAmount(pricing, voucher);
    return await this.createAsaasSession(finalAmount, data);
}
```

Clear separation. Business logic in services. Type-safe interfaces.

## Frontend Architecture

React with Context API. Component-based UI. Service layer for API calls.

Structure:
- **pages/**: Route pages (EnrollmentTracking, FormularioInscricao, Checkout)
- **services/**: Business logic (enrollmentService, paymentService, pricingService)
- **contexts/**: Global state (AuthContext, EnrollmentContext)
- **components/**: Reusable UI components
- **api/**: Axios client configuration

```typescript
const AppContent: React.FC = () => {
    const { user } = useAuthContext();
    if (!user) return <Login />;
    return <Layout />;
};
```

Context-based state. Protected routes. Service layer abstraction.

## Smart Pricing Engine

Date-based pricing calculation:

```typescript
async getPricing(context: PricingContext): Promise<PricingResult> {
    const timezoneInfo = this.getTimezoneInfo(context.referenceDate);
    const coursePricing = await this.getCoursePricing(courseId, shift);
    const preEnrollmentWindow = this.getPreEnrollmentWindow(timezoneInfo, coursePricing);
    const campaignInfo = await this.getCampaignInfo(courseId);
    
    return this.calculatePricing({
        timezoneInfo,
        campaignInfo,
        preEnrollmentWindow,
        coursePricing,
    });
}
```

Timezone-aware calculations. Pre-enrollment windows. Campaign support. Dynamic pricing.

## Document Upload System

Supabase Storage integration:

```typescript
async uploadDocument(file: File, enrollmentId: string, documentType: string) {
    const cpf = await getStudentCpf(enrollmentId);
    const fileName = `${cpf}/${documentType}-${Date.now()}.${file.name.split('.').pop()}`;
    
    const { data, error } = await supabase.storage
        .from('documents')
        .upload(fileName, file, { contentType: file.type });
    
    if (error) throw error;
    
    await supabase.from('documents').insert({
        enrollment_id: enrollmentId,
        document_type: documentType,
        file_url: data.path,
        status: 'pending',
    });
}
```

CPF-based folder structure. Organized file storage. Database tracking. Status management.

## Payment Integration

Asaas checkout session:

```typescript
async createCheckoutSession(data: CheckoutData) {
    const pricing = await pricingService.getPricing({ courseId: data.courseId, shift: data.shift });
    const voucher = await this.fetchApplicableVoucher(data.studentCpf, data.selectedVoucher);
    const finalAmount = this.applyVoucherDiscount(pricing.amount, voucher);
    
    const session = await asaas.post('/checkout', {
        name: data.courseName,
        description: `Matrícula - ${data.courseName}`,
        billingType: 'PIX',
        value: finalAmount,
        customer: {
            name: data.studentName,
            email: data.studentEmail,
            cpf: data.studentCpf,
        },
        callbackUrl: CALLBACK_URLS.success,
    });
    
    return session;
}
```

Pricing calculation. Voucher application. Customer creation. Session management.

