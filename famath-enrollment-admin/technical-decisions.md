# Technical Decisions

## Layered Architecture

Routes → Controllers → Services → Repositories pattern:

**Benefits**: Separation of concerns, testability, maintainability, scalability.

```typescript
// Clear boundaries
Route → Controller → Service → Repository → Database
```

Each layer has single responsibility. Easy to test in isolation. Type-safe interfaces.

## Express over Fastify

Express chosen for ecosystem and team familiarity:

- Large ecosystem
- Extensive middleware
- Team experience
- Good TypeScript support

Helmet for security. CORS for cross-origin. Rate limiting for protection.

## Supabase with RLS

Supabase chosen for managed PostgreSQL with RLS:

```sql
CREATE POLICY "Department isolation"
ON vouchers FOR SELECT
USING (department_id IN (
    SELECT department_id FROM collaborators WHERE user_id = auth.uid()
));
```

Database-level security. Multi-tenant isolation. Real-time subscriptions.

## React Context for State

Context API for global state management:

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

Simple state management. No external library needed. Suitable for app scope.

## TypeScript Strict Mode

Strict TypeScript configuration:

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitAny": true
  }
}
```

Type safety. Better IDE support. Catch errors at compile time.

## Vite over Create React App

Vite chosen for faster development:

- Faster HMR
- Better build performance
- Modern tooling
- Smaller bundle size

Development experience improved. Faster builds.

## Monorepo Structure

PNPM workspaces for monorepo:

```
gerenciamento-matriculas-famath/
├── frontend/
├── backend/
├── shared/
└── database/
```

Shared types and constants. Single repository. Coordinated releases.

## Department-Based Permissions

Granular permissions per department:

```typescript
const PERMISSIONS = {
    VIEW_CANDIDATES: 'view_candidates',
    EDIT_CANDIDATES: 'edit_candidates',
    MANAGE_VOUCHERS: 'manage_vouchers',
};
```

Flexible access control. Department-level isolation. Easy to extend.

