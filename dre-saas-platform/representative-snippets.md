# Representative Snippets

## FastAPI Router with Dependency Injection

Clean route handler with user context:

```python
@router.get("/dre", response_model=DREResponse)
async def get_dre(
    year: int = Query(..., ge=2000, le=2100),
    user_id: str = Depends(get_current_user_id),
):
    service = AggregationService(user_id)
    return await service.get_dre(year)
```

## Service Layer - Business Logic

DRE aggregation with database-level calculations:

```python
class AggregationService:
    def __init__(self, user_id: str):
        self.user_id = user_id
        self.repo = TransactionRepository(user_id)
    
    async def get_dre(self, year: int) -> DREResponse:
        # Database aggregation for performance
        income, expenses = await asyncio.gather(
            self.repo.aggregate_amount(
                type='income',
                date_from=date(year, 1, 1),
                date_to=date(year, 12, 31)
            ),
            self.repo.aggregate_amount(
                type='expense',
                date_from=date(year, 1, 1),
                date_to=date(year, 12, 31)
            )
        )
        
        return DREResponse(
            year=year,
            total_revenue=income,
            total_expenses=expenses,
            net_income=income - expenses
        )
```

Database aggregation reduces memory usage and improves performance for large datasets (10k+ transactions).

## Repository Pattern

Type-safe data access with multi-tenant filtering:

```python
class TransactionRepository:
    def __init__(self, user_id: str):
        self.user_id = user_id
        self.client = get_supabase_client()
    
    async def get_by_date_range(self, date_from: date, date_to: date):
        return self.client.table("transactions").select("*").eq(
            "user_id", self.user_id
        ).gte("date", date_from.isoformat()).lte(
            "date", date_to.isoformat()
        ).execute()
```

## Recurring Rule Processing

Smart duplicate detection and transaction creation:

```python
async def process_rule(self, rule_id: str) -> dict:
    rule = await self.get_by_id(rule_id)
    next_date = self._calculate_next_date(rule)
    
    if await self._transaction_exists(next_date, rule):
        return {"processed": False}
    
    transaction = TransactionCreate(
        date=next_date,
        amount=rule.amount,
        category_id=rule.category_id,
        recurring_rule_id=rule.id
    )
    
    await self.transaction_service.create(transaction)
    await self.repository.update(rule_id, {
        "repetitions_completed": rule.repetitions_completed + 1
    })
    
    return {"processed": True}
```

## Smart Versioning

Rule update with version control:

```python
async def update_smart(self, rule_id: str, update: RecurringRuleSmartUpdate):
    if update.mode == 'all':
        await self.repository.update(rule_id, update)
        transactions = await self.get_rule_transactions(rule_id)
        for t in transactions:
            await self.transaction_service.update(t.id, update)
    else:
        new_version = await self.repository.create_rule_version(rule_id, update)
        future_transactions = [t for t in transactions if t.date >= today]
        for t in future_transactions:
            await self.transaction_service.update(t.id, {
                "recurring_rule_id": new_version.id
            })
```

## Pydantic Schema

Type-safe request/response models:

```python
class TransactionCreate(BaseModel):
    date: date
    type: Literal['income', 'expense']
    amount: PositiveFloat
    category_id: UUID
    account_id: UUID
    description: str | None = None

class DREResponse(BaseModel):
    year: int
    total_revenue: Decimal
    total_expenses: Decimal
    net_income: Decimal
    monthly_breakdown: list[MonthlyDRE]
```

## Next.js API Client

Type-safe HTTP client:

```typescript
export async function apiRequest<T>(
    endpoint: string,
    options: RequestOptions = {}
): Promise<T> {
    const response = await fetch(`${API_URL}${endpoint}`, {
        ...options,
        headers: {
            'Content-Type': 'application/json',
            ...(token && { Authorization: `Bearer ${token}` }),
        },
    });

    if (!response.ok) {
        const error = await response.json().catch(() => ({}));
        throw new Error(error.detail || `HTTP ${response.status}`);
    }

    return response.json();
}
```

