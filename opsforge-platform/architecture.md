# Architecture

## System Overview

SaaS platform built as monorepo with three services: API (HTTP), Worker (background jobs), Web (Next.js dashboard). Shared Go packages for domain models, database, observability. PostgreSQL with Row Level Security for tenant isolation. Designed for horizontal scaling and multi-region deployment.

## Backend Architecture

### API Service

Gin HTTP server with structured logging (zap) and OpenTelemetry instrumentation. Clean architecture: controllers → usecases → repositories. Prometheus metrics exposed at `/metrics`.

```go
func main() {
    cfg := config.Load()
    log, _ := logger.New("opsforge-api")
    defer log.Sync()

    mp, _ := observability.InitOTel(ctx, "opsforge-api")
    defer mp.Shutdown(ctx)

    db, _ := database.NewPool(ctx)
    defer db.Close()

    router := gin.New()
    router.Use(gin.Recovery())
    router.Use(observabilityMiddleware(log))
    
    routes.Register(router, db, log)
    router.GET("/metrics", gin.WrapH(promhttp.Handler()))

    srv := &http.Server{Addr: ":" + cfg.Port, Handler: router}
    go srv.ListenAndServe()
    
    <-quit
    srv.Shutdown(ctx)
}
```

### Worker Service

Background job processor with ticker-based polling. Two job types: event delivery (5s interval) and monitor checks (30s interval). Graceful shutdown with context cancellation.

**Scalability Note**: Ticker-based polling chosen for simplicity at current scale (<1000 events/min). For higher throughput, would migrate to message queue (RabbitMQ/NATS) with worker pool pattern.

```go
type Dispatcher struct {
    db     *pgxpool.Pool
    log    *zap.Logger
    stopCh chan struct{}
}

func (d *Dispatcher) Start(ctx context.Context) {
    go d.runEventDelivery(ctx)
    go d.runMonitorChecks(ctx)
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

### Data Layer

PostgreSQL with pgx connection pooling. Domain models in `internal/domain`. Repository pattern for data access. RLS policies for tenant isolation.

```go
type Project struct {
    ID        uuid.UUID
    TenantID  uuid.UUID
    Name      string
    Slug      string
    CreatedAt time.Time
    UpdatedAt time.Time
}
```

Tables: tenants, projects, endpoints, webhook_events, deliveries, monitors, monitor_logs. Indexes on foreign keys and query patterns.

### Observability

Prometheus metrics for HTTP requests, worker jobs, database queries, event delivery latency. OpenTelemetry integration for distributed tracing. Grafana dashboards for visualization.

```go
var (
    HTTPRequestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Buckets: prometheus.DefBuckets,
        },
        []string{"method", "route", "status"},
    )
    
    EventDeliveryLatency = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "event_delivery_latency_seconds",
        },
        []string{"endpoint_id", "status"},
    )
)
```

## Frontend Architecture

Next.js 14 with App Router. Server and client components. shadcn/ui component library. Real-time dashboard with live metric updates. Type-safe API client.

Component structure:
- **Pages**: Route-level components (dashboard, projects, alerts, infrastructure, SLO)
- **Components**: Reusable UI (charts, tables, cards, metric cards)
- **Lib**: API client, mock services (dev), utilities

State management: React hooks with local state. React Query for server state caching (planned). WebSocket for real-time updates (planned).

## Data Flow

### Webhook Ingestion

1. Client POST to `/api/v1/webhooks`
2. API validates payload, creates `webhook_events` record
3. Returns 202 Accepted immediately
4. Worker picks up event on next polling cycle

### Event Delivery

1. Worker queries pending events (not in deliveries with status='delivered')
2. For each event, fetches endpoint configuration
3. HTTP POST to endpoint URL with payload
4. Records delivery attempt in `deliveries` table
5. On failure, schedules retry with exponential backoff
6. Updates metrics (latency, success/failure)

### Monitor Checks

1. Worker queries enabled monitors every 30s
2. HTTP GET to monitor URL
3. Records response time and status in `monitor_logs`
4. Updates monitor status if threshold exceeded

## Security

- Row Level Security (RLS) on all tables
- Tenant isolation via RLS policies
- UUID primary keys prevent enumeration
- Structured logging (no sensitive data)

## Deployment

Docker Compose for local development. Separate containers for API, Worker, PostgreSQL, Prometheus, Grafana. Production: API and Worker as separate services, shared PostgreSQL connection pool.

**Scaling Strategy**:
- **Current**: Single API instance, single Worker instance. Handles ~1000 events/min.
- **Horizontal**: API stateless, scales horizontally. Worker uses database locking for coordination.
- **Database**: Connection pool (25 per instance). Read replicas planned for >10k events/min.
- **Observability**: Metrics exported to Prometheus. Alerting on p95 latency >1s, error rate >1%.

