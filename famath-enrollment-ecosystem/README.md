# FAMATH Enrollment Ecosystem

End-to-end enrollment management system for higher education. Handles candidate registration, document validation, dynamic pricing, payment processing, and operational workflows across three integrated components sharing a unified database.

## What This System Does

The ecosystem manages the complete student enrollment lifecycle from initial candidate contact through payment confirmation. Three components coordinate enrollment workflows:

**Admin Backoffice**: Internal tool for secretaries and marketing staff. Manages candidate records, document validation, voucher and campaign administration, enrollment status tracking via Kanban interface, and granular permission controls by department.

**Student Portal**: Public-facing enrollment system. Candidates register, select courses, upload required documents, calculate dynamic pricing, apply vouchers, complete payment via Asaas (PIX or credit card), and view enrollment timeline. Enrollment data is designed to sync to a legacy SQL Server system (planned; not implemented in codebase yet).

**WhatsApp Agent**: Conversational enrollment automation. Receives messages via Z-API webhooks, processes asynchronously via BullMQ, uses LLM for natural conversation, validates data, creates enrollments in shared database, and sends credentials via WhatsApp.

## How to Read This Case Study

Start with **overview.md** to understand the scope, roles, integrations, and what's implemented vs planned. **architecture.md** explains the component boundaries, data flow, async vs synchronous processing, and data stores.

**key-flows.md** describes four critical flows: WhatsApp agent enrollment, portal enrollment with documents and payment, admin operational workflows, and legacy SQL Server sync. **technical-decisions.md** covers eight major decisions: component separation, state model, payment integration, legacy sync strategy, permission model, WhatsApp reliability, data validation, and observability.

**reliability-and-ops.md** documents implemented reliability mechanisms, known gaps, and operational troubleshooting guidance. **security-and-data-boundaries.md** covers authentication, authorization, permission boundaries, and sensitive data handling.

**results-and-measurement.md** presents outcomes, current measurements, and guidance on what to measure next. **verification-checklist.md** provides a checklist for validating the ecosystem with evidence file paths.

For detailed component-specific documentation, see the **deep-dives/** folder: `admin/`, `portal/`, and `agent/` each contain their own architecture, flows, decisions, and code snippets.

## Documentation Structure

### Main Documentation

- **README.md** (this file) – Entry point and navigation guide
- **overview.md** – Scope, roles, integrations, implemented vs planned features
- **architecture.md** – Components, data flow, async vs sync, data stores, isolation model
- **key-flows.md** – Four high-impact flows across the ecosystem
- **technical-decisions.md** – Eight major technical decisions with rationale and trade-offs
- **reliability-and-ops.md** – Reliability mechanisms, known gaps, operational guide
- **security-and-data-boundaries.md** – Authn/Authz, permission boundaries, sensitive data handling
- **results-and-measurement.md** – Outcomes, current measurements, future measurement guidance
- **verification-checklist.md** – Checklist for ecosystem validation with evidence paths

### Deep Dives

Each component (`admin/`, `portal/`, `agent/`) contains:

- **overview.md** – Component-specific description and features
- **architecture.md** – Component architecture and boundaries
- **key-flows.md** – Component-specific flows
- **technical-decisions.md** – Component-specific decisions
- **challenges.md** – Component-specific challenges
- **representative-snippets.md** – Code examples for the component
- **results.md** – Component-specific outcomes

## Key Technical Highlights

- **Shared database** with Row Level Security (RLS) policies for multi-component access control
- **Async processing** for WhatsApp ingestion via BullMQ with retry and backoff
- **Synchronous flows** for portal and admin operations (timeouts are explicit mainly in the Agent; Portal/Admin timeouts are a known hardening gap)
- **Dynamic pricing engine** with timezone awareness and voucher/campaign integration
- **Conversational LLM agent** for natural language enrollment via WhatsApp
- **Legacy system integration** with SQL Server sync (planned implementation)
- **Permission model** with granular department-based access control
- **Database triggers** for automatic status updates and protocol generation

## Technology Stack

**Backend**: Node.js, Fastify, Express.js, TypeScript  
**Frontend**: React, Next.js  
**Database**: PostgreSQL via Supabase (RLS)  
**Queue**: BullMQ (Redis)  
**Integrations**: Z-API, Asaas, OpenAI, ViaCEP  
**Infrastructure**: Docker, Render, Vercel
