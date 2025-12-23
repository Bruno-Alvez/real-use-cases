# Technical Decisions

## 1. Monolithic Architecture (Web + Worker in Same Process)

**Decision**: Fastify and BullMQ worker run in the same Node.js process.

**Alternatives considered**:
- Separate into two services (web service + worker service)
- Use serverless functions (AWS Lambda, Vercel)
- Use dedicated job platform (AWS SQS + Lambda)

**Why chosen**:
- Cost: monolithic architecture reduces operational cost compared to separate services
- Low volume: doesn't justify separation
- Simplicity: single deploy, less configuration
- Memory sharing: state can be shared (not currently used)

**Consequences**:
- Web and worker compete for CPU/memory (acceptable for low volume)
- Cannot scale independently (not needed yet)
- Single point of failure (acceptable for use case)
- If web crashes, worker also stops (but jobs remain in Redis queue)

**Future improvements**:
- Separate when there is recurring backlog, increasing latency, or need to scale worker independently from web
- Migrate to separated architecture if performance degrades

**Evidence**: `README.md`, `docs/PROJECT_CONTEXT.md`, `src/index.ts`

---

## 2. Conversational Mode vs State Machine

**Decision**: Conversational mode (LLM-driven) is default, state machine is fallback/legacy.

**Alternatives considered**:
- State machine only (deterministic, no LLM)
- Conversational mode only (no fallback)
- Hybrid: state machine with LLM only for intent classification

**Why chosen**:
- Natural conversation: user doesn't need to follow rigid format
- Flexible data extraction: LLM extracts CPF, email, etc. from natural language
- Better UX: less frustration when user doesn't follow exact expected format
- Fallback: if OpenAI fails, falls back to state machine

**Consequences**:
- Cost: each message generates OpenAI API call (variable cost per use)
- Latency: adds external API call latency (OpenAI) per message
- Less predictable: LLM may misinterpret (mitigated with prompts and validation)
- External dependency: system doesn't work if OpenAI is down (fallback to state machine)

**Future improvements**:
- Cache OpenAI responses for similar messages
- Fine-tune model for specific context
- More robust automatic fallback

**Evidence**: `src/agents/orchestrator.ts`, `src/agents/conversationalAgent.ts`, `src/config/env.ts`

---

## 3. Redis for State and Queue (Not PostgreSQL)

**Decision**: Conversation state and job queue use Redis, not PostgreSQL.

**Alternatives considered**:
- PostgreSQL for state (`conversation_states` table)
- In-memory (Map/Set in Node.js)
- Dedicated database for state (MongoDB, etc.)

**Why chosen**:
- Performance: Redis is faster for frequent read/write than PostgreSQL
- Native TTL: automatic expiration of abandoned conversations
- BullMQ requires Redis: cannot use PostgreSQL for queue
- Cost: managed Redis has lower cost than maintaining state in PostgreSQL

**Consequences**:
- Additional dependency: system needs Redis running
- Persistence: if Redis crashes, state is lost (acceptable, user can restart)
- Scalability: Redis may become bottleneck if volume grows significantly

**Future improvements**:
- Periodic backup of critical state to PostgreSQL
- Redis Cluster if volume grows

**Evidence**: `src/services/redis.ts`, `src/config/constants.ts`, `src/queue.ts`

---

## 4. Zod for External Data Validation

**Decision**: All external data (webhooks, env vars, API responses) is validated with Zod.

**Alternatives considered**:
- Manual validation with if/else
- Joi or Yup (other validation libraries)
- Runtime validation only (TypeScript doesn't validate runtime)

**Why chosen**:
- Type safety: Zod generates TypeScript types automatically
- Runtime validation: TypeScript doesn't validate data at runtime (webhooks, APIs)
- Clear error messages: Zod returns structured errors
- TypeScript integration: `z.infer<typeof Schema>` generates types

**Consequences**:
- Overhead: parsing/validation on each webhook (synchronous operations, low overhead)
- Dependency: one more library in project (but lightweight)
- Maintenance: schemas need to be updated when APIs change

**Future improvements**:
- Generate schemas automatically from OpenAPI/Swagger contracts
- Cache compiled schemas

**Evidence**: `src/types/schemas.ts`, `src/config/env.ts`, `src/index.ts`

---

## 5. PBKDF2 for Password Hashing (Not Simple SHA-256)

**Decision**: Passwords are hashed with PBKDF2 (SHA-256, 100k iterations, 16-byte salt), not simple SHA-256.

**Alternatives considered**:
- Simple SHA-256 (fast, but insecure)
- bcrypt (more common, but different format)
- Argon2 (more secure, but not compatible with web system)

**Why chosen**:
- Compatibility: web system uses PBKDF2, passwords must be compatible
- Security: PBKDF2 with 100k iterations is secure enough
- Format: salt + hash concatenated in Base64 (format expected by web system)

**Consequences**:
- Performance: hash is expensive operation (PBKDF2 with 100k iterations), but rare operation
- Complexity: more complex implementation than simple SHA-256
- Dependency: uses Node.js native crypto (no external lib needed)

**Future improvements**:
- Migrate web system to Argon2 (more secure)
- Or migrate both to bcrypt (more common)

**Evidence**: `src/tools/protocol.ts`

---

## 6. Concurrency 1 in BullMQ Worker

**Decision**: Worker processes 1 job at a time (concurrency 1).

**Alternatives considered**:
- Concurrency 2-5 (process multiple jobs in parallel)
- Dynamic concurrency based on load

**Why chosen**:
- Rate limits: external APIs (Z-API, OpenAI, ViaCEP) have rate limits
- Resources: process shares CPU/memory with Fastify
- Simplicity: fewer race conditions, easier logs to debug
- Low volume: doesn't justify parallelism

**Consequences**:
- Limited throughput: maximum 1 message processed at a time
- Latency: jobs in queue wait for sequential processing (time depends on each job duration)
- Acceptable: low volume, users don't notice delay

**Future improvements**:
- Increase to 2-3 when volume grows
- Implement rate limiting per API to avoid blocks

**Evidence**: `src/config/constants.ts`, `src/queue.ts`

---

## 7. Retry 2 Attempts with 5s Backoff

**Decision**: Failed jobs are retried 2 times with exponential backoff of 5 seconds.

**Alternatives considered**:
- 3 attempts with 1s backoff
- 5 attempts with 10s backoff
- No retry (immediate failure)

**Why chosen**:
- Balance: 2 attempts covers temporary failures without overloading
- 5s backoff: avoids rate limits (external APIs may block if too many requests)
- Cost: fewer OpenAI/Supabase calls in case of persistent failure

**Consequences**:
- Failed jobs take time to fail definitively (2 attempts with exponential backoff of 5s)
- If failure is persistent (bug in code), job is reprocessed 2x before failing
- No dead-letter queue: failed jobs are lost (logged, but not reprocessed later)

**Future improvements**:
- Dead-letter queue for jobs that fail after all attempts
- Alerts when failure rate > X%
- Retry with adaptive backoff based on error type

**Evidence**: `src/config/constants.ts`, `src/queue.ts`

---

## 8. 24h TTL in Redis (Not Eternal Persistence)

**Decision**: State and temporary data expire after 24 hours in Redis.

**Alternatives considered**:
- Infinite TTL (data never expires)
- 1 hour TTL (more aggressive)
- 7 day TTL (more conservative)

**Why chosen**:
- Automatic cleanup: abandoned conversations are automatically removed
- Privacy: sensitive data doesn't stay in Redis indefinitely
- Cost: Redis doesn't accumulate old data
- UX: user can restart conversation after 24h without losing much context

**Consequences**:
- If user returns after 24h, needs to restart flow (acceptable)
- Temporary data is lost if user takes too long (mitigated: flow normally complete in few minutes)
- Handoff has 48h TTL (more time for attendant to respond)

**Future improvements**:
- Adaptive TTL: extend TTL if user is active
- Persist critical state to PostgreSQL before expiring

**Evidence**: `src/config/constants.ts`

---

## 9. Two Separate Redis Clients (General vs BullMQ)

**Decision**: Two separate Redis clients: one for general operations, another for BullMQ.

**Alternatives considered**:
- Single Redis client for everything
- Connection pool

**Why chosen**:
- BullMQ requires `maxRetriesPerRequest: null` (specific behavior)
- General operations need normal retry strategy
- Isolation: problems in one client don't affect the other
- Simplicity: clear and separated configuration

**Consequences**:
- More Redis connections (2 instead of 1, but acceptable)
- More configuration code (but clearer)

**Future improvements**:
- None needed, solid decision

**Evidence**: `src/services/redis.ts`

---

## 10. Timeout 20s on Z-API, 10s on ViaCEP, 30s on OpenAI

**Decision**: Different timeouts per service: Z-API 20s, ViaCEP 10s, OpenAI 30s.

**Alternatives considered**:
- Single timeout for all (e.g., 30s)
- More aggressive timeouts (5s for all)
- More conservative timeouts (60s for all)

**Why chosen**:
- Z-API 20s: message sending may take time (network, processing)
- ViaCEP 10s: public API, may be slow, but response is simple
- OpenAI 30s: LLM may take time to generate response, especially with large context
- Balance: not too aggressive (avoids false negatives), not too conservative (avoids long wait)

**Consequences**:
- If API is slow but responds in 25s, Z-API fails but OpenAI doesn't (different behavior)
- User may wait up to 30s for response (acceptable for WhatsApp)

**Future improvements**:
- Adaptive timeout based on historical latency
- Circuit breaker for APIs that fail frequently

**Evidence**: `src/tools/zapi.ts`, `src/tools/viacep.ts`, `src/services/openai.ts`, `src/config/constants.ts`
