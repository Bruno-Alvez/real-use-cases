# Representative Snippets

## Express App Setup

Clean initialization with middleware:

```typescript
const app = express();

app.use(helmet());
app.use(cors({ origin: config.cors.origin }));
app.use(express.json({ limit: '10mb' }));
app.use(requestLogger);
app.use(`/api/${config.apiVersion}`, apiLimiter);

app.use(`/api/${config.apiVersion}`, routes);
```

## Service Layer

Business logic with validation:

```typescript
async createVoucher(voucherData: CreateVoucherData, userId: string) {
    this.validateVoucherData(voucherData);
    const codigo = await this.generateUniqueCode();
    const dataExpiracao = this.calculateExpirationDate(voucherData.validade_dias);
    
    return await this.voucherRepository.create({
        ...voucherData,
        codigo,
        data_expiracao,
        criado_por: userId,
    });
}
```

## Repository Pattern

Type-safe data access:

```typescript
async findAll(filters?: VoucherFilters): Promise<Voucher[]> {
    let query = supabase.from('vouchers').select('*');
    
    if (filters?.cpf) query = query.ilike('cpf_aluno', `%${filters.cpf}%`);
    if (filters?.status) query = query.eq('status', filters.status);
    
    const { data } = await query;
    return this.mapToVoucherArray(data || []);
}
```

## Controller

HTTP request handling:

```typescript
async create(req: Request, res: Response) {
    try {
        const voucher = await voucherService.createVoucher(req.body, req.user.id);
        res.status(201).json({ success: true, data: voucher });
    } catch (error) {
        if (error instanceof AppError) {
            return res.status(error.statusCode).json({ success: false, error: error.message });
        }
        res.status(500).json({ success: false, error: 'Internal server error' });
    }
}
```

## Permission Check

Department-based access control:

```typescript
const hasPermission = (user: User, permission: string): boolean => {
    if (!user.department) return false;
    return user.department.permissions.includes(permission);
};

if (!hasPermission(user, PERMISSIONS.MANAGE_VOUCHERS)) {
    throw new AppError('Unauthorized', 403);
}
```

## React Context

Global state management:

```typescript
const AppContext = createContext<AppContextType | null>(null);

export const AppProvider: React.FC = ({ children }) => {
    const [user, setUser] = useState<User | null>(null);
    return (
        <AppContext.Provider value={{ user, setUser }}>
            {children}
        </AppContext.Provider>
    );
};
```

## Custom Hook

Data fetching with Supabase:

```typescript
export function useVouchers(filters?: VoucherFilters) {
    const [vouchers, setVouchers] = useState<Voucher[]>([]);
    const [loading, setLoading] = useState(true);
    
    useEffect(() => {
        fetchVouchers(filters).then(setVouchers).finally(() => setLoading(false));
    }, [filters]);
    
    return { vouchers, loading };
}
```

