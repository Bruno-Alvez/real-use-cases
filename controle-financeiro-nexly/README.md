# Controle Financeiro Nexly

Multi-tenant financial management SaaS platform for Brazilian companies. Handles transaction management, recurring payment rules with versioning, DRE (Demonstração do Resultado do Exercício) generation, and financial forecasting with advanced algorithms.

## What This System Does

Companies use this platform to manage their financial operations: import transactions via CSV, set up recurring payment rules, generate automated financial statements, and create revenue/expense forecasts. The system enforces subscription plan limits (Simple vs Pro) for features like CSV imports, categories, and advanced forecasting capabilities.

## How to Read This Case Study

Start with **overview.md** to understand the problem, solution approach, and key features. Then read **architecture.md** to see how components are organized and how data flows through the system. **key-flows.md** describes the core system behaviors in detail.

**technical-decisions.md** explains the major architectural choices, alternatives considered, and trade-offs. **challenges.md** covers non-trivial problems encountered during implementation. **representative-snippets.md** shows code examples that demonstrate the patterns used.

Finally, **results.md** presents outcomes observed in production, metrics, and guidance on what to measure next.

## Documentation Structure

- **overview.md** – System description, problem statement, solution approach, technology stack, and implemented vs planned features
- **architecture.md** – Component boundaries, data flow, multi-tenant isolation, failure handling, and observability
- **key-flows.md** – Core flows: transaction import, recurring rule processing, DRE generation, forecast calculation
- **technical-decisions.md** – Major decisions: layered architecture, recurring rule versioning, plan limits enforcement, forecasting algorithms
- **challenges.md** – Complex problems: recurring rule versioning, transaction deduplication, plan limit enforcement, CSV import processing
- **representative-snippets.md** – Code examples demonstrating architectural patterns, validation, and business logic
- **results.md** – Production outcomes, metrics observed, and measurement guidance

## Key Technical Highlights

- **Multi-tenant isolation** via Row Level Security (RLS) policies in Supabase
- **Recurring rule versioning** to handle rule changes without breaking historical data
- **Plan-based feature limits** enforced at the service layer
- **Forecasting algorithms** with basic and advanced modes (EWMA, trend, seasonality)
- **Layered architecture** (routers → services → repositories) enabling future queue migration
- **Transaction deduplication** to prevent duplicate entries from recurring rules

## Technology Stack

**Backend**: Python 3.11, FastAPI, Supabase, Pydantic, pandas, reportlab  
**Frontend**: Next.js, TypeScript  
**Database**: PostgreSQL via Supabase with RLS  
**Infrastructure**: Docker, Docker Compose, Uvicorn

