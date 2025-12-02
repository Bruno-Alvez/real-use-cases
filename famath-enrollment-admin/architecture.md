# Architecture

## System Overview

Monorepo with React frontend and Express backend. Layered architecture: Routes → Controllers → Services → Repositories → Supabase. Department-based permissions with RLS.

## Backend Architecture

### Express Application

Express with security middleware and structured routing:

```typescript
const app = express();

app.use(helmet());
app.use(cors({ origin: config.cors.origin }));
app.use(express.json({ limit: '10mb' }));
app.use(requestLogger);
app.use(`/api/${config.apiVersion}`, apiLimiter);

app.use(`/api/${config.apiVersion}`, routes);
```

Security headers, CORS, rate limiting, request logging.

### Layered Architecture

**Routes**: API endpoint definitions  
**Controllers**: HTTP request handling, validation  
**Services**: Business logic, orchestration  
**Repositories**: Data access, Supabase queries

```typescript
// Route
router.post('/vouchers', authMiddleware, voucherController.create);

// Controller
async create(req: Request, res: Response) {
    const voucher = await voucherService.createVoucher(req.body, req.user.id);
    res.status(201).json({ success: true, data: voucher });
}

// Service
async createVoucher(voucherData: CreateVoucherData, userId: string) {
    this.validateVoucherData(voucherData);
    const codigo = await this.generateUniqueCode();
    return await this.voucherRepository.create({ ...voucherData, codigo });
}

// Repository
async create(data: CreateVoucherData) {
    const { data: voucher, error } = await supabase
        .from('vouchers')
        .insert(data)
        .select()
        .single();
    return voucher;
}
```

Clear separation of concerns. Testable layers. Type-safe interfaces.

## Frontend Architecture

React with Context API for state management. Component-based UI. Custom hooks for data fetching.

Structure:
- **components/**: UI components (Kanban, Voucher, Commercial)
- **hooks/**: Custom hooks (useCandidates, useVouchers, usePermissions)
- **services/**: Supabase client wrappers
- **context/**: AppContext for global state
- **types/**: TypeScript type definitions

```typescript
const AppContent: React.FC = () => {
    const { user } = useAppContext();
    if (!user) return <Login />;
    return <Layout />;
};
```

Context-based authentication. Protected routes. Component composition.

## Data Layer

Supabase PostgreSQL with RLS:

```sql
CREATE POLICY "Users can view vouchers from their department"
ON vouchers FOR SELECT
USING (
    EXISTS (
        SELECT 1 FROM collaborators
        WHERE collaborators.user_id = auth.uid()
        AND collaborators.department_id = vouchers.department_id
    )
);
```

Department-based RLS policies. Multi-tenant isolation. Service role key for admin operations.

## Kanban Board

Drag & drop enrollment visualization:

```typescript
const columns = [
    { id: 'new', title: 'Novos', candidates: [] },
    { id: 'contacted', title: 'Contatados', candidates: [] },
    { id: 'enrolled', title: 'Matriculados', candidates: [] },
];

const handleDragEnd = (result: DropResult) => {
    if (!result.destination) return;
    updateCandidateStatus(result.draggableId, result.destination.droppableId);
};
```

Column-based organization. Status transitions. Real-time updates via Supabase subscriptions.

## Permission System

Department-based permissions:

```typescript
export const PERMISSIONS = {
    VIEW_CANDIDATES: 'view_candidates',
    EDIT_CANDIDATES: 'edit_candidates',
    MANAGE_VOUCHERS: 'manage_vouchers',
    VIEW_ANALYTICS: 'view_analytics',
};

const hasPermission = (user: User, permission: string): boolean => {
    return user.department?.permissions?.includes(permission) ?? false;
};
```

Granular permissions. Department-level access control. RLS enforcement.

