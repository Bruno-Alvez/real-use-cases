# Key Flows

## Analytics Dashboard Load

User opens dashboard → API aggregates data → visualizations render:

```typescript
const { data, isLoading } = useQuery({
    queryKey: ['analytics', { startDate, endDate }],
    queryFn: () => api.get('/api/v1/analytics', { params: { start_date: startDate, end_date: endDate } }),
    staleTime: 30000,
});
```

Flow: Request → FastAPI service → Supabase queries → Aggregation → Response → React Query cache → UI update.

## Customer Segmentation Flow

User selects month → RFM data fetched → clusters calculated:

```typescript
const { data: customers } = useQuery({
    queryKey: ['customers', selectedMonth],
    queryFn: () => api.get('/api/v1/customers', { params: { month: selectedMonth } }),
});

const clusterDistribution = useMemo(() => {
    const counts: Record<string, number> = {};
    customers?.forEach(c => counts[c.cluster] = (counts[c.cluster] || 0) + 1);
    return Object.entries(counts).map(([name, count]) => ({
        label: name,
        value: count,
        color: clusterColors[name],
    }));
}, [customers]);
```

Backend queries Supabase RFM view. Frontend calculates distribution.

## Data Aggregation Flow

Multiple sources → Supabase warehouse → API response:

```python
async def get_analytics(filters: AnalyticsFilters) -> AnalyticsResponse:
    revenue, customers, wordpress, bling, crm = await asyncio.gather(
        get_revenue_metrics(filters),
        get_customer_metrics(filters),
        get_wordpress_metrics(filters),
        get_bling_metrics(filters),
        get_crm_metrics(filters),
        return_exceptions=True,
    )
    
    return AnalyticsResponse(
        revenue=revenue if not isinstance(revenue, Exception) else 0,
        customers=customers if not isinstance(customers, Exception) else 0,
        sources={"wordpress": wordpress, "bling": bling, "crm": crm}
    )
```

Parallel queries reduce latency. Graceful degradation on source failures.

## RFM Analysis Flow

Customer data → RFM calculation → Cluster assignment:

```python
async def calculate_rfm(customer_id: str, period: str) -> RFMResult:
    transactions = supabase.table("transactions").select("*").eq(
        "customer_id", customer_id
    ).gte("date", period_start).execute()
    
    recency = calculate_recency(transactions)
    frequency = calculate_frequency(transactions)
    monetary = calculate_monetary(transactions)
    cluster = assign_cluster(recency, frequency, monetary)
    
    supabase.table("rfm_analysis").upsert({
        "customer_id": customer_id,
        "recency": recency,
        "frequency": frequency,
        "monetary": monetary,
        "cluster": cluster,
        "period": period,
    }).execute()
    
    return RFMResult(recency, frequency, monetary, cluster)
```

RFM calculated per customer per period. Clusters: Champions, Fiéis, Oportunistas, Em Risco, Hibernando, Perdidos.

## Campaign Monitoring Flow

Campaign events → Storage → Aggregation:

```python
async def track_campaign_event(campaign_id: str, event_type: str, metadata: dict):
    supabase.table("campaign_events").insert({
        "campaign_id": campaign_id,
        "event_type": event_type,
        "metadata": metadata,
        "timestamp": datetime.now(timezone.utc),
    }).execute()

async def get_campaign_performance(campaign_id: str) -> CampaignMetrics:
    events = supabase.table("campaign_events").select("*").eq(
        "campaign_id", campaign_id
    ).execute()
    
    return CampaignMetrics(
        total_events=len(events.data),
        conversion_rate=calculate_conversion(events),
        revenue=calculate_revenue(events),
    )
```

Events stored with timestamps. Metrics calculated on-demand.

## Historical Snapshot Flow

User selects date range → Server-side filtering → Trends calculated:

```typescript
const { data } = useQuery({
    queryKey: ['analytics', { startDate, endDate }],
    queryFn: () => api.get('/api/v1/analytics', {
        params: { start_date: startDate, end_date: endDate }
    }),
    staleTime: 30000,
});

const momVariation = calculateMoM(
    data?.currentMonth.receita,
    data?.previousMonth.receita
);
```

Server-side filtering with database indexes. Reduces payload size. MoM calculations. Trend visualization.

## Data Export Flow

User clicks export → API generates file → Download:

```typescript
const handleExport = async () => {
    const data = await api.get('/api/v1/customers/export', { params: { format: 'csv' } });
    const blob = new Blob([data], { type: 'text/csv' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = `customers-${Date.now()}.csv`;
    a.click();
};
```

Backend formats data. Frontend triggers download.

