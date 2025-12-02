# Representative Snippets

## FastAPI Setup

Clean initialization with CORS:

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI(title="Silva Analytics API", version="0.1.0")
app.add_middleware(CORSMiddleware, allow_origins=settings.CORS_ORIGINS)
```

## JWT Authentication

Token generation and validation:

```python
def create_access_token(subject: str, expires_delta: timedelta | None = None) -> str:
    expire = datetime.now(timezone.utc) + (expires_delta or timedelta(minutes=1440))
    return jwt.encode(
        {"exp": expire, "sub": str(subject)},
        settings.SECRET_KEY,
        algorithm=settings.ALGORITHM
    )
```

## Supabase Queries

Type-safe data access:

```python
async def get_customers_by_cluster(cluster: str, month: str) -> list[dict]:
    response = supabase.table("rfm_analysis").select("*").eq(
        "cluster", cluster
    ).eq("mes_referencia", month).execute()
    return response.data or []
```

## API Client

Type-safe HTTP client:

```typescript
export async function apiRequest<T>(endpoint: string, options?: RequestOptions): Promise<T> {
    const response = await fetch(`${API_URL}${endpoint}`, {
        ...options,
        headers: {
            'Content-Type': 'application/json',
            ...(token && { Authorization: `Bearer ${token}` }),
        },
    });

    if (!response.ok) {
        const error = await response.json().catch(() => ({}));
        throw new Error(error.detail || `HTTP ${response.status}`);
    }

    return response.json();
}
```

## React Query Integration

Server state management:

```typescript
const { data, isLoading } = useQuery({
    queryKey: ['analytics', filters],
    queryFn: () => api.get<AnalyticsData>('/api/v1/analytics', { params: filters }),
    staleTime: 30000,
});
```

## RFM Cluster Analysis

Memoized calculations:

```typescript
const clusterDistribution = useMemo(() => {
    const counts: Record<string, number> = {};
    customers?.forEach(c => counts[c.cluster] = (counts[c.cluster] || 0) + 1);
    return Object.entries(counts).map(([name, count]) => ({
        label: name,
        value: count,
        color: clusterColors[name],
    })).sort((a, b) => b.value - a.value);
}, [customers]);
```

## Date Filtering

Client-side filtering:

```typescript
const filteredData = useMemo(() => {
    if (!startDate && !endDate) return monthlyData;
    return monthlyData.filter(data => {
        const dataDate = monthToDate(data.month);
        if (startDate && dataDate < new Date(startDate)) return false;
        if (endDate && dataDate > new Date(endDate)) return false;
        return true;
    });
}, [startDate, endDate]);
```
