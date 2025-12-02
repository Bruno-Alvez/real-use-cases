# Technical Decisions

## Go for Backend

Go chosen for performance and concurrency in observability workloads. HTTP handlers handle thousands of requests. Worker goroutines process deliveries and monitoring checks concurrently. Low memory footprint critical for SaaS cost efficiency.

```go
router := gin.New()
router.Use(gin.Recovery())
router.Use(observabilityMiddleware(log))
```

Gin framework for HTTP routing. Fast, minimal dependencies. Middleware for logging and metrics.

## Ticker-Based Polling vs Message Queue

Ticker-based polling chosen over RabbitMQ/Kafka for simplicity:

1. **Simplicity**: No additional infrastructure
2. **Reliability**: Database as single source of truth
3. **Cost**: No message broker to operate
4. **PostgreSQL**: Handles concurrency well

Trade-off: Higher database load. Mitigated by:
- Efficient queries with indexes
- Batch processing (LIMIT 10)
- Connection pooling

```go
ticker := time.NewTicker(5 * time.Second)
for {
    select {
    case <-ticker.C:
        d.processEventDelivery(ctx)
    }
}
```

Future: Can migrate to message queue if scale requires.

## PostgreSQL with RLS

PostgreSQL chosen over NoSQL for:

1. **ACID**: Delivery guarantees require transactions
2. **RLS**: Built-in multi-tenant isolation
3. **JSONB**: Flexible payload storage
4. **Mature**: Proven reliability

RLS policies enforce tenant boundaries at database level. No application-level filtering needed.

```sql
CREATE POLICY "Users can view their own events" 
ON webhook_events FOR SELECT 
USING (tenant_id = current_setting('app.tenant_id')::uuid);
```

## pgx Connection Pooling

pgx chosen over GORM for:

1. **Performance**: Lower overhead
2. **Control**: Explicit query control
3. **Type Safety**: Compile-time query validation possible
4. **Connection Pooling**: Built-in pool management

```go
db, err := pgxpool.New(ctx, cfg.DatabaseURL)
```

Connection pool configured for production load. Automatic retry on connection failures.

## Prometheus + OpenTelemetry

Prometheus for metrics, OpenTelemetry for tracing:

1. **Standard**: Industry-standard observability
2. **Grafana**: Rich visualization
3. **Export**: Metrics exportable to other systems
4. **Low Overhead**: Efficient metric collection

```go
HTTPRequestDuration.WithLabelValues(method, route, status).Observe(duration)
```

Histograms for latency, counters for throughput. Labels for filtering.

## Next.js App Router

Next.js 14 App Router for observability dashboard:

1. **Server Components**: Reduced client bundle for faster load times
2. **Type Safety**: TypeScript throughout for API client and components
3. **Performance**: Automatic code splitting for large dashboards
4. **DX**: Fast refresh, excellent tooling

shadcn/ui for consistent component library. Tailwind for rapid UI development. Real-time updates via polling (WebSocket planned).

## Monorepo with Turborepo

Turborepo for monorepo management:

1. **Shared Code**: Common packages (UI, SDKs)
2. **Build Cache**: Faster CI/CD
3. **Dependency Management**: Single lockfile
4. **TypeScript**: Shared types across packages

Packages: `ui` (shared components), `sdk-ts` (TypeScript client), `sdk-go` (Go client).

## Repository Pattern

Repository pattern for data access:

1. **Testability**: Easy to mock
2. **Separation**: Business logic separate from data access
3. **Flexibility**: Can swap implementations

```go
type ProjectRepository struct {
    db *pgxpool.Pool
}

func (r *ProjectRepository) GetByID(ctx context.Context, id uuid.UUID) (*domain.Project, error) {
    // Query implementation
}
```

Usecase layer orchestrates business logic. Controllers handle HTTP concerns.

## Graceful Shutdown

Context-based shutdown for clean termination:

```go
quit := make(chan os.Signal, 1)
signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
<-quit

ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

srv.Shutdown(ctx)
dispatcher.Stop()
```

5-second timeout for in-flight requests. Worker stops accepting new jobs.

## Structured Logging with zap

zap chosen over standard log:

1. **Performance**: Zero-allocation in hot paths
2. **Structured**: JSON output for parsing
3. **Levels**: Fine-grained log levels
4. **Fields**: Rich context in logs

```go
log.Info("event delivered", 
    zap.String("event_id", event.ID.String()),
    zap.String("endpoint_id", endpoint.ID.String()),
    zap.Duration("latency", duration),
)
```

Production: JSON format. Development: Console format.

