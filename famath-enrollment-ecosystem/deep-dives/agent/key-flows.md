# Key Flows

## 1. Message Reception and Processing

**Trigger**: Webhook POST `/webhook/whatsapp` receives message from Z-API

**Steps**:
1. Validates payload with `WebhookDataSchema` (Zod)
2. Ignores messages from the bot itself (`fromMe === true`)
3. Ignores empty messages
4. Enqueues job in BullMQ with `jobId: {phone}-{timestamp}`
5. Responds 200 immediately

**Failure points**:
- Invalid payload (returns 400)
- Redis unavailable (BullMQ fails to enqueue)
- Worker doesn't process (job stays in queue)

**Handling**:
- Zod validation returns structured error
- BullMQ automatic retry (2 attempts, backoff 5s)
- Structured logs with context

**Evidence**: `src/index.ts`, `src/queue.ts`

---

## 2. Complete Enrollment Flow (Conversational Mode)

**Trigger**: User sends initial message (e.g., "hi", "I want to enroll")

**Steps**:
1. Orchestrator detects conversational mode (`CONVERSATIONAL_MODE=true`)
2. Loads conversation history from Redis (last 10 messages)
3. Loads temporary data collected so far
4. Sends prompt to OpenAI with context (history + data + FAMATH knowledge)
5. LLM responds with natural message and data extraction instructions
6. If data complete: creates student and enrollment in Supabase
7. Generates protocol and password, sends credentials via WhatsApp
8. Saves messages to history (Redis LIST)

**Failure points**:
- OpenAI timeout (30s) or API error
- ViaCEP unavailable (fallback: user provides address manually)
- Supabase fails to create student/enrollment
- Z-API fails to send message

**Handling**:
- OpenAI: 30s timeout, fallback to state machine mode if it fails
- ViaCEP: retry 2x with progressive backoff, if it fails asks for manual address
- Supabase: error is logged, user receives friendly error message
- Z-API: 20s timeout, error is logged but not retried (user can try again)

**Idempotency**: No explicit dedupe. Job ID uses `{phone}-{timestamp}`, but multiple messages from the same user create distinct jobs. State is updated in Redis (separate `setex` operations, not transactional).

**Evidence**: `src/agents/conversationalAgent.ts`, `src/services/openai.ts`, `src/tools/viacep.ts`, `src/tools/zapi.ts`

---

## 3. CPF Validation and Candidate Lookup

**Trigger**: User sends CPF during enrollment flow

**Steps**:
1. Cleans CPF formatting (removes dots and dashes)
2. Validates format (11 digits)
3. Validates checksum (Brazilian algorithm)
4. Searches candidate in Supabase by CPF
5. If found: checks if has active enrollment
6. If has enrollment: shows protocol and offers help
7. If no enrollment: continues data collection flow
8. If not found: starts personal data collection

**Failure points**:
- Invalid CPF (format or checksum)
- Supabase unavailable
- Query timeout

**Handling**:
- Invalid CPF: friendly message asking for correction
- Supabase: implicit client timeout, error is logged and state is reset
- User receives message asking to try again

**Evidence**: `src/tools/validation.ts`, `src/tools/supabase.ts`, `src/agents/enrollmentFlow.ts`

---

## 4. Address Lookup via ViaCEP

**Trigger**: User sends postal code during enrollment flow

**Steps**:
1. Validates postal code format (8 digits)
2. Cleans formatting
3. Makes GET request to `https://viacep.com.br/ws/{cep}/json/`
4. 10 second timeout
5. Validates response with `ViaCepResponseSchema` (Zod)
6. If `erro: true`: returns null (postal code not found)
7. If success: extracts street, neighborhood, city, state
8. Saves to Redis `temp_data` and asks for complement/number

**Failure points**:
- Invalid postal code (format)
- ViaCEP timeout (10s)
- ViaCEP returns HTTP error
- Invalid response (doesn't pass Zod validation)

**Handling**:
- Automatic retry: 2 attempts with progressive backoff (1s, 2s)
- Timeout: error is logged, user can provide address manually
- Postal code not found: returns null, flow continues asking for manual address
- Validation fails: error is logged, user provides address manually

**Idempotency**: None. Each query is independent. Multiple queries of the same postal code make multiple requests.

**Evidence**: `src/tools/viacep.ts`, `src/tools/validation.ts`

---

## 5. Enrollment Creation and Credentials Sending

**Trigger**: Complete data collected, user confirms enrollment

**Steps**:
1. Generates protocol: searches last protocol of the year in Supabase, increments (format: `FAM2025XXXX`)
2. Generates password: `{6firstCPFdigits}@{nameInitials}` (e.g., `408604@Bt`)
3. Password hash: PBKDF2 with SHA-256, 100k iterations, 16-byte salt (compatible with web system)
4. Creates student in Supabase (`students` table)
5. Creates enrollment in Supabase (`enrollments` table)
6. Sends WhatsApp message with protocol and password
7. Updates state to `ENROLLMENT_CREATED`

**Failure points**:
- Protocol generation fails (Supabase query)
- Password hash fails (unlikely, local operation)
- Student creation fails (duplicate CPF, constraint violation)
- Enrollment creation fails (foreign key violation, etc.)
- WhatsApp message sending fails

**Handling**:
- Protocol: if query fails, starts from 1
- Duplicate student: error is logged, user is informed (should have been detected before)
- Enrollment: error is logged, user receives error message
- WhatsApp: 20s timeout, error is logged but enrollment was already created (credentials can be queried later)

**Idempotency**: None. If job is reprocessed, will attempt to create student/enrollment again. Unique CPF in database prevents duplicates, but may generate error.

**Evidence**: `src/tools/protocol.ts`, `src/tools/supabase.ts`, `src/tools/zapi.ts`
