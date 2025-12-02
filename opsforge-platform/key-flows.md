# Key Flows

## Webhook Event Ingestion

Client sends webhook → API validates → event queued → 202 response:

```go
func (c *WebhookController) Create(ctx *gin.Context) {
    var req WebhookRequest
    if err := ctx.ShouldBindJSON(&req); err != nil {
        ctx.JSON(400, gin.H{"error": err.Error()})
        return
    }

    event := &domain.WebhookEvent{
        ID:         uuid.New(),
        TenantID:   req.TenantID,
        EndpointID: req.EndpointID,
        EventType:  req.EventType,
        Payload:    req.Payload,
        CreatedAt:  time.Now(),
    }

    if err := c.usecase.CreateEvent(ctx, event); err != nil {
        ctx.JSON(500, gin.H{"error": "failed to create event"})
        return
    }

    ctx.JSON(202, gin.H{"id": event.ID, "status": "queued"})
}
```

Flow: Validate → Create event record → Return immediately → Worker processes async.

## Event Delivery Flow

Worker polls → fetches pending events → delivers → records attempt:

```go
func (d *Dispatcher) processEventDelivery(ctx context.Context) {
    rows, _ := d.db.Query(ctx, `
        SELECT id, tenant_id, endpoint_id, event_type, payload, created_at
        FROM webhook_events
        WHERE id NOT IN (
            SELECT event_id FROM deliveries WHERE status = 'delivered'
        )
        ORDER BY created_at ASC
        LIMIT 10
    `)
    defer rows.Close()

    for rows.Next() {
        var event domain.WebhookEvent
        rows.Scan(&event.ID, &event.TenantID, &event.EndpointID, 
                  &event.EventType, &event.Payload, &event.CreatedAt)

        if err := d.deliverEvent(ctx, &event); err != nil {
            d.log.Error("delivery failed", zap.Error(err))
            d.recordRetry(ctx, &event)
        } else {
            d.recordSuccess(ctx, &event)
        }
    }
}
```

Delivery: Fetch endpoint config → HTTP POST → Record delivery → Update metrics.

## Retry Logic

Failed delivery → exponential backoff → retry queue:

```go
func (d *Dispatcher) recordRetry(ctx context.Context, event *domain.WebhookEvent) {
    delivery := &domain.Delivery{
        ID:         uuid.New(),
        EventID:    event.ID,
        EndpointID: event.EndpointID,
        Status:     "failed",
        Attempts:   1,
        CreatedAt:  time.Now(),
    }

    d.db.Exec(ctx, `
        INSERT INTO deliveries (id, event_id, endpoint_id, status, attempts)
        VALUES ($1, $2, $3, $4, $5)
    `, delivery.ID, delivery.EventID, delivery.EndpointID, 
       delivery.Status, delivery.Attempts)

    nextRetry := time.Now().Add(time.Duration(math.Pow(2, float64(delivery.Attempts))) * time.Second)
    d.scheduleRetry(ctx, event.ID, nextRetry)
}
```

Exponential backoff: 2^attempts seconds. Max attempts configurable.

## Monitor Check Flow

Worker polls monitors → HTTP check → log result → update metrics:

```go
func (d *Dispatcher) processMonitorChecks(ctx context.Context) {
    rows, _ := d.db.Query(ctx, `
        SELECT id, tenant_id, project_id, name, url, interval, enabled
        FROM monitors
        WHERE enabled = true
    `)
    defer rows.Close()

    for rows.Next() {
        var monitor domain.Monitor
        rows.Scan(&monitor.ID, &monitor.TenantID, &monitor.ProjectID,
                  &monitor.Name, &monitor.URL, &monitor.Interval, &monitor.Enabled)

        start := time.Now()
        resp, err := http.Get(monitor.URL)
        duration := time.Since(start)

        status := "success"
        statusCode := 200
        if err != nil || resp.StatusCode >= 400 {
            status = "failed"
            if resp != nil {
                statusCode = resp.StatusCode
            }
        }

        d.recordMonitorLog(ctx, &monitor, status, statusCode, duration)
        
        // Update Prometheus metrics
        observability.MonitorCheckDuration.WithLabelValues(
            monitor.ID.String(), status,
        ).Observe(duration.Seconds())
    }
}
```

Check every 30s. Log response time and status. Update metrics for dashboard visualization. Alert on consecutive failures (planned).

## Metrics Collection

HTTP middleware and worker jobs instrument all operations for observability:

```go
func observabilityMiddleware(log *zap.Logger) gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        c.Next()
        
        duration := time.Since(start).Seconds()
        status := fmt.Sprintf("%d", c.Writer.Status())
        method := c.Request.Method
        route := c.FullPath()
        
        observability.HTTPRequestDuration.WithLabelValues(
            method, route, status,
        ).Observe(duration)
        
        observability.HTTPRequestCount.WithLabelValues(
            method, route, status,
        ).Inc()
    }
}
```

Prometheus scrapes `/metrics` endpoint. Grafana dashboards visualize metrics. OpenTelemetry provides distributed tracing. Full observability stack for SaaS platform.

## Multi-Tenant Isolation

RLS policies enforce tenant boundaries for SaaS security:

```sql
ALTER TABLE webhook_events ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can view their own webhook_events" 
ON webhook_events FOR SELECT 
USING (tenant_id = current_setting('app.tenant_id')::uuid);
```

Middleware sets tenant context from JWT. All queries automatically filtered by tenant_id. Zero cross-tenant data access. Critical for SaaS compliance and security.

