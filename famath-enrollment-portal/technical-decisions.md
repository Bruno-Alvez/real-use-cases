# Technical Decisions

## Monorepo Architecture

Separated frontend and backend in single repository:

**Benefits**: Shared types, coordinated releases, single source of truth, easier development.

**Trade-offs**:
- **Pros**: Type safety, atomic changes, simplified dependency management
- **Cons**: Larger repository, slower CI/CD, potential for tight coupling
- **Decision**: Acceptable for current team size (<5 engineers). For larger teams, would consider polyrepo with shared packages.

```typescript
// Shared types
export interface Enrollment {
    id: string;
    protocol_number: string;
    student_id: string;
    status: string;
}
```

Type safety across stack. Single repository. Coordinated deployments. CI/CD runs tests for affected packages only.

## Express with Security Middleware

Express chosen with comprehensive security:

```typescript
app.use(helmet(SECURITY_CONFIG.helmet));
app.use(cors(SECURITY_CONFIG.cors));
app.use(rateLimit(SECURITY_CONFIG.rateLimit));
```

Helmet for security headers. CORS for cross-origin. Rate limiting for DDoS protection.

## Smart Pricing Engine

Date-based pricing with timezone awareness:

```typescript
const timezoneInfo = this.getTimezoneInfo(context.referenceDate);
const preEnrollmentWindow = this.getPreEnrollmentWindow(timezoneInfo, coursePricing);

if (preEnrollmentWindow.isActive) {
    return { amount: coursePricing.pre_enrollment_fee, feeType: 'pre_enrollment' };
}
```

Timezone-aware calculations. Pre-enrollment windows. Campaign support. Dynamic pricing rules.

## Supabase Storage for Documents

Supabase Storage chosen for document management:

- CPF-based folder structure
- Public access policies
- File size limits (10MB)
- Automatic organization

```typescript
const fileName = `${cpf}/${documentType}-${Date.now()}.${extension}`;
await supabase.storage.from('documents').upload(fileName, file);
```

Organized storage. Easy retrieval. Database tracking.

## Asaas Payment Integration

Asaas chosen for payment processing:

- PIX and Credit Card support
- Webhook-based confirmation
- Checkout session management
- Customer creation

```typescript
const session = await asaas.post('/checkout', {
    value: finalAmount,
    customer: { name, email, cpf },
    callbackUrl: CALLBACK_URLS.success,
});
```

Complete payment flow. Webhook integration. Session tracking.

## Voucher System

Voucher integration with pricing:

```typescript
async fetchApplicableVoucher(cpf: string, voucherCode?: string) {
    let query = supabase.from('vouchers')
        .eq('cpf_aluno', cpf)
        .eq('status', 'ativo');
    
    if (voucherCode) query = query.eq('codigo', voucherCode);
    
    const voucher = await query.limit(1).single();
    return voucher?.data_expiracao > new Date() ? voucher : null;
}
```

CPF-based lookup. Code validation. Expiration check. Discount application.

## React Context for State

Context API for global state:

```typescript
const AuthContext = createContext<AuthContextType | null>(null);

export const AuthProvider: React.FC = ({ children }) => {
    const [user, setUser] = useState<User | null>(null);
    return (
        <AuthContext.Provider value={{ user, setUser }}>
            {children}
        </AuthContext.Provider>
    );
};
```

Simple state management. No external library. Suitable for app scope.

