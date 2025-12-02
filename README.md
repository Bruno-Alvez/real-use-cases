# Bruno Alves – Technical Case Studies

This repository contains a curated set of technical case studies from systems I designed and delivered in production. Each case documents engineering reasoning, architectural decisions, operational constraints, and representative implementation patterns.

These are selected projects, not an exhaustive list. The focus is on systems that illustrate my work across backend architecture, distributed flows, automation, real-time pipelines, and LLM-driven components.

---

## Purpose

The goal of this repository is to provide clear, engineering-oriented documentation of:

- how the systems were structured  
- why specific architectural decisions were made  
- which constraints shaped the design  
- how the solutions were implemented  
- what trade-offs were accepted  

The emphasis is on architecture and applied engineering, not code dumps or theoretical patterns.

---

## Structure

Each case is isolated in its own directory and includes:

### **overview.md**  
High-level description of the system and the operational problem.

### **architecture.md**  
Architecture, components, boundaries, and design rationale.

### **key-flows.md**  
Core flows that define system behavior.

### **technical-decisions.md**  
Major technical choices and alternatives considered.

### **challenges.md**  
Non-trivial problems and how they were resolved.

### **representative-snippets.md**  
Selected code excerpts demonstrating architectural patterns.

### **results.md**  
Delivered outcomes, stability, and business impact.

All cases follow the same structure for consistency and clarity.

---

## Included Case Studies

### Enrollment and Academic Systems
- **famath-enrollment-portal** – Student enrollment portal with pricing engine and document intake  
- **famath-enrollment-admin** – Administrative enrollment and validation flow  
- **famath-whatsapp-agent** – Deterministic conversational agent for enrollment automation

### CRM and Lead Intelligence
- **inmov-crm-platform** – CRM platform with WhatsApp ingestion and AI SDR agent

### Financial SaaS
- **dre-saas-platform** – Multi-tenant DRE and financial control system (FastAPI + Supabase)

### Analytics and Data
- **silva-analytics-platform** – Customer intelligence and analytics platform with RFM modeling

### Observability and Integrations
- **opsforge-platform** – Webhook delivery, retries, and monitoring (Go + Next.js)

---

## Technology Scope

The systems documented here involve:

- **Languages:** Python, TypeScript, JavaScript, Go  
- **Backend:** FastAPI, Fastify, Node.js, Prisma, SQLAlchemy  
- **Frontend:** React, Next.js  
- **Datastores:** PostgreSQL, Supabase, Redis  
- **Infrastructure:** Docker, Render, Vercel  
- **Real-Time & Messaging:** SSE, Webhooks, BullMQ  
- **AI & Automation:** LLM orchestration, domain-bounded agents, OpenAI, Groq  

Each case focuses on applied engineering and architectural clarity.

---

## Notes

No proprietary code or client-specific data is included.  
All examples and explanations are limited to architectural reasoning and high-level implementation concepts.