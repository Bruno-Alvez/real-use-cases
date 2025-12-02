# Key Flows

## DRE Generation Flow

User requests DRE → service aggregates transactions → DRE response:

```python
async def get_dre(year: int) -> DREResponse:
    transactions = await self.repo.get_by_date_range(
        date(year, 1, 1), date(year, 12, 31)
    )
    
    income = sum(t.amount for t in transactions if t.type == 'income')
    expenses = sum(t.amount for t in transactions if t.type == 'expense')
    
    return DREResponse(
        year=year,
        total_revenue=income,
        total_expenses=expenses,
        net_income=income - expenses,
        monthly_breakdown=self._group_by_month(transactions)
    )
```

Flow: Request → date range query → aggregation → monthly breakdown → response.

## Recurring Rule Processing

Rule processing → date calculation → duplicate check → transaction creation:

```python
async def process_rule(self, rule_id: str) -> dict:
    rule = await self.get_by_id(rule_id)
    
    next_date = self._calculate_next_date(rule)
    
    if await self._transaction_exists(next_date, rule):
        return {"processed": False, "reason": "already_exists"}
    
    transaction = TransactionCreate(
        date=next_date,
        amount=rule.amount,
        category_id=rule.category_id,
        recurring_rule_id=rule.id
    )
    
    await self.transaction_service.create(transaction)
    await self.repository.update(rule_id, {
        "repetitions_completed": rule.repetitions_completed + 1,
        "last_processed_date": next_date
    })
    
    return {"processed": True}
```

Smart duplicate detection via transaction key. Lock mechanism prevents concurrent processing.

## Smart Rule Versioning

Rule update → version decision → transaction update:

```python
async def update_smart(self, rule_id: str, update: RecurringRuleSmartUpdate):
    if update.mode == 'all':
        # Update rule and all linked transactions
        await self.repository.update(rule_id, update)
        transactions = await self.get_rule_transactions(rule_id)
        for t in transactions:
            await self.transaction_service.update(t.id, update)
    else:
        # Create new version for future transactions
        new_version = await self.repository.create_rule_version(
            rule_id, update, datetime.now().date()
        )
        # Update future transactions to use new version
        future_transactions = [t for t in transactions if t.date >= today]
        for t in future_transactions:
            await self.transaction_service.update(t.id, {
                "recurring_rule_id": new_version.id
            })
```

Two modes: update all transactions or create version for future only.

## Transaction Creation

Request → validation → repository → response:

```python
async def create(self, transaction: TransactionCreate) -> TransactionResponse:
    # Validate category exists and matches transaction type
    category = await self.category_repo.get_by_id(transaction.category_id)
    if category.kind != transaction.type:
        raise ValidationError("Category type mismatch")
    
    return await self.repository.create(transaction)
```

Business rules enforced in service layer. Repository handles data persistence.

## Multi-Tenant Query

All queries automatically filter by user_id:

```python
class TransactionRepository:
    def __init__(self, user_id: str):
        self.user_id = user_id
    
    async def get_all(self):
        return self.client.table("transactions").select("*").eq(
            "user_id", self.user_id
        ).execute()
```

User context injected at repository initialization. No cross-tenant data access possible.

