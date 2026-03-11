# Controle Financeiro Nexly

Multi-tenant financial management SaaS built for Brazilian companies at Nexly Group. The platform handles transaction management, recurring payment rules with versioning, automated DRE generation, financial forecasting, and plan-based feature gating, all with full tenant isolation enforced at the PostgreSQL layer.

**Period:** 2025-2026
**Status:** In production
**Role:** Sole architect and implementer

---

| Dimension | Outcome |
|---|---|
| DRE generation | Automated monthly and annual statements in under 30 seconds |
| Recurring rule deduplication | 0 duplicate transactions in production |
| Multi-tenant isolation | 100% RLS coverage across all tenant tables |
| Plan limit enforcement | Enforced at service layer before any data operation |
| Recurring rule versioning | Full history preserved via original_rule_id linkage |
| CSV import model | Per-row partial success: valid rows always process |

---

## How to Read This Case Study

- [Overview](overview.md) — business context, problem, solution, and key features
- [Architecture](architecture.md) — component boundaries, data flow, and multi-tenant isolation model
- [Technical Decisions](technical-decisions.md) — major architectural choices, alternatives considered, and trade-offs accepted
- [Key Flows](key-flows.md) — transaction import, recurring rule processing, DRE generation, forecast calculation
- [Representative Snippets](representative-snippets.md) — code examples showing deduplication, versioning, and plan enforcement patterns
- [Results](results.md) — quantified outcomes across DRE automation, reliability, and security posture

## Key Technical Highlights

- **Multi-tenant isolation via RLS:** every tenant table has a Row Level Security policy enforced at the PostgreSQL layer, independently of application code
- **Recurring rule versioning:** "update future only" operations create a new rule version, preserving historical accuracy through the original_rule_id and effective_from_date chain
- **Transaction deduplication:** a composite key across date, amount, description, category, and account ensures recurring rules are safe to process multiple times
- **Plan limits at the service layer:** feature gating happens before any data operation, not after, with checks consolidated in a single PlanLimitsService called explicitly from each feature service
- **Queue-ready service layer:** the router-to-service-to-repository boundary was designed from day one so that synchronous processing can migrate to background queues without changing API contracts
- **Composite indexes on (company_id, date):** all tenant queries use this compound index, keeping per-tenant filtered queries fast as each tenant's data volume grows independently

## Technology Stack

| Layer | Technology |
|---|---|
| Backend | Python 3.11, FastAPI, Pydantic |
| Database | PostgreSQL via Supabase, RLS enforced |
| Data processing | pandas (CSV import), reportlab (PDF export) |
| Auth | JWT validation, company_id resolved via database |
| Frontend | Next.js, TypeScript |
| Infrastructure | Docker, Docker Compose, Uvicorn |
