# Technical Decisions

## FastAPI for Backend

FastAPI chosen for async support and automatic OpenAPI docs. Type hints with Pydantic provide validation. Fast for I/O-bound operations.

```python
@router.get("/dre", response_model=DREResponse)
async def get_dre(year: int, user_id: str = Depends(get_current_user_id)):
    service = AggregationService(user_id)
    return await service.get_dre(year)
```

## Layered Architecture

Router → Service → Repository pattern for separation of concerns:

1. **Routers**: HTTP concerns, validation, auth
2. **Services**: Business logic, orchestration
3. **Repositories**: Data access, Supabase queries

Benefits: Testability, maintainability, clear boundaries.

## Supabase with RLS

Supabase chosen for managed PostgreSQL with RLS. Multi-tenant isolation at database level. No application-level filtering needed.

```python
# Repository automatically filters by user_id
query = self.client.table("transactions").select("*").eq("user_id", self.user_id)
```

RLS policies enforce isolation. Service role key for admin operations.

## Smart Recurring Versioning

Two update modes for recurring rules:

1. **Update all**: Changes apply to all past and future transactions
2. **Version future**: Creates new rule version, only future transactions updated

```python
if mode == 'all':
    await update_rule_and_transactions(rule_id, update)
else:
    new_version = await create_rule_version(rule_id, update)
    await update_future_transactions(new_version)
```

Enables rule changes without breaking historical data.

## Transaction Key for Duplicates

Unique key prevents duplicate recurring transactions:

```python
def _generate_transaction_key(date, amount, description, category_id, account_id):
    return f"{date}-{amount}-{description.lower()}-{category_id}-{account_id}"
```

**Trade-off**: String-based key is simple but has collision risk (very low probability). Alternative: composite unique index in database. Chosen for simplicity and performance (O(1) cache lookup).

Cache and database checks prevent duplicates. Handles edge cases (same day, same amount). Database unique constraint as final safeguard.

## Next.js App Router

Next.js 14 App Router for modern React patterns. Server components for initial load. Client components for interactivity.

Type-safe API client with error handling. Custom hooks for data fetching.

## Pydantic for Validation

Pydantic v2 for request/response validation:

```python
class TransactionCreate(BaseModel):
    date: date
    type: Literal['income', 'expense']
    amount: PositiveFloat
    category_id: UUID
    account_id: UUID
```

Runtime validation with clear error messages. Type hints for IDE support.

