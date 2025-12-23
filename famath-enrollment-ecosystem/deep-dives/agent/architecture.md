# Architecture

## Components

### Web API (Fastify)

HTTP server that receives WhatsApp webhooks. Validates payload with Zod, ignores messages from the bot itself and empty messages, and enqueues jobs in BullMQ. Responds immediately after enqueuing (synchronous operations: Zod validation and enqueuing).

**Evidence**: `src/index.ts`

### Worker (BullMQ)

Processes messages from the queue asynchronously. Concurrency set to 1 (avoids overload). Automatic retry: 2 attempts with exponential backoff of 5 seconds. Worker runs in the same Node.js process as the Fastify server (monolithic architecture).

**Evidence**: `src/queue.ts`, `src/config/constants.ts`

### Redis

Used for four purposes:
1. **Job queue** (BullMQ): stores pending and in-progress jobs
2. **State machine**: stores conversation state by phone number (key `state:{phone}`)
3. **Temporary data**: stores data collected during the flow (key `temp_data:{phone}`)
4. **Conversation history**: stores last 10 messages for conversational mode (key `history:{phone}`, LIST type)

Two separate Redis clients: one for general operations (with normal retry) and another for BullMQ (with `maxRetriesPerRequest: null`).

**Evidence**: `src/services/redis.ts`, `src/config/constants.ts`

### Database (Supabase/PostgreSQL)

Shared database with web systems. Main tables:
- `students`: personal data (CPF, name, email, address, phone)
- `enrollments`: enrollments (protocol, student, course, modality, shift, status)
- `courses`: available courses (name, code, shifts, status)
- `entry_modalities`: entry modalities (Vestibular, ENEM, Transfer, etc.)

Read operations use Supabase anon key. Write operations (create student, create enrollment) use service role key.

**Evidence**: `src/tools/supabase.ts`, `src/services/supabase.ts`

### Orchestrator

Routes messages to handlers based on current state. Supports two modes:
- **Conversational mode** (default): uses LLM for entire conversation, extracts data from natural language
- **State machine mode** (legacy): deterministic flow with explicit states

**Evidence**: `src/agents/orchestrator.ts`

## Data Boundaries

No multi-tenancy. Each conversation is isolated by phone number. State and temporary data are stored with key `{prefix}:{phone}`. 24-hour TTL automatically removes abandoned conversations.

**Evidence**: `src/config/constants.ts`, `src/services/redis.ts`

## Async Processing

Messages are enqueued immediately in the webhook (fast response). Worker processes jobs sequentially (concurrency 1) to avoid rate limits and overload. Failed jobs are automatically retried by BullMQ (2 attempts, exponential backoff of 5s).

**Evidence**: `src/index.ts`, `src/queue.ts`

## Storage

### Redis (temporary)

- `state:{phone}`: current conversation state (STRING, TTL 24h)
- `temp_data:{phone}`: data collected during flow (STRING JSON, TTL 24h)
- `history:{phone}`: message history (LIST, last 10, TTL 24h)
- `handoff:{phone}`: handoff requested flag (STRING, TTL 48h)

### PostgreSQL (persistent)

- `students`: candidate personal data
- `enrollments`: created enrollments
- `courses`: course catalog
- `entry_modalities`: entry modalities

**Evidence**: `src/config/constants.ts`, `src/tools/supabase.ts`

## Evidence by Component

- **Web API**: `src/index.ts`
- **Worker**: `src/queue.ts`
- **Redis**: `src/services/redis.ts`
- **Orchestrator**: `src/agents/orchestrator.ts`
- **Conversational Mode**: `src/agents/conversationalAgent.ts`
- **WhatsApp Integration**: `src/tools/zapi.ts`
- **Supabase Integration**: `src/tools/supabase.ts`, `src/services/supabase.ts`
- **ViaCEP Integration**: `src/tools/viacep.ts`
- **OpenAI Integration**: `src/services/openai.ts`, `src/agents/intentClassifier.ts`
- **Validations**: `src/tools/validation.ts`
- **Protocol/password generation**: `src/tools/protocol.ts`
