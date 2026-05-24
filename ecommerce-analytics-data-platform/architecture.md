# Architecture

## Migration strategy: Strangler Fig

The core architectural decision was to build a parallel production environment rather than correcting the existing database in place. In-place migration would have required modifying live tables, rewriting constraints on data that violated them, and coordinating with active services that were querying those tables during the process.

The parallel approach works differently: a new PostgreSQL project was provisioned with a clean schema, data was extracted from the legacy environment, validated and corrected, and loaded into production. At any point during the migration, if something went wrong, the fallback was a single DNS or connection string change to point services back to the legacy environment. That rollback option stayed open throughout.

The AI agent (conversational agent) and N8N automations continued running on the legacy environment without any modification. They were untouched until the new environment was ready and verified.

## Data warehouse: Star Schema

The schema was designed for analytical workloads, not transactional ones. The primary access pattern is aggregation across large time ranges, joining orders to customers and products, and running RFV calculations across the full customer base. Star Schema fits this well.

```
DIMENSIONAL LAYER    dim_clientes, dim_produtos
FACT LAYER           fato_vendas, fato_itens_vendas, fato_estoque
ANALYTICS LAYER      cluster_base_rfv, cluster_clientes, segmentos
OPERATIONAL LAYER    pedidos_bling, objetos_bling, bling_controle_datas
AUXILIARY            tipos_logisticos, servicos_logisticos, faq
CONVERSATIONAL       conversas_silvinha, mensagens_silvinha (pgvector)
```

Key schema constraints enforced by the SGBD:

- Every table has a primary key. This was non-negotiable given the UPSERT-based pipeline design.
- Foreign keys use `ON DELETE RESTRICT`, not CASCADE. Deleting a customer with associated orders should fail at the database level, not silently cascade or leave orphans.
- All monetary fields are `NUMERIC(12,2)`. The prior system had 16 monetary fields typed as TEXT. Aggregations now run natively in PostgreSQL.
- RLS is enabled on all 18 tables, with three role levels: `service_role` (full access for ETL pipelines), `authenticated` (read access to dimensional and fact tables), and `anon` (read access to products, stock, and FAQ for the conversational agent agent).

## ETL pipeline: AWS Lambda

Nine Lambda functions handle data synchronization. All run on Python 3.12 with a 512MB memory allocation and a 5-minute timeout. Eight run as Zip packages via a shared layer; one (`recalc_clusters`) runs as a container image because scikit-learn exceeds the Zip size limit.

```
sync_clientes       EventBridge 6h    WooCommerce customers    UPSERT via PK
sync_produtos       EventBridge 6h    WooCommerce products     UPSERT via PK
sync_vendas         EventBridge 3h    WooCommerce orders       UPSERT + FK check
sync_pedidos_bling  EventBridge 6h    Bling ERP orders         UPSERT via PK
sync_objetos_bling  EventBridge 6h    Bling logistics objects  UPSERT via PK
sync_logistica_bling EventBridge 2am  Bling carriers/services  UPSERT via PK
recalc_rfv          EventBridge 4am   SQL full refresh         TRUNCATE + INSERT
recalc_clusters     EventBridge 5am   KMeans (scikit-learn)    DELETE + INSERT
slack_notifier      SNS event         Alert formatting         Stateless
```

All ETL Lambdas share a common engineering pattern: retry with exponential backoff via Tenacity (max 3 attempts), rate limiting per API constraints, a circuit breaker that halts execution after 5 consecutive failures, and idempotent writes via `ON CONFLICT DO UPDATE`. No function can create duplicate records by running twice.

## Shared Layer

Seven Python modules are packaged as a Lambda Layer and shared across all functions.

```
config.py         Secrets Manager client with 5-minute TTL cache
logger.py         Structured JSON logging with execution context
metrics.py        CloudWatch PutMetricData wrapper
postgres_client.py  Async PostgreSQL client with connection pooling
woocommerce_client.py  REST API v3 with pagination and rate limiting
bling_client.py   OAuth2 refresh with distributed lock via DynamoDB
dynamodb_lock.py  Distributed lock for concurrent execution safety
```

## Observability

CloudWatch receives structured JSON logs from every Lambda. Metric filters extract numeric fields (`records_processed`, `errors`, `execution_time_ms`) and publish them as custom metrics. Twenty-eight alarms cover error rates, execution duration, zero-record conditions, Bling token refresh failures, and circuit breaker events.

Alerts route through an SNS topic to two email subscriptions and a `slack_notifier` Lambda that formats the event as a Slack Blocks message and posts it to the `#project-client-analytics` channel. The Slack webhook URL is stored in Secrets Manager, not in the Lambda environment variables.

## Application layer

The analytics web application runs on Next.js (frontend) and FastAPI (backend). The audit identified several critical issues in the original application that were corrected during the reconstruction:

- 15 endpoints with no authentication were secured with mandatory FastAPI middleware
- Client-side fetching of 100k records was replaced with server-side pagination using cursor-based queries
- A 3,963-line monolith file was broken into modules under 300 lines each
- N+1 query patterns (101 queries for 100 segments) were replaced with single JOIN queries
- Blocking `httpx.get()` calls in async FastAPI handlers were replaced with `await httpx.AsyncClient()`
