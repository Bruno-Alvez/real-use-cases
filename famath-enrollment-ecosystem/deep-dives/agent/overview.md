# Overview

## What the System Does

WhatsApp enrollment automation system for educational institutions. Replicates the web enrollment flow through natural conversation, allowing candidates to complete enrollment without leaving WhatsApp. The system classifies intents, collects personal data, validates information, queries addresses, presents courses and modalities, and creates enrollments in the shared database with the web system.

## Integrations

- **Z-API**: Receives WhatsApp message webhooks and sends messages (legacy support for Evolution API in code, not used in production)
- **Supabase (PostgreSQL)**: Shared database with web systems (tables: students, enrollments, courses, entry_modalities)
- **ViaCEP**: Address lookup by postal code (fallback: user provides address manually if it fails)
- **OpenAI (GPT-3.5-turbo)**: Intent classification and full conversational mode

## Stack

- **Runtime**: Node.js 20 LTS
- **Language**: TypeScript (strict mode)
- **Web Server**: Fastify
- **Job Queue**: BullMQ
- **Cache/State**: Redis 7+
- **Database**: Supabase (PostgreSQL)
- **Validation**: Zod
- **Logging**: Pino (structured logging)
- **Containerization**: Docker (multi-stage build)

**Evidence**: `package.json`, `Dockerfile`, `src/index.ts`, `src/queue.ts`

## Implemented

- Webhook endpoint to receive WhatsApp messages
- Async queue with BullMQ for message processing
- State machine for enrollment flow (legacy mode)
- Full conversational mode with LLM (default mode)
- CPF validation (format + checksum)
- Email and postal code validation
- ViaCEP integration with retry and fallback
- Student and enrollment creation in Supabase
- Protocol generation (format FAM2025XXXX)
- Password generation (format: 6CPFdigits@initials)
- Password hashing with PBKDF2 (compatible with web system)
- Handoff to human attendant
- Enrollment status query
- Sensitive data sanitization in logs
- Timeouts on all external HTTP calls
- Retry with backoff (BullMQ: exponential 5s, ViaCEP: manual 1s/2s)
- Automatic TTL in Redis (24h for state, 48h for handoff)

**Evidence**: `src/agents/orchestrator.ts`, `src/agents/conversationalAgent.ts`, `src/tools/validation.ts`, `src/tools/viacep.ts`, `src/tools/protocol.ts`

## Scope and Out of Scope

**Included**:
- Complete enrollment flow via WhatsApp
- Integration with shared database (Supabase)
- Data validation and address lookup
- Handoff to human attendant

**Not included** (planned for future versions):
- Document upload via WhatsApp
- Payment links
- Proactive notifications
- Multi-institution support
- Architecture separation (web + worker in separate processes)
- Instrumented metrics and dashboards
- Dead-letter queue for failed jobs
- Rate limiting on webhook
- Explicit idempotency with dedupe keys
