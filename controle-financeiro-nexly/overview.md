# Controle Financeiro Nexly - Overview

Multi-tenant financial management SaaS platform for Brazilian companies. Handles transactions, recurring rules, forecasts, and financial reporting. Users create companies, import transactions via CSV, set up recurring payment rules, and generate revenue/expense forecasts. The system enforces subscription plan limits (Simple vs Pro) for features like CSV imports, categories, and advanced forecasting algorithms.

## Problem

Companies need automated financial statement generation and transaction management. Manual DRE creation is time-consuming. Recurring transactions require complex versioning when rules change.

## Solution

FastAPI backend with layered architecture (routers → services → repositories). Next.js dashboard for financial management. Supabase with RLS for multi-tenant isolation. Smart recurring rules with versioning support.

## Key Features

- DRE generation (monthly/yearly)
- Transaction management (income/expense)
- Recurring rules with smart versioning
- Financial forecasts (basic and advanced with EWMA, trend, seasonality)
- Multi-tenant SaaS architecture
- CSV import with plan-based rate limiting
- Plan limits enforcement (categories, CSV imports, features)

## Stack

**Backend:**
- FastAPI 0.104.0+ (Python 3.11)
- Supabase Python client 2.0.0+
- Pydantic 2.5.0+ for validation
- python-jose for JWT handling
- httpx for HTTP client operations
- pandas for CSV processing
- reportlab for PDF generation

**Database:**
- PostgreSQL via Supabase
- Row Level Security (RLS) for multi-tenant isolation

**Infrastructure:**
- Docker for containerization
- Docker Compose for local development
- Uvicorn ASGI server

**Evidence:**
- `backend/requirements.txt` - dependency versions
- `backend/app/core/database.py` - Supabase client initialization
- `backend/app/core/security.py` - JWT validation
- `backend/docker-compose.yml` - local development setup
- `backend/Dockerfile` - production container

## Implemented vs Planned

**Implemented**
- Multi-tenant data isolation via `company_id` and RLS policies
- JWT authentication with token validation
- CSV import with plan-based rate limiting
- Recurring transaction rules with versioning
- Forecast generation (basic and advanced with EWMA, trend, seasonality)
- Plan limits enforcement (categories, CSV imports, features)
- File storage integration (Supabase Storage)
- Health check endpoint
- Layered architecture (API → Service → Repository)
- Transaction deduplication for recurring rules

**Current State vs Design Intent**

**Current State:**
- Processing is on-demand via API endpoints
- Recurring rules processed via HTTP endpoint (`POST /recurring-rules/process-all`)
- CSV imports processed synchronously within request handler
- No background job queues or workers
- No scheduled cron jobs

**Design Intent:**
- Service layer abstraction enables queue migration without API changes
- Heavy operations (CSV import, forecast generation, recurring rule processing) are candidates for background queues
- Architecture supports adding queue infrastructure when volume requires independent scaling

**Planned**
- Async job queues or workers
- Redis or caching layer
- Rate limiting middleware (slowapi in requirements but not configured)
- Distributed tracing or structured metrics
- Dead-letter queues or retry mechanisms
- Scheduled cron jobs
- Webhook handlers for external integrations
- JWT signature verification (currently development-only)
- Token refresh mechanism
