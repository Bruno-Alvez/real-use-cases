# Engineering Case Studies

This repository documents real-world systems I designed, built, and delivered end-to-end in production. Each case study represents a complete project where I was responsible for architecture decisions, implementation, and operational concerns. Due to client confidentiality, original source code is not shared, but comprehensive documentation with representative snippets covers the architectural patterns, technical decisions, and production outcomes.

## Case Studies

### [Silva Nutrition Ecosystem](silva-nutrition-ecosystem/)

Full reconstruction of a production data warehouse for a Brazilian supplement e-commerce. The legacy system had accumulated years of unmanaged debt: 62,466 fraudulent customer records contaminating the entire dataset, no primary keys on 8 of 22 tables, 17 tables with RLS disabled, 15 unauthenticated API endpoints exposing CPF data, and pipelines down for weeks at a time with no observability. The intervention replaced everything: a Strangler Fig migration to a clean Star Schema, 9 AWS Lambda ETL functions replacing N8N pipelines, full RLS coverage, and 28 CloudWatch alarms with SNS routing to email and Slack.

**Key outcomes**: data quality score 2/10 to 9/10, p95 API latency from >30 seconds to <2 seconds, 89,487 problematic records eliminated, MTTD from >24 hours to <5 minutes

**Technologies**: Python, AWS Lambda, Terraform, Supabase, PostgreSQL, WooCommerce API, Bling API v3, FastAPI, CloudWatch, SNS

---

### [Nexly Cloud Infrastructure](nexly-cloud-infrastructure/)

Infrastructure platform for Nexly Group's client delivery operations. Before this project, every client environment was provisioned manually through the AWS console, with no version control, no audit trail, and no reproducibility. A new environment required 4 to 8 hours of console navigation and left behind undocumented configuration state that drifted over time. The intervention replaced ClickOps with modular Terraform, migrated N8N data pipelines to AWS Lambda, and established a GitHub Actions CI/CD pipeline with an approval-gated production apply.

**Key outcomes**: operational cost from R$8,230/month to R$415/month (-95%), new client environment time from 4-8 hours to 15 minutes, disaster recovery from impossible to 15-minute RTO via terraform apply, MTTD from >24 hours to <5 minutes

**Technologies**: Terraform, AWS Lambda, EventBridge, CloudWatch, SNS, Secrets Manager, GitHub Actions, Python, DynamoDB

---

### [Fintech CRM Platform](fintech-crm-platform/)

CRM platform for a fintech operating in real estate and vehicle financing. The manual lead pipeline was producing cold data and slow response times: web forms, manual distribution via spreadsheet, and consultants spending mornings on unqualified prospects. The replacement is an AI agent that qualifies leads via WhatsApp 24/7 using Groq LLM, a round-robin scheduler that assigns qualified leads in under two seconds, and a real-time Kanban board with integrated WhatsApp chat so consultants never leave the interface.

**Key outcomes**: initial contact time from 2 hours to 18 minutes, leads per consultant up 40%, conversion up 12%, AI qualification accuracy 75-82% vs human expert

**Technologies**: Node.js 20, Fastify, Prisma, PostgreSQL, Baileys, Groq, Socket.io, BullMQ, Redis, React 19, TypeScript, Render, Vercel

---

### [Controle Financeiro Nexly](controle-financeiro-nexly/)

Multi-tenant financial management SaaS for Brazilian companies. Manual DRE consolidation and recurring transaction management in spreadsheets was the target state it replaced. The platform delivers automated DRE generation, recurring rules with a versioning model that preserves historical accuracy, EWMA-based financial forecasting with trend and seasonality, CSV import with per-row partial success tracking, and full multi-tenant isolation via RLS enforced at the PostgreSQL layer independently of application code.

**Key outcomes**: DRE generation automated to under 30 seconds, 0 duplicate transactions from recurring rule processing, 100% RLS coverage across all tenant tables, plan-based feature gating enforced before any data operation

**Technologies**: Python 3.11, FastAPI, Supabase, PostgreSQL, Pydantic, pandas, reportlab, Next.js, TypeScript, Docker

---

### [FAMATH Enrollment Ecosystem](famath-enrollment-ecosystem/)

Complete enrollment management system for a higher education institution. Three integrated components: admin backoffice for operational workflows, student portal for enrollment and payments, and a WhatsApp conversational agent for automated enrollment handling. Shared database with RLS policies, async processing via BullMQ for WhatsApp ingestion, synchronous flows for portal operations, and Asaas payment gateway integration.

**Technologies**: Node.js, Fastify, Next.js, Supabase, BullMQ, Z-API, OpenAI, Asaas

---

## Engineering Practices

These case studies document production-grade practices applied across real systems.

- **Idempotency**: UPSERT-based writes, cursor-based incremental sync, transaction deduplication keys
- **Resilience**: Exponential backoff with retry budgets, circuit breakers, distributed locks for shared mutable state
- **Observability**: Structured JSON logging, CloudWatch metric filters, alarm-per-failure-mode design, MTTD under 5 minutes
- **Multi-tenant isolation**: Row Level Security enforced at the database layer, independently of application code
- **Infrastructure as code**: Terraform modules, S3-backed state, DynamoDB locking, CI/CD with approval-gated production applies
- **Data quality**: FK constraints, RLS policies, deduplication before insert, validation against authoritative sources before migration
