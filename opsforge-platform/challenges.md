# Challenges

## Database Connection Pool Exhaustion

**Problem**: High concurrency causes connection pool exhaustion. Worker and API compete for connections.

**Solution**: Separate connection pools with appropriate limits. API: 25 max, Worker: 10 max. Connection lifetime and idle timeout configured.

```go
cfg.MaxConns = 25
cfg.MinConns = 5
cfg.MaxConnLifetime = time.Hour
cfg.MaxConnIdleTime = time.Minute * 30
```

Connection pool monitoring via Prometheus. Alerts on high connection usage.

## Event Delivery Ordering

**Problem**: Ticker-based polling doesn't guarantee delivery order. Multiple workers could process same event. Race conditions on concurrent access.

**Solution**: Database-level locking with SELECT FOR UPDATE SKIP LOCKED. Ensures one worker processes each event. Maintains FIFO ordering per endpoint.

```sql
SELECT id, tenant_id, endpoint_id, event_type, payload, created_at
FROM webhook_events
WHERE id NOT IN (SELECT event_id FROM deliveries WHERE status = 'delivered')
ORDER BY created_at ASC
FOR UPDATE SKIP LOCKED
LIMIT 10
```

**Trade-off**: Database locking adds latency (~5-10ms per query) but ensures correctness. SKIP LOCKED allows concurrent workers without blocking. For higher throughput, would use partitioned message queue with per-endpoint ordering keys.

## Retry Backoff Strategy

**Problem**: Failed deliveries need retry, but immediate retry wastes resources.

**Solution**: Exponential backoff with jitter. Max attempts limit prevents infinite retries.

```go
func calculateBackoff(attempts int) time.Duration {
    base := time.Duration(math.Pow(2, float64(attempts))) * time.Second
    jitter := time.Duration(rand.Intn(1000)) * time.Millisecond
    return base + jitter
}
```

Jitter prevents thundering herd. Max attempts: 5. After max, event marked as failed.

## Multi-Tenant Isolation

**Problem**: SaaS platform requires strict tenant isolation. RLS policies need tenant context. How to set tenant_id in queries?

**Solution**: Middleware extracts tenant from JWT token. Database session variable stores tenant_id. All queries automatically filtered by RLS policies.

```go
func tenantMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        tenantID := extractTenantFromToken(c)
        c.Set("tenant_id", tenantID)
        
        // Set in database session
        db.Exec(ctx, "SET app.tenant_id = $1", tenantID)
        c.Next()
    }
}
```

RLS policies reference session variable. No application-level filtering needed.

## Worker Graceful Shutdown

**Problem**: Worker must finish in-flight jobs before shutdown. Ticker continues during shutdown.

**Solution**: Stop channel signals goroutines. Context cancellation for in-flight operations.

```go
func (d *Dispatcher) Stop() {
    close(d.stopCh)
    d.log.Info("worker dispatcher stopped")
}

func (d *Dispatcher) runEventDelivery(ctx context.Context) {
    ticker := time.NewTicker(5 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-d.stopCh:
            return
        case <-ticker.C:
            d.processEventDelivery(ctx)
        }
    }
}
```

5-second timeout for graceful shutdown. In-flight jobs complete or timeout.

## Metrics Cardinality

**Problem**: SaaS platform with many tenants and endpoints. High-cardinality labels cause Prometheus memory issues. Endpoint IDs as labels explode cardinality (10k endpoints × 3 statuses = 30k series).

**Solution**: Cardinality budget per metric. Use endpoint_id only for critical metrics. Aggregate by tenant/project for dashboards. Sampling for high-volume endpoints.

```go
// High cardinality - use sparingly, with sampling
if shouldSample(endpointID) {
    EventDeliveryLatency.WithLabelValues(endpointID, status).Observe(latency)
}

// Low cardinality - safe
HTTPRequestCount.WithLabelValues(method, route, status).Inc()

// Aggregated - safe
EventDeliveryLatencyAgg.WithLabelValues(tenantID, status).Observe(latency)
```

**Cardinality Budget**: Max 10k series per metric. Sampling rate: 1% for endpoints with >1000 events/day. Grafana dashboards aggregate by tenant/project. Individual endpoint metrics via separate query API.

## Database Query Performance

**Problem**: Pending events query scans large table. No index on delivery status.

**Solution**: Composite index on (event_id, status) in deliveries. Partial index for pending status.

```sql
CREATE INDEX idx_deliveries_event_status 
ON deliveries(event_id, status) 
WHERE status != 'delivered';
```

Query uses index efficiently. LIMIT 10 prevents large result sets.

## Error Handling in Workers

**Problem**: Worker errors shouldn't crash service. Need retry and dead-letter queue.

**Solution**: Structured error handling. Transient errors retry. Permanent errors mark as failed.

```go
func (d *Dispatcher) deliverEvent(ctx context.Context, event *domain.WebhookEvent) error {
    endpoint, err := d.getEndpoint(ctx, event.EndpointID)
    if err != nil {
        return fmt.Errorf("get endpoint: %w", err)
    }

    resp, err := http.Post(endpoint.URL, "application/json", bytes.NewReader(payload))
    if err != nil {
        return fmt.Errorf("http post: %w", err) // Transient, retry
    }
    defer resp.Body.Close()

    if resp.StatusCode >= 500 {
        return fmt.Errorf("server error: %d", resp.StatusCode) // Transient, retry
    }

    if resp.StatusCode >= 400 {
        return ErrPermanentFailure // Don't retry
    }

    return nil
}
```

Permanent failures skip retry. Transient failures retry with backoff.

