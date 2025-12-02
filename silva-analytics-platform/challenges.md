# Challenges

## Multi-Source Data Integration

**Problem**: Data from WordPress, Bling ERP, CRM, and AI models in different formats. Need unified view without data loss.

**Solution**: Supabase as central warehouse. ETL processes normalize data on ingestion. Common schema for customer, order, and campaign entities.

```python
async def sync_wordpress_data():
    wp_data = await fetch_wordpress_api()
    normalized = normalize_wordpress_orders(wp_data)
    supabase.table("orders").upsert(normalized).execute()

async def sync_bling_data():
    bling_data = await fetch_bling_api()
    normalized = normalize_bling_orders(bling_data)
    supabase.table("orders").upsert(normalized).execute()
```

Normalization functions map source-specific fields to common schema. Upsert prevents duplicates.

## RFM Calculation Performance

**Problem**: RFM calculation for thousands of customers is slow. Recalculating on every request is expensive. O(n) complexity for n customers.

**Solution**: Materialized view in PostgreSQL. Refresh on schedule (daily). Query materialized view instead of calculating on-the-fly.

**Trade-off Analysis**:
- **Materialized View**: Fast reads (~10ms vs ~500ms), requires refresh. Stale data for up to 24h acceptable for analytics.
- **Incremental Refresh**: Possible but adds complexity. Daily full refresh chosen for simplicity.
- **Index Strategy**: Composite index on (customer_id, cluster) for common query patterns.

```sql
CREATE MATERIALIZED VIEW rfm_analysis_mv AS
SELECT 
    customer_id,
    EXTRACT(DAY FROM (CURRENT_DATE - MAX(order_date))) as recency,
    COUNT(*) as frequency,
    SUM(order_value) as monetary,
    assign_cluster(recency, frequency, monetary) as cluster
FROM orders
GROUP BY customer_id;

CREATE INDEX ON rfm_analysis_mv(customer_id, cluster);

-- Refresh daily
REFRESH MATERIALIZED VIEW rfm_analysis_mv;
```

Materialized view refreshes once per day. Queries are fast. Indexes optimize lookups. Scales to 100k+ customers.

## Real-time Data Updates

**Problem**: Dashboard needs near-real-time updates. Polling every few seconds is inefficient.

**Solution**: React Query with smart caching. Stale-while-revalidate pattern. Background refetch on window focus.

```typescript
const { data } = useQuery({
    queryKey: ['analytics'],
    queryFn: () => api.get('/api/v1/analytics'),
    staleTime: 30000, // 30s
    refetchInterval: 60000, // 1min background refetch
    refetchOnWindowFocus: true,
});
```

Future: Supabase real-time subscriptions for instant updates.

## Date Range Filtering

**Problem**: Filtering large datasets by date range is slow. Client-side filtering doesn't scale.

**Solution**: Server-side filtering with database indexes. Date range passed as query params. Indexes on date columns.

```python
@app.get("/api/v1/analytics")
async def get_analytics(
    start_date: datetime | None = None,
    end_date: datetime | None = None,
):
    query = supabase.table("orders").select("*")
    
    if start_date:
        query = query.gte("date", start_date.isoformat())
    if end_date:
        query = query.lte("date", end_date.isoformat())
    
    return query.execute()
```

Database indexes on date columns. Query planner optimizes range scans.

## Cluster Assignment Logic

**Problem**: RFM cluster assignment rules complex. Hard to maintain in application code.

**Solution**: Database function for cluster assignment. Rules defined in SQL. Easy to update without code changes.

```sql
CREATE FUNCTION assign_cluster(recency INT, frequency INT, monetary NUMERIC)
RETURNS TEXT AS $$
BEGIN
    IF recency <= 30 AND frequency >= 10 AND monetary >= 1000 THEN
        RETURN 'Champions';
    ELSIF recency <= 60 AND frequency >= 5 THEN
        RETURN 'Fiéis';
    ELSIF recency <= 90 AND monetary >= 500 THEN
        RETURN 'Oportunistas';
    -- ... more rules
    ELSE
        RETURN 'Perdidos';
    END IF;
END;
$$ LANGUAGE plpgsql;
```

Business logic in database. Version controlled via migrations.

## Large Dataset Rendering

**Problem**: Rendering thousands of customers in table causes performance issues.

**Solution**: Virtual scrolling or pagination. React Query with pagination params. Server-side pagination.

```typescript
const { data } = useQuery({
    queryKey: ['customers', { page, limit }],
    queryFn: () => api.get('/api/v1/customers', {
        params: { page, limit: 50 }
    }),
});

// Virtual scrolling for large lists
<VirtualizedList items={data.items} />
```

Pagination reduces initial load. Virtual scrolling handles large lists smoothly.

## Type Safety Across Stack

**Problem**: Type mismatches between Python backend and TypeScript frontend.

**Solution**: Shared type definitions. OpenAPI schema generation from FastAPI. TypeScript types generated from OpenAPI.

```python
class CustomerResponse(BaseModel):
    id: str
    name: str
    cluster: str
    recency: int
    frequency: int
    monetary: float
```

FastAPI generates OpenAPI schema. TypeScript types generated from schema. Single source of truth.

## Error Handling in Async Operations

**Problem**: Multiple async data source queries. Single failure shouldn't break entire response. Timeout handling needed.

**Solution**: asyncio.gather with return_exceptions and timeout. Graceful degradation. Partial results returned. Circuit breaker pattern for repeated failures.

```python
async with asyncio.timeout(5.0):  # 5s timeout per source
    results = await asyncio.gather(
        get_wordpress_data(),
        get_bling_data(),
        get_crm_data(),
        return_exceptions=True,
    )

wordpress = results[0] if not isinstance(results[0], Exception) else {}
bling = results[1] if not isinstance(results[1], Exception) else {}
crm = results[2] if not isinstance(results[2], Exception) else {}

# Return partial results
return AnalyticsResponse(
    wordpress=wordpress,
    bling=bling,
    crm=crm,
    errors=[str(r) for r in results if isinstance(r, Exception)],
)
```

**Trade-off**: Partial results acceptable for analytics. Timeout prevents hanging requests. Circuit breaker prevents cascading failures. System remains operational if one source fails. Errors logged for investigation.

