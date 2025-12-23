# Reliability and Operations

## Timeouts

All external HTTP requests have configured timeout:

- **Z-API (message sending)**: 20 seconds (`AbortSignal.timeout(20000)`)
- **ViaCEP (address lookup)**: 10 seconds (`AbortSignal.timeout(10000)`)
- **OpenAI (LLM calls)**: 30 seconds (configured in OpenAI client)

**Evidence**: `src/tools/zapi.ts`, `src/tools/viacep.ts`, `src/services/openai.ts`, `src/config/constants.ts`

---

## Retries and Backoff

### BullMQ Jobs

Failed jobs are automatically retried:
- **Attempts**: 2 (in addition to initial attempt = 3 total)
- **Backoff**: Exponential, initial delay of 5 seconds
- **Configuration**: `attempts: 2, backoff: { type: 'exponential', delay: 5000 }`

**Evidence**: `src/config/constants.ts`, `src/queue.ts`

### ViaCEP

Manual retry implemented in code:
- **Attempts**: 2 retries (total 3 attempts)
- **Backoff**: Manual progressive (1s, 2s)
- **Logic**: Doesn't retry on validation errors or postal code not found, only on timeouts/network errors

**Evidence**: `src/tools/viacep.ts`

### Other APIs

- **Z-API**: No retry (timeout or final error)
- **OpenAI**: No explicit retry (BullMQ retries entire job if it fails)
- **Supabase**: No explicit retry (Supabase client may have internal retry)

---

## Idempotency

No explicit idempotency implemented. Jobs are identified by `jobId: {phone}-{timestamp}`, but multiple messages from the same user create distinct jobs.

**Consequence**: If webhook receives the same message twice (e.g., Z-API retry), two jobs will be created and processed.

**Existing protections**:
- Unique CPF constraint in PostgreSQL prevents duplicate student creation
- Protocol generation searches last number and increments (theoretically possible race condition, but concurrency is 1)

**Evidence**: `src/index.ts`, `src/tools/supabase.ts`, `src/tools/protocol.ts`

---

## Rate Limiting

No rate limiting implemented on webhook or API calls.

**Consequence**: If Z-API sends many messages simultaneously or external APIs have rate limits, system may fail without backpressure control.

**Considerations**:
- Z-API controls rate of messages received on webhook
- External API rate limits (OpenAI, ViaCEP) are not handled in code
- Concurrency 1 in worker reduces impact, but doesn't prevent rate limits

**Evidence**: `src/index.ts` has no rate limiting, `src/queue.ts` doesn't implement throttling

---

## Dead-Letter Queue

No dead-letter queue implemented. Jobs that fail after all attempts are removed from queue and only logged.

**Consequence**: Failed jobs cannot be manually reprocessed. Errors need to be investigated via logs. Persistent failures (e.g., bug in code) are lost after 2 retries.

**Evidence**: `src/queue.ts` - failure events are logged, but no DLQ configured

---

## Logging and Observability

### Implementation

- **Library**: Pino (structured logging)
- **Format**: JSON in production, pretty-print in development
- **Levels**: fatal, error, warn, info, debug, trace (configurable via `LOG_LEVEL`)
- **Sanitization**: CPF, phone, email are sanitized in logs (last digits masked)

### Context in Logs

Logs include structured context:
- `phone`: sanitized phone
- `cpf`: sanitized CPF
- `jobId`: BullMQ job ID
- `state`: conversation state
- `event`: event type (e.g., 'job_started', 'state_transition')
- `error`: complete error object

### What is Logged

- Webhook received (info)
- Processing message job / Message job completed / Job failed (info/error)
- State transitions (info)
- External API calls (info/debug)
- Errors with complete stack trace (error)
- Failed validations (warn)

### What is NOT Logged

- Complete user messages (only preview of 100 chars)
- Passwords (never logged)
- API tokens (masked)

**Evidence**: `src/utils/logger.ts`, `src/utils/sanitizers.ts`, use of `logger.info/error/warn` throughout code

---

## CI/CD

### Docker

- **Build**: Multi-stage (builder + production)
- **Base**: `node:20-alpine`
- **Optimization**: Only production dependencies in final image
- **Command**: `node dist/index.js`

**Evidence**: `Dockerfile`

### Docker Compose

- **Services**: `redis` (Redis 7-alpine) and `app` (application)
- **Healthcheck**: Redis has healthcheck configured
- **Dependencies**: App depends on Redis being healthy
- **Volumes**: Redis data persists in named volume

**Evidence**: `docker-compose.yml`

### Render Deployment

- **Type**: Web Service (monolithic: web + worker)
- **Build**: `npm install && npm run build`
- **Start**: `node dist/index.js`
- **Redis**: Separate managed service

**Evidence**: `render.yaml` (if exists), `README.md`

### CI/CD Pipeline

No CI/CD pipeline configured (GitHub Actions, GitLab CI, etc.). Deploy is manual via Render dashboard.

**Future improvements**:
- GitHub Actions for automated tests
- Automatic deploy on push to main
- Linting and type-check in PRs

---

## Basic Runbook

### How to Verify System is Working

1. **Health check**: `GET /health` should return `{ status: 'ok', timestamp: '...' }`
2. **Logs**: Check logs for recent errors
3. **Redis**: Check Redis connection (logs show "Redis connected")
4. **Queue**: Check if jobs are being processed (logs "Message job completed")

### Troubleshooting

#### Webhook not receiving messages

1. Check if Z-API is sending webhooks (provider logs)
2. Check webhook URL configured correctly
3. Check if server is accessible (ngrok in dev, Render in prod)
4. Check webhook error logs (`src/index.ts`)

#### Jobs not being processed

1. Check if worker is running (logs "BullMQ worker initialized")
2. Check Redis connection (logs "Redis connected (BullMQ)")
3. Check if there are jobs in queue (use Redis CLI: `KEYS bull:messages:*`)
4. Check worker error logs (`src/queue.ts`)

#### Messages not being sent

1. Check Z-API credentials (env vars: `ZAPI_TOKEN`, `ZAPI_INSTANCE_ID`)
2. Check Z-API error logs (`src/tools/zapi.ts`)
3. Check timeout (20s may be insufficient if API is slow)
4. Check Z-API rate limits

#### OpenAI fails

1. Check `OPENAI_API_KEY` is configured
2. Check OpenAI rate limits (OpenAI dashboard)
3. Check timeout (30s may be insufficient)
4. Check error logs (`src/agents/intentClassifier.ts`, `src/agents/conversationalAgent.ts`)

#### ViaCEP fails

1. Check if postal code is valid (8 digit format)
2. Check retry logs (`src/tools/viacep.ts`)
3. Check if API is accessible (curl `https://viacep.com.br/ws/01310100/json/`)
4. System has fallback: user can provide address manually

#### State lost in Redis

1. Check TTL (24h expires automatically)
2. Check if Redis is persisting data (AOF enabled in production)
3. Check Redis error logs (`src/services/redis.ts`)

### Where to Look for Logs

- **Development**: Console (pretty-print)
- **Production**: Render logs dashboard or stdout/stderr of container
- **Structure**: JSON with fields `level`, `time`, `msg`, `phone`, `jobId`, `error`, etc.

### Useful Commands

```bash
# Clear user session (dev)
npm run clear:session -- {phone}

# Test Supabase integration
npm run test:supabase

# Test OpenAI integration
npm run test:openai

# Test ViaCEP integration
npm run test:viacep

# Check TypeScript types
npm run type-check

# Build
npm run build
```

**Evidence**: `scripts/clear-session.ts`, `package.json` (scripts), logs throughout code
