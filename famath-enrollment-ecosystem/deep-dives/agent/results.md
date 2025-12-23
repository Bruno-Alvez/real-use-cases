# Results and Measurement

## Outcomes (Qualitative)

### Delivered Functionality

- System receives WhatsApp messages via webhook and processes asynchronously
- Complete enrollment flow via natural conversation (conversational mode)
- CPF, email, postal code validation with friendly user feedback
- ViaCEP integration for automatic address filling
- Student and enrollment creation in shared database with web system
- Protocol and password generation compatible with web system
- Handoff to human attendant when requested
- Existing enrollment status query

**Evidence**: Code implemented in `src/agents/`, `src/tools/`, `src/services/`

### Architecture

- Monolithic architecture: web server and worker in same Node.js process
- Async processing: webhook enqueues job and responds immediately
- Automatic retry with backoff: BullMQ retries failed jobs (2 attempts, 5s backoff)
- Automatic TTL in Redis: state expires after 24h, cleans abandoned conversations
- Sensitive data sanitization in logs: CPF, phone and email are masked

**Evidence**: `src/index.ts`, `src/queue.ts`, `src/services/redis.ts`, `src/utils/sanitizers.ts`

---

## What is Measured Today

### Structured Logs

System generates structured logs with Pino containing:

- **Webhook events**: reception, validation, enqueuing
- **Job events**: start, completion, failure (with attempts)
- **State transitions**: conversation state changes
- **API calls**: message sending, ViaCEP lookup, OpenAI call
- **Errors**: complete stack traces with context

**Evidence**: Use of `logger.info/error/warn` throughout code, `src/utils/logger.ts`

### Metrics Derivable via Logs

System has no instrumented metrics, but structured logs allow extracting metrics via parsing:

- **Throughput**: count logs with `msg: "Message job completed"` per time interval
- **Failure rate**: count logs with `msg: "Job failed"` vs `msg: "Message job completed"`
- **Latency**: calculate timestamp difference between logs `msg: "Processing message job"` and `msg: "Message job completed"`
- **Errors per API**: count logs with `error` per service (Z-API, ViaCEP, OpenAI, Supabase)
- **Conversion rate**: count logs `msg: "Enrollment created"` vs `msg: "Webhook received - queuing message"`

**Existing log events**: `Webhook received - queuing message`, `Processing message job`, `Message job completed`, `Job failed`, `Enrollment created`, `ViaCEP lookup successful`, specific errors per API.

**Evidence**: Structured logs in JSON (`src/utils/logger.ts`), events logged in `src/index.ts`, `src/queue.ts`, `src/tools/`

### Health Check

Endpoint `/health` returns basic system status.

**Evidence**: `src/index.ts`

---

## Suggested Metrics for Future Implementation

### 1. Message Throughput

**Metric**: Messages processed per minute/hour

**How to measure**: Count logs "Message job completed" per time interval

**Where to instrument**: `src/queue.ts` - add counter when processing job

**Relevance**: Indicates system capacity, detects degradation

---

### 2. Processing Latency

**Metric**: Time between webhook reception and user response sending (p50, p95, p99)

**How to measure**: Timestamp at job start vs timestamp at message sending

**Where to instrument**: `src/queue.ts` - capture timestamp at job start, `src/tools/zapi.ts` - capture timestamp at message sending

**Relevance**: UX, detects slowness in external APIs

---

### 3. Job Failure Rate

**Metric**: Percentage of jobs that fail after all attempts

**How to measure**: Count logs "Job failed" vs "Message job completed"

**Where to instrument**: `src/queue.ts` - increment counter on `failed` event

**Relevance**: System reliability, detects bugs or integration problems

---

### 4. Failure Rate per External API

**Metric**: Error rate per service (Z-API, ViaCEP, OpenAI, Supabase)

**How to measure**: Count specific errors per service in logs

**Where to instrument**: `src/tools/zapi.ts`, `src/tools/viacep.ts`, `src/services/openai.ts`, `src/tools/supabase.ts` - increment counters in catch blocks

**Relevance**: Identifies which integration is problematic

---

### 5. Conversion Rate (Enrollments Created)

**Metric**: Percentage of conversations that result in enrollment creation

**How to measure**: Count logs "Enrollment created" vs "Webhook received"

**Where to instrument**: `src/tools/supabase.ts` - increment counter after creating enrollment

**Relevance**: Flow effectiveness, detects abandonment points

---

### 6. Average Enrollment Flow Time

**Metric**: Time from first message to enrollment creation

**How to measure**: Timestamp of first message (INITIAL state) vs timestamp of enrollment creation

**Where to instrument**: `src/agents/orchestrator.ts` - save timestamp on first message, `src/tools/supabase.ts` - calculate difference when creating enrollment

**Relevance**: UX, identifies if flow is too long

---

### 7. ViaCEP Usage Rate vs Manual Address

**Metric**: Percentage of addresses filled via ViaCEP vs provided manually

**How to measure**: Count logs "ViaCEP lookup successful" vs manual address collection logs

**Where to instrument**: `src/tools/viacep.ts` - success counter, `src/agents/enrollmentFlow.ts` - manual collection counter

**Relevance**: ViaCEP integration effectiveness, detects API problems

---

### 8. Handoff Request Rate

**Metric**: Percentage of conversations that result in handoff to human

**How to measure**: Count logs "Handoff requested" vs "Webhook received"

**Where to instrument**: `src/agents/handoffHandler.ts` - increment counter when processing handoff

**Relevance**: Indicates when bot cannot help, may indicate need for improvements

---

### 9. Validation Error Rate (CPF, Email, Postal Code)

**Metric**: Number of failed validation attempts per type

**How to measure**: Count failed validation logs (already exists in code, but not aggregated)

**Where to instrument**: `src/tools/validation.ts` - add counters per type

**Relevance**: UX, identifies problematic fields or confusing error messages

---

### 10. BullMQ Queue Size

**Metric**: Number of pending jobs in queue

**How to measure**: Query Redis to count jobs in BullMQ queue

**Where to instrument**: Metrics endpoint or periodic job that queries `bull:messages:waiting`

**Relevance**: Detects backlog, indicates if worker is processing successfully

---

### 11. Timeout Rate per API

**Metric**: Percentage of timeouts per service (Z-API, ViaCEP, OpenAI)

**How to measure**: Count `AbortError` or "timeout" errors in logs per service

**Where to instrument**: `src/tools/zapi.ts`, `src/tools/viacep.ts`, `src/services/openai.ts` - detect `AbortError` or timeout messages

**Relevance**: Indicates if timeouts are too aggressive or APIs are slow

---

### 12. Conversational Mode vs State Machine Usage Rate

**Metric**: Percentage of messages processed in conversational mode vs state machine

**How to measure**: Count logs "Processing message (conversational mode)" vs "Processing message (state machine mode)"

**Where to instrument**: `src/agents/orchestrator.ts` - counter per mode at processing start

**Relevance**: Indicates conversational mode effectiveness, detects frequent fallbacks

---

**Evidence**: No current instrumentation beyond logs. Suggestions above are for future implementation.
