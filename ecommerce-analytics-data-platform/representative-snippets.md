# Representative Snippets

These snippets illustrate the architectural patterns used in this project. They are representative examples, not production source code.

## UPSERT pattern (all ETL Lambdas)

Every write in every Lambda uses this pattern. Running the same sync twice produces the same result.

```python
# Representative pattern from sync_clientes handler

def upsert_customers(records: list[dict], postgres_client) -> dict:
    if not records:
        return {"inserted": 0, "updated": 0}

    result = (
        postgres_client
        .table("dim_clientes")
        .upsert(records, on_conflict="id_cliente")
        .execute()
    )
    return {"upserted": len(result.data)}
```

The `on_conflict` column is always the primary key. If a record with that ID exists, all other columns are updated. If it does not, a new row is inserted. The pipeline never needs to check which case applies.

## Incremental cursor pattern

```python
# Representative cursor logic shared across WooCommerce sync Lambdas

def get_sync_cursor(postgres_client, table: str, cursor_field: str) -> str:
    result = postgres_client.table(table).select(cursor_field).order(cursor_field, desc=True).limit(1).execute()

    if result.data:
        return result.data[0][cursor_field]

    # No records: start from a safe baseline (pre-migration cutoff date)
    return "2026-01-03T15:00:00"


def fetch_since_cursor(api_client, endpoint: str, cursor: str) -> list[dict]:
    all_records = []
    page = 1

    while True:
        records = api_client.get(endpoint, params={"modified_after": cursor, "per_page": 100, "page": page})
        if not records:
            break
        all_records.extend(records)
        page += 1

    return all_records
```

## RLS policy structure

Three access levels, applied consistently across all 18 tables.

```sql
-- Representative RLS policy pattern

-- ETL pipelines use service_role (bypasses RLS entirely, not shown here)

-- Authenticated users can read customer data
CREATE POLICY "authenticated_read_clientes"
ON dim_clientes FOR SELECT
TO authenticated
USING (true);

-- Anonymous access only to non-PII tables (products, FAQ)
CREATE POLICY "anon_read_produtos"
ON dim_produtos FOR SELECT
TO anon
USING (true);

-- No anonymous access to customer PII
-- (No policy for dim_clientes + anon = blocked by default)
```

## FK constraint with validation before insert

`sync_vendas` checks that each order's customer exists before inserting it. If the customer is missing, it attempts to fetch and insert them first rather than silently skipping the order.

```python
# Representative FK validation from sync_vendas

def ensure_customer_exists(customer_id: int, postgres_client, wc_client) -> bool:
    existing = postgres_client.table("dim_clientes").select("id_cliente").eq("id_cliente", customer_id).execute()

    if existing.data:
        return True

    # Customer missing: fetch from WooCommerce and insert
    customer = wc_client.get(f"/customers/{customer_id}")
    if not customer:
        log("WARNING", "customer_not_found_in_source", customer_id=customer_id)
        return False

    postgres_client.table("dim_clientes").upsert(transform_customer(customer), on_conflict="id_cliente").execute()
    log("INFO", "customer_backfilled", customer_id=customer_id)
    return True
```

## Star Schema: RFV calculation

```sql
-- Representative RFV calculation pattern

INSERT INTO cluster_base_rfv (
  id_cliente,
  recencia_dias,
  frequencia_pedidos,
  valor_total,
  ticket_medio,
  id_produto_favorito,
  faixa_recencia,
  faixa_frequencia,
  faixa_valor
)
SELECT
  v.id_cliente,
  EXTRACT(DAY FROM NOW() - MAX(v.data_criacao))::INTEGER,
  COUNT(DISTINCT v.id_pedido),
  SUM(v.val_total),
  AVG(v.val_total),
  (
    SELECT item_id FROM fato_itens_vendas
    WHERE id_pedido IN (SELECT id_pedido FROM fato_vendas WHERE id_cliente = v.id_cliente)
    GROUP BY item_id ORDER BY SUM(quantidade) DESC LIMIT 1
  ),
  NTILE(4) OVER (ORDER BY MAX(v.data_criacao) DESC),
  NTILE(4) OVER (ORDER BY COUNT(DISTINCT v.id_pedido) DESC),
  NTILE(4) OVER (ORDER BY SUM(v.val_total) DESC)
FROM fato_vendas v
WHERE v.status IN ('completed', 'processing')
GROUP BY v.id_cliente;
```

## Product Bundles correction script

This was a one-time migration fix. The pattern is illustrative of how the issue was resolved.

```python
# Representative logic for WooCommerce Product Bundles correction

def extract_bundle_items(order: dict) -> list[dict]:
    items = []
    for line_item in order.get("line_items", []):
        if line_item.get("product_id") == 0:
            # Bundle container: parse _bundled_items metadata
            bundled_meta = next(
                (m["value"] for m in line_item.get("meta_data", []) if m["key"] == "_bundled_items"),
                None
            )
            if bundled_meta:
                for component in json.loads(bundled_meta):
                    items.append({
                        "id_pedido": order["id"],
                        "item_id": component["product_id"],
                        "item_id_pai": line_item["id"],
                        "quantidade": component.get("qty", 1),
                        "val_total_item": component.get("total", "0")
                    })
        else:
            items.append(transform_line_item(order["id"], line_item))
    return items
```

## Structured logging format

Every Lambda emits the same JSON structure. CloudWatch Insights queries across all Lambdas use the same field names.

```python
# Representative log output from sync_vendas completion

{
    "timestamp": "2026-02-16T04:15:23Z",
    "level": "INFO",
    "lambda": "client-analytics-sync-vendas",
    "event": "sync_completed",
    "records_processed": 89,
    "records_inserted": 62,
    "records_updated": 27,
    "records_skipped": 0,
    "execution_time_ms": 4312,
    "cursor_from": "2026-02-15T22:00:00Z",
    "cursor_next": "2026-02-16T04:00:00Z"
}
```
