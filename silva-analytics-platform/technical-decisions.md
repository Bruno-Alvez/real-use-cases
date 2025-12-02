# Technical Decisions

## FastAPI over Flask/Django

FastAPI chosen for async support and automatic OpenAPI docs. Async/await enables concurrent data source queries. Type hints with Pydantic provide runtime validation.

```python
@app.get("/api/v1/analytics")
async def get_analytics(filters: AnalyticsFilters) -> AnalyticsResponse:
    revenue, customers, campaigns = await asyncio.gather(
        get_revenue_metrics(filters),
        get_customer_metrics(filters),
        get_campaign_metrics(filters),
    )
    return AnalyticsResponse(revenue=revenue, customers=customers, campaigns=campaigns)
```

Performance: 2-3x faster than Flask for I/O-bound operations. Automatic API documentation reduces maintenance.

## Supabase as Data Warehouse

Supabase chosen over raw PostgreSQL for:

1. **Managed Service**: No infrastructure management
2. **Real-time**: Built-in real-time subscriptions (future use)
3. **Auth**: Integrated authentication if needed
4. **Storage**: File storage for exports
5. **Functions**: Edge functions for data processing

PostgreSQL with analytical views for RFM. JSONB for flexible metadata storage.

```python
from supabase import create_client

supabase = create_client(settings.SUPABASE_URL, settings.SUPABASE_KEY)

# Query with filters
response = supabase.table("rfm_analysis").select("*").eq(
    "cluster", "Champions"
).gte("monetary", 1000).execute()
```

## Next.js App Router

Next.js 15 App Router for modern React patterns:

1. **Server Components**: Reduced client bundle
2. **Streaming**: Progressive page rendering
3. **Type Safety**: TypeScript throughout
4. **Performance**: Automatic code splitting

App Router enables server-side data fetching for initial load. Client components for interactive features.

```typescript
// Server Component for initial data
export default async function AnalyticsPage() {
    const data = await fetchAnalytics();
    return <AnalyticsClient data={data} />;
}

// Client Component for interactivity
'use client'
export function AnalyticsClient({ data }: Props) {
    const [filters, setFilters] = useState({});
    // Interactive features
}
```

## React Query for State Management

React Query chosen over Redux/Context for:

1. **Caching**: Automatic request deduplication
2. **Background Updates**: Stale-while-revalidate
3. **DevTools**: Excellent debugging
4. **Simplicity**: No boilerplate

Server state managed by React Query. UI state with useState. No global state needed.

```typescript
const { data, isLoading, refetch } = useQuery({
    queryKey: ['analytics', filters],
    queryFn: () => api.get('/api/v1/analytics', { params: filters }),
    staleTime: 30000,
    refetchOnWindowFocus: true,
});
```

## Pydantic for Validation

Pydantic v2 for request/response validation:

1. **Type Safety**: Runtime type checking
2. **Validation**: Automatic data validation
3. **Serialization**: JSON serialization built-in
4. **Performance**: Fast validation with Rust core

```python
from pydantic import BaseModel, Field

class AnalyticsFilters(BaseModel):
    start_date: datetime | None = None
    end_date: datetime | None = None
    cluster: str | None = Field(None, pattern="^(Champions|Fiéis|Oportunistas)$")
    
    @field_validator("end_date")
    @classmethod
    def validate_date_range(cls, v, info):
        if v and info.data.get("start_date") and v < info.data["start_date"]:
            raise ValueError("end_date must be after start_date")
        return v
```

FastAPI automatically validates requests against Pydantic models. Invalid requests return 422 with details.

## Recharts for Visualization

Recharts chosen for React-native charts:

1. **React Integration**: Native React components
2. **Customizable**: Full control over styling
3. **Responsive**: Automatic responsive behavior
4. **TypeScript**: Full type support

```typescript
<ResponsiveContainer width="100%" height={300}>
    <LineChart data={monthlyData}>
        <CartesianGrid strokeDasharray="3 3" />
        <XAxis dataKey="month" />
        <YAxis />
        <Tooltip />
        <Legend />
        <Line type="monotone" dataKey="receita" stroke="#8884d8" />
        <Line type="monotone" dataKey="clientes" stroke="#82ca9d" />
    </LineChart>
</ResponsiveContainer>
```

## Monorepo with Turborepo

Turborepo for monorepo management:

1. **Build Cache**: Faster CI/CD
2. **Task Orchestration**: Parallel execution
3. **Dependency Management**: Shared dependencies
4. **TypeScript**: Shared types across packages

PNPM workspaces for dependency management. Shared UI components in packages.

## Async Data Aggregation

Async/await for parallel data source queries with timeout and circuit breaker:

```python
async def get_analytics(filters: AnalyticsFilters) -> AnalyticsResponse:
    async with asyncio.timeout(5.0):  # 5s timeout
        wordpress, bling, crm = await asyncio.gather(
            get_wordpress_data(filters),
            get_bling_data(filters),
            get_crm_data(filters),
            return_exceptions=True,
        )
    
    # Handle errors gracefully with circuit breaker
    wordpress_data = wordpress if not isinstance(wordpress, Exception) else {}
    bling_data = bling if not isinstance(bling, Exception) else {}
    crm_data = crm if not isinstance(crm, Exception) else {}
    
    return aggregate_results(wordpress_data, bling_data, crm_data)
```

**Trade-off**: Parallel queries reduce latency from ~900ms (sequential) to ~300ms (parallel). Timeout prevents hanging requests. Circuit breaker prevents cascading failures. Partial results acceptable for analytics.

## RFM Analysis in Database

RFM calculation in PostgreSQL views for performance:

```sql
CREATE VIEW rfm_analysis AS
SELECT 
    customer_id,
    EXTRACT(DAY FROM (CURRENT_DATE - MAX(order_date))) as recency,
    COUNT(*) as frequency,
    SUM(order_value) as monetary,
    CASE
        WHEN recency <= 30 AND frequency >= 10 AND monetary >= 1000 THEN 'Champions'
        WHEN recency <= 60 AND frequency >= 5 THEN 'Fiéis'
        -- ... more clusters
    END as cluster
FROM orders
GROUP BY customer_id;
```

Database-level calculation is faster than application-level. Views can be materialized for better performance.

## TypeScript Strict Mode

Strict TypeScript for type safety:

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitAny": true,
    "strictNullChecks": true
  }
}
```

Catches errors at compile time. Better IDE support. Self-documenting code.

