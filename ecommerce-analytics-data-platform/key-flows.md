# Key Flows

## Parallel migration flow

```
Audit legacy environment
  └── Document all tables, constraints, and data quality issues
  └── Classify findings by severity (critical, high, medium)

Build production environment (clean slate)
  └── Design new schema with constraints enforced from the start
  └── Apply DDL: PKs, FKs, NUMERIC fields, RLS policies
  └── Enable pgvector extension

Migrate data by layer
  └── dim_clientes: filter 62,466 bot records, deduplicate by email
  └── dim_produtos: migrate with corrected types
  └── fato_vendas: migrate only orders with valid customer FK
  └── fato_itens_vendas: migrate and correct Product Bundles items
  └── cluster_base_rfv: recalculate from clean data
  └── Static tables: tipos_logisticos, servicos_logisticos, faq

Verify
  └── Row counts match expected (source minus eliminated records)
  └── FK constraints hold (no orphan insertion possible)
  └── RLS policies block anon access to PII
  └── Application connects to new environment, smoke test

Cut over
  └── Update connection strings in all services
  └── Legacy environment remains on standby
  └── Monitor for 48 hours before decommissioning legacy
```

## Incremental sync flow (all ETL Lambdas)

Each Lambda runs on an EventBridge schedule. The incremental sync logic is the same across all WooCommerce Lambdas:

```
Lambda invoked
  └── Load credentials from Secrets Manager (cached, 5min TTL)
  └── Query target table: SELECT MAX(data_modificacao) as cursor
  └── Call WooCommerce API: GET /endpoint?modified_after={cursor}&per_page=100
  └── For each page of results:
        └── Transform records to target schema
        └── UPSERT to PostgreSQL via ON CONFLICT DO UPDATE
        └── Log structured JSON: {records_processed, records_inserted, records_updated}
        └── Emit CloudWatch custom metric
  └── Fetch next page until no more results
  └── Log completion event
```

If the Lambda runs twice in the same window (manual trigger + scheduled), the UPSERT ensures no duplicates. If the Lambda fails mid-page, the next run picks up from the same cursor and reprocesses the partial page, which is safe because UPSERT is idempotent.

## WooCommerce Product Bundles incident

During the initial migration, the `fato_itens_vendas` table was missing 27,020 records after the first load, 47.7% of expected items. The foreign key constraint on `item_id` was rejecting rows with `product_id = 0`, which is what the WooCommerce Product Bundles plugin writes for bundle container items. They are virtual line items with no SKU that exist only to group the actual component products.

Resolution:

```
1. Fetch all 18,941 orders from WooCommerce API (complete dataset)
2. For each order, parse the `_bundled_items` metadata field (JSON-serialized)
3. Map each bundle component to its parent item_id
4. Insert a virtual product entry in dim_produtos with id = bundle_id
5. UPSERT all 31,138 corrected items via ON CONFLICT (id_pedido, item_id) DO UPDATE
6. Recalculate cluster_base_rfv (produto_favorito was wrong for bundle purchasers)
```

The total correction took 4 hours. After recalculation, 6 customers had their `produto_favorito` change because their most-purchased product had previously been recorded under the wrong item ID.

This incident reinforced the constraint validation approach: the FK on `item_id` caught the problem immediately on first load. Without the constraint, the missing records would have been invisible until someone noticed that product-level analysis was producing wrong numbers.

## RFV recalculation flow

`recalc_rfv` runs at 04:00 UTC daily. It does not use an incremental cursor. RFV scores are relative metrics: a customer's recency score depends on where they fall relative to the rest of the customer base. Partial updates produce incorrect rankings.

```sql
-- Simplified representative pattern
TRUNCATE cluster_base_rfv;

INSERT INTO cluster_base_rfv (
  id_cliente, recencia_dias, frequencia_pedidos, valor_total,
  ticket_medio, faixa_recencia, faixa_frequencia, faixa_valor
)
SELECT
  id_cliente,
  EXTRACT(DAY FROM NOW() - MAX(data_criacao)) as recencia_dias,
  COUNT(id_pedido) as frequencia_pedidos,
  SUM(val_total) as valor_total,
  AVG(val_total) as ticket_medio,
  NTILE(4) OVER (ORDER BY MAX(data_criacao) DESC) as faixa_recencia,
  NTILE(4) OVER (ORDER BY COUNT(id_pedido) DESC) as faixa_frequencia,
  NTILE(4) OVER (ORDER BY SUM(val_total) DESC) as faixa_valor
FROM fato_vendas
WHERE status IN ('completed', 'processing')
GROUP BY id_cliente;
```

`recalc_clusters` runs at 05:00 UTC, one hour after `recalc_rfv` completes. It reads the fresh RFV scores and runs KMeans clustering (k=4) to assign segment labels for the current month.

## Alert routing flow

```
CloudWatch alarm enters ALARM state
  └── Publishes to SNS topic: client-analytics-alertas
        ├── Email subscription: bruno.santos@nexlygroup.com
        ├── Email subscription: joao.passos@nexlygroup.com
        └── Lambda subscription: slack_notifier
              └── Parses SNS event
              └── Formats Slack Blocks message:
                    - Header: alarm name and state
                    - Section: Lambda affected, error description
                    - Section: Direct link to CloudWatch log stream
              └── POST to Slack webhook (from Secrets Manager)
              └── Posts to #project-client-analytics channel
```

The `slack_notifier` Lambda is the only one not triggered by EventBridge. It is event-driven, triggered by SNS, and stateless. It has no circuit breaker and no retry logic. If Slack is unreachable, the email subscriptions already delivered the alert.
