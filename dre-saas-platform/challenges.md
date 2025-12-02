# Challenges

## Recurring Rule Versioning

**Problem**: Updating recurring rules affects past transactions. Need to preserve historical accuracy while allowing rule changes.

**Solution**: Smart versioning with two modes. "Update all" modifies rule and linked transactions. "Version future" creates new rule version, only future transactions updated.

```python
if mode == 'all':
    await update_rule_and_transactions(rule_id, update)
else:
    new_version = await create_rule_version(rule_id, update)
    await update_future_transactions(new_version)
```

Historical data preserved. Future transactions use new version.

## Duplicate Transaction Prevention

**Problem**: Recurring rules can create duplicate transactions if processed multiple times.

**Solution**: Transaction key generation for duplicate detection. Cache and database checks prevent duplicates.

```python
def _generate_transaction_key(date, amount, description, category_id, account_id):
    return f"{date}-{amount}-{description.lower()}-{category_id}-{account_id}"

async def _transaction_exists(self, date, rule, cache: Set[str]) -> bool:
    key = self._generate_transaction_key(...)
    if key in cache:
        return True
    # Check database
    return await self.repo.exists(key)
```

Processing lock prevents concurrent execution. Cache reduces database queries.

## DRE Aggregation Performance

**Problem**: Aggregating thousands of transactions for yearly DRE is slow. Loading all transactions into memory doesn't scale.

**Solution**: Database-level aggregation with GROUP BY. Indexes on (user_id, date, type) for efficient queries. Parallel aggregation for income/expense.

```sql
CREATE INDEX idx_transactions_user_date_type 
ON transactions(user_id, date, type);

-- Aggregation query
SELECT 
    type,
    SUM(amount) as total,
    COUNT(*) as count
FROM transactions
WHERE user_id = $1 
  AND date >= $2 
  AND date <= $3
GROUP BY type;
```

**Performance**: Query time reduced from ~500ms (in-memory) to ~50ms (database aggregation) for 10k transactions. Memory usage constant regardless of dataset size.

## Multi-Tenant Isolation

**Problem**: Ensure zero cross-tenant data access in SaaS platform.

**Solution**: User context injected at repository initialization. All queries filter by user_id. RLS policies as additional layer.

```python
class TransactionRepository:
    def __init__(self, user_id: str):
        self.user_id = user_id
    
    async def get_all(self):
        return self.client.table("transactions").eq("user_id", self.user_id)
```

Double protection: application-level filtering + RLS policies.

## Transaction Type Validation

**Problem**: Category type must match transaction type. Validation needed at service layer.

**Solution**: Service validates category before transaction creation. Clear error messages.

```python
async def create(self, transaction: TransactionCreate):
    category = await self.category_repo.get_by_id(transaction.category_id)
    if category.kind != transaction.type:
        raise ValidationError("Category type mismatch")
    return await self.repository.create(transaction)
```

Business rules enforced in service layer. Repository handles data access only.

