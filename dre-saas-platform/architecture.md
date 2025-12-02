# Architecture

## System Overview

Monorepo with FastAPI backend and Next.js frontend. Layered architecture: API routers → services → repositories → Supabase. Multi-tenant isolation via RLS and user_id filtering.

## Backend Architecture

### FastAPI Application

FastAPI with Pydantic validation. Router-based API organization. JWT authentication via dependency injection.

```python
from fastapi import FastAPI
from app.api.v1 import api_router

app = FastAPI(title="DRE SaaS Nexly API", version="0.1.0")
app.include_router(api_router, prefix="/api/v1")
```

### Layered Architecture

**Routers**: HTTP request handling, validation, authentication  
**Services**: Business logic, orchestration, complex operations  
**Repositories**: Data access, Supabase queries, multi-tenant filtering

```python
# Router
@router.get("/dre", response_model=DREResponse)
async def get_dre(year: int, user_id: str = Depends(get_current_user_id)):
    service = AggregationService(user_id)
    return await service.get_dre(year)

# Service
class AggregationService:
    def __init__(self, user_id: str):
        self.user_id = user_id
        self.repo = TransactionRepository(user_id)
    
    async def get_dre(self, year: int) -> DREResponse:
        transactions = await self.repo.get_by_date_range(
            date(year, 1, 1), date(year, 12, 31)
        )
        return self._aggregate_dre(transactions)

# Repository
class TransactionRepository:
    def __init__(self, user_id: str):
        self.user_id = user_id
        self.client = get_supabase_client()
    
    async def get_by_date_range(self, date_from: date, date_to: date):
        return self.client.table("transactions").select("*").eq(
            "user_id", self.user_id
        ).gte("date", date_from).lte("date", date_to).execute()
```

### Multi-Tenancy

User context injected via dependency. All queries filter by user_id. RLS policies enforce isolation at database level.

## Frontend Architecture

Next.js 14 with App Router. Component-based UI. Custom hooks for data fetching. Type-safe API client.

Structure:
- **app/dashboard/**: Route pages (dre, transactions, accounts)
- **components/pages/**: Page-level components
- **components/ui/**: Reusable UI primitives
- **hooks/**: Custom hooks (useAuth, useFinancialData)
- **lib/**: API client, utilities

## Data Model

Core entities: transactions, categories, accounts, recurring_rules, forecasts. Foreign keys enforce referential integrity. Indexes on user_id and date columns for performance.

```sql
CREATE TABLE transactions (
    id UUID PRIMARY KEY,
    date DATE NOT NULL,
    type VARCHAR(20) CHECK (type IN ('income', 'expense')),
    amount DECIMAL(10, 2) NOT NULL,
    category_id UUID REFERENCES categories(id),
    account_id UUID REFERENCES accounts(id),
    user_id UUID NOT NULL,
    recurring_rule_id UUID REFERENCES recurring_rules(id)
);

CREATE INDEX idx_transactions_user_date ON transactions(user_id, date);
```

## Key Services

**AggregationService**: DRE generation, KPI calculation  
**RecurringService**: Rule processing, versioning, transaction creation  
**ForecastService**: Financial forecasting (planned)  
**TransactionService**: CRUD operations, validation

