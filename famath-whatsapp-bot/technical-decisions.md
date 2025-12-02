# Technical Decisions

## Monolithic Architecture

Fastify server and BullMQ worker in same process. Reduces infrastructure costs. Suitable for low-volume use case.

**Benefits**: Single deployment, lower cost (~$14/month), simpler operations.  
**Trade-offs**: Less scalability, single point of failure.

## BullMQ for Job Processing

BullMQ chosen over alternatives for Redis-backed queue with retries:

```typescript
const messageQueue = new Queue('messages', {
    connection: getRedisBullMQ(),
    defaultJobOptions: {
        attempts: 3,
        backoff: { type: 'exponential', delay: 2000 },
    },
});
```

Automatic retries. Exponential backoff. Redis persistence. Job deduplication.

## Two-Mode Architecture

Conversational mode (LLM) and state machine mode (structured):

```typescript
if (env.CONVERSATIONAL_MODE) {
    return await handleConversationalMessage(phone, messageText);
}
// Fallback to state machine
```

Conversational mode for natural interactions. State machine for structured flows. Environment-based switching.

## Redis for State Management

Redis stores conversation state with TTL:

```typescript
await redis.setex(`${STATE_PREFIX}${phone}`, 86400, ConversationState.COLLECTING_NAME);
```

24-hour TTL. Fast lookups. Automatic expiration. Separate clients for BullMQ and general operations.

## Zod for Validation

Zod for runtime validation:

```typescript
const WebhookDataSchema = z.object({
    from: z.string(),
    message: z.object({ text: z.string() }),
    fromMe: z.boolean(),
});
```

Type-safe validation. Clear error messages. Schema composition.

## OpenAI for Intent Classification

OpenAI for intent classification and conversational mode:

```typescript
const response = await openai.chat.completions.create({
    model: 'gpt-4o-mini',
    temperature: 0.3, // Deterministic for classification
    max_tokens: 10,
});
```

Low temperature for classification. Higher temperature for conversational. Cost-effective model (gpt-4o-mini).

## Fastify over Express

Fastify chosen for performance:

- 2x faster than Express
- Built-in schema validation
- TypeScript support
- Lower memory footprint

## TypeScript Strict Mode

Strict TypeScript configuration:

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitAny": true
  }
}
```

Type safety. No `any` types. Better IDE support.

## Pino for Logging

Pino for structured logging:

```typescript
logger.info({ phone: sanitizePhone(phone), state }, 'State transition');
```

JSON format in production. Pretty printed in development. Sensitive data sanitized.

## Evolution API Integration

Evolution API for WhatsApp messaging:

- Native WhatsApp Web support
- Webhook-based message reception
- Simple REST API for sending
- No session management needed

