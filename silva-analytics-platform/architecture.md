# Architecture

## System Overview

Monorepo with FastAPI backend and Next.js frontend. Supabase as data warehouse with analytical views for RFM analysis. Clean architecture with separation of concerns: API routes → services → data access.

## Backend Architecture

### FastAPI Application

FastAPI with Pydantic for validation, async/await for performance. Structured configuration with environment variables. CORS middleware for frontend integration.

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from app.core.config import settings

app = FastAPI(
    title=settings.PROJECT_NAME,
    version=settings.VERSION,
    description="Analytics and customer intelligence platform",
    docs_url="/docs",
    redoc_url="/redoc",
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.BACKEND_CORS_ORIGINS,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

### Service Layer

MVC + Service hybrid pattern. Services handle business logic and data aggregation. Supabase client for data access. Future: Celery for async tasks.

Structure:
- **api/v1/**: Route handlers (analytics, customers, segmentation)
- **services/**: Business logic (analytics service, segmentation service)
- **schemas/**: Pydantic models for request/response validation
- **models/**: SQLAlchemy models (if needed)
- **core/**: Configuration, security, database setup

### Data Layer

Supabase PostgreSQL with analytical views:
- RFM analysis views (Recency, Frequency, Monetary)
- Customer segmentation tables
- Campaign performance metrics
- Historical snapshots

Supabase client handles connection pooling and query execution. Async queries for performance.

```python
from supabase import create_client

supabase = create_client(
    settings.SUPABASE_URL,
    settings.SUPABASE_KEY
)

# Query RFM data
response = supabase.table("rfm_analysis").select("*").execute()
```

### Security

JWT authentication with python-jose. Password hashing with bcrypt. Token expiration configurable. Role-based access control (planned).

```python
def create_access_token(subject: str, expires_delta: timedelta | None = None) -> str:
    if expires_delta:
        expire = datetime.now(timezone.utc) + expires_delta
    else:
        expire = datetime.now(timezone.utc) + timedelta(
            minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES
        )
    
    to_encode = {"exp": expire, "sub": str(subject)}
    return jwt.encode(to_encode, settings.SECRET_KEY, algorithm=settings.ALGORITHM)
```

## Frontend Architecture

Next.js 15 with App Router. Server and client components. React Query for server state management. Recharts for data visualization.

Component structure:
- **app/**: Route-level pages (analytics, customers, segmentation)
- **components/ui/**: Reusable UI primitives (Card, Button, Charts)
- **components/features/**: Feature-specific components (CreateSegmentModal)
- **components/layout/**: Layout components (Header, Footer)
- **lib/**: API client, utilities
- **data/**: Mock data for development

State management: React Query for server state caching. Local state with useState for UI. No global state management needed.

```typescript
const { data, isLoading } = useQuery({
    queryKey: ['analytics', filters],
    queryFn: () => api.get<AnalyticsData>('/api/v1/analytics', { params: filters }),
    staleTime: 30000,
});
```

## Data Flow

### Analytics Dashboard

1. Frontend requests analytics data with date filters
2. FastAPI service queries Supabase for aggregated metrics
3. Data aggregated from multiple sources (WordPress, Bling, CRM)
4. Response includes KPIs, trends, cluster performance
5. React Query caches response
6. Recharts renders visualizations

### Customer Segmentation

1. User selects month filter
2. Frontend requests customers for selected period
3. Backend queries Supabase RFM analysis view
4. Data includes recency, frequency, monetary, cluster assignment
5. Frontend calculates metrics (total customers, avg value, MoM)
6. Charts display cluster distribution

### Campaign Monitoring

1. Campaign data ingested from external sources
2. Stored in Supabase with timestamps
3. Analytics service aggregates performance metrics
4. Dashboard displays campaign effectiveness
5. AI model performance tracked separately

## Integration Points

- **WordPress**: Webhook or API integration for e-commerce data
- **Bling ERP**: API integration for order and inventory data
- **CRM**: API integration for customer interactions
- **AI Models**: Event tracking for model activations and results
- **Supabase**: Central data warehouse with analytical views

## Deployment

Docker Compose for local development. Separate containers for backend, frontend, Supabase. Production: Backend on cloud (Render/Railway), Frontend on Vercel, Supabase cloud instance.

