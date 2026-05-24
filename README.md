# Engineering Case Studies

Real production systems I helped design, build, rebuild, or operate.

This repository does not publish client source code. It documents the engineering work behind selected projects: architecture, trade-offs, failure modes, data boundaries, reliability patterns, and production outcomes. Client names and sensitive implementation details are intentionally generalized.

Start here if you want the shortest path through the repo: [START_HERE.md](START_HERE.md).

## What This Shows

- Backend systems with real production constraints.
- Legacy rescue, migration, and cutover risk reduction.
- Data quality work: constraints, deduplication, RLS, ETL/ELT, and observability.
- AWS infrastructure patterns: Terraform, IAM, Lambda, EventBridge, CloudWatch, SNS, and Secrets Manager.
- AI applied to operations with human review and deterministic boundaries.

## Case Studies

| Case | Main Signal | Stack |
|---|---|---|
| [Higher Education Enrollment Platform](higher-education-enrollment-platform/) | Legacy rebuild, backend boundary, RBAC, audit logs, production migration | Node.js, TypeScript, NestJS, Next.js, PostgreSQL, Prisma, AWS patterns |
| [E-commerce Analytics Data Platform](ecommerce-analytics-data-platform/) | Data warehouse rebuild, ETL/ELT, data quality, observability | Python, TypeScript, PostgreSQL, AWS Lambda, EventBridge, CloudWatch, Terraform |
| [AWS Infrastructure Platform](aws-infrastructure-platform/) | IaC standardization, least-privilege IAM, CI/CD, cloud operations | Terraform, AWS Lambda, IAM, EventBridge, CloudWatch, SNS, GitHub Actions |
| [Internal Proposal Agent](internal-proposal-agent/) | RAG, tool use, deterministic pricing, human-in-the-loop workflow | Python, FastAPI, PostgreSQL, pgvector, LLM APIs, Pydantic, Next.js |
| [Financial Management Platform](financial-management-platform/) | Financial rules, tenant isolation, recurring transactions, reporting | Python, FastAPI, PostgreSQL, Pydantic, pandas, Next.js |
| [Fintech Lead Qualification Platform](fintech-lead-qualification-platform/) | WhatsApp intake, async processing, lead routing, real-time Kanban | Node.js, TypeScript, Fastify, PostgreSQL, Redis, BullMQ, WebSockets |

## Selected Outcomes

- Reduced p95 latency in analytics workloads from ~30s to <2s.
- Rebuilt production data models with star schema, FK constraints, RLS, and validation checks.
- Replaced manual cloud setup with reusable Terraform modules and reproducible AWS environments.
- Moved sensitive business rules from browser/client execution into backend-controlled flows.
- Added operational visibility with structured logs, CloudWatch alarms, SNS routing, and explicit failure modes.
- Designed AI-assisted workflows where LLMs retrieve and structure context while deterministic code handles critical calculations.

## Engineering Patterns

- **Backend boundaries**: core rules, pricing, permissions, and writes belong on the server.
- **Data correctness**: constraints, FK validation, deduplication, RLS, and migration checks before cutover.
- **Reliability**: idempotency, retries with backoff, circuit breakers, checkpoints, and explicit failure modes.
- **Cloud operations**: infrastructure as code, least-privilege IAM, structured logs, CloudWatch alarms, and billing controls.
- **AI in production workflows**: LLMs assist with reasoning and context retrieval; deterministic code handles critical calculations and state changes.
