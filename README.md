# Engineering Case Studies

This repository documents real-world systems I designed, built, and delivered end-to-end in production. Each case study represents a complete project where I was responsible for architecture decisions, implementation, and operational concerns.

## About This Repository

These case studies are based on actual production systems I led. Due to client confidentiality and proprietary constraints, I cannot share the original source code. Instead, I've created comprehensive documentation with representative code snippets that demonstrate the architectural patterns, technical decisions, and implementation approaches used in each system.

The code examples are carefully crafted to reflect the real implementation patterns while ensuring no sensitive data, proprietary logic, or client-specific details are exposed. They serve as evidence of the engineering practices applied in production.

## What You'll Find

Each case study includes:

- **System architecture** – Component boundaries, data flow, failure handling
- **Technical decisions** – Rationale, alternatives considered, trade-offs
- **Key flows** – Core system behaviors and interactions
- **Implementation patterns** – Representative code demonstrating architectural choices
- **Operational concerns** – Reliability, observability, security boundaries
- **Results and measurement** – Outcomes observed in production, metrics, and measurement guidance

These documents are written from an engineering perspective, focusing on how systems were built, why certain choices were made, and what constraints were encountered.

## Case Studies

### [FAMATH Enrollment Ecosystem](famath-enrollment-ecosystem/)

Complete enrollment management system for higher education. Three integrated components: admin backoffice for operational workflows, student portal for enrollment and payments, and WhatsApp conversational agent for automated enrollment. Shared database with RLS policies, async processing for WhatsApp ingestion, synchronous flows for portal operations.

**Technologies**: Node.js, Fastify, Next.js, Supabase, BullMQ, Z-API, OpenAI, Asaas

### [Controle Financeiro Nexly](controle-financeiro-nexly/)

Multi-tenant financial management SaaS platform. Handles transaction management, recurring rules with versioning, DRE generation, and financial forecasting. Plan-based feature limits, CSV import processing, and multi-tenant data isolation via Row Level Security.

**Technologies**: Python, FastAPI, Next.js, Supabase, PostgreSQL, Pydantic

### [Fintech CRM Platform](fintech-crm-platform/)

CRM platform for real estate and vehicle financing with AI-powered lead qualification via WhatsApp. Dual WhatsApp sessions (consultant and AI agent), round-robin lead distribution, real-time Kanban board with WebSocket synchronization, and integrated chat interface.

**Technologies**: Node.js, Fastify, React, Prisma, PostgreSQL, Baileys, Groq, Socket.io

## Engineering Practices Demonstrated

These case studies illustrate production-grade engineering practices:

- **Retries and backoff** – Exponential backoff strategies for transient failures
- **Timeouts** – Request and operation timeouts to prevent resource exhaustion
- **Idempotency** – Safe retry mechanisms and duplicate handling
- **Async processing** – Queue-based job processing with BullMQ, Go workers, ticker-based polling
- **Observability** – Structured logging, metrics collection, distributed tracing
- **Multi-tenant isolation** – Row Level Security, tenant boundaries, data separation
- **Security boundaries** – Authentication, authorization, sensitive data handling
- **Reliability patterns** – Failure handling, graceful degradation, circuit breakers

## Technology Stack

**Languages**: Python, TypeScript, JavaScript, Go

**Backend**: FastAPI, Fastify, Express, Node.js, Gin, Prisma, SQLAlchemy

**Frontend**: React, Next.js, Vite, TypeScript, Tailwind CSS

**Datastores**: PostgreSQL, Supabase, Redis

**Infrastructure**: Docker, Docker Compose, Render, Vercel

**Real-time & Messaging**: WebSocket, SSE, Webhooks, BullMQ

**AI & Automation**: LLM orchestration, OpenAI, Groq, domain-bounded conversational agents

**Observability**: Prometheus, OpenTelemetry, Grafana, structured logging (Pino, zap, Winston)

## Reference Architecture

For detailed architecture documentation and design patterns, see: [Reference Architecture Documentation](https://sequoia-architecture-docs.vercel.app/)

## Notes

All case studies follow a consistent structure for readability. Metrics and outcomes are based on production observations where available. For systems in active development, "Planned" sections indicate future work and current limitations.
