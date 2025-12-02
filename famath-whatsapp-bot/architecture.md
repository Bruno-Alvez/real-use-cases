# Architecture

## System Overview

Monolithic service: Fastify web server and BullMQ worker run in same Node.js process. Optimized for cost and simplicity. Redis for state machine and job queue. Supabase for data persistence.

## Architecture Pattern

**Monolithic Design**: Single process handles webhooks and job processing. Reduces infrastructure costs. Suitable for low-volume use case.

```
Webhook → Fastify → BullMQ Queue → Worker → Orchestrator → Handlers
```

## Backend Architecture

### Fastify Server

High-performance web server for webhook reception:

```typescript
fastify.post('/webhook/whatsapp', async (request, reply) => {
    const webhookData = WebhookDataSchema.parse(request.body);
    
    if (webhookData.fromMe) return reply.code(200).send({ status: 'ignored' });
    
    await messageQueue.add('process-message', webhookData);
    return reply.code(200).send({ status: 'queued' });
});
```

Webhook validation with Zod. Messages queued for async processing. Self-messages ignored.

### BullMQ Worker

Background job processor in same process:

```typescript
messageQueue.process('process-message', async (job) => {
    const { from, message } = job.data;
    await processMessage(from, message.text);
});
```

Jobs processed asynchronously. Automatic retries on failure. Redis-backed queue.

### Orchestrator

State machine controller routes messages to handlers:

```typescript
export async function processMessage(phone: string, messageText: string): Promise<void> {
    if (env.CONVERSATIONAL_MODE) {
        return await handleConversationalMessage(phone, messageText);
    }
    
    const currentState = await redis.get(`${STATE_PREFIX}${phone}`) || ConversationState.INITIAL;
    await routeToHandler(phone, currentState, messageText);
}
```

Two modes: conversational (LLM-powered) or state machine (structured flow).

## State Management

Redis stores conversation state with TTL:

```typescript
const stateKey = `${REDIS_KEY_PREFIXES.STATE}${phone}`;
await redis.setex(stateKey, REDIS_TTL.STATE, ConversationState.COLLECTING_NAME);
```

24-hour TTL. State transitions logged for debugging. Temp data stored separately.

## Data Layer

Supabase for data persistence:

```typescript
export async function getCandidateByCpf(cpf: string): Promise<Candidate | null> {
    const { data } = await supabase
        .from('candidates')
        .select('*')
        .eq('cpf', cpf)
        .single();
    return data;
}
```

Shared database with web systems. Type-safe queries. Service role key for admin operations.

## AI Integration

OpenAI for intent classification and conversational mode:

```typescript
export async function classifyIntent(messageText: string): Promise<Intent> {
    const response = await openai.chat.completions.create({
        model: 'gpt-4o-mini',
        messages: [
            { role: 'system', content: systemPrompt },
            { role: 'user', content: userPrompt },
        ],
        temperature: 0.3,
    });
    return response.choices[0]?.message?.content?.trim() as Intent;
}
```

Intent classification for routing. Conversational mode for natural interactions. Low temperature for deterministic responses.

## Enrollment Flow

Structured flow with validation:

1. CPF collection → validation → candidate lookup
2. If exists: show enrollment status or courses
3. If new: collect name, email, birth date, CEP, phone, marital status
4. Show courses → select course → select shift → select modality
5. Generate protocol and password → create enrollment → send credentials

Each step validates input. State transitions tracked in Redis. Error handling with user-friendly messages.

## Tools Layer

Specialized tools for external integrations:

- **validation.ts**: CPF, email, CEP validation
- **viacep.ts**: Address lookup via ViaCEP API
- **protocol.ts**: Protocol and password generation
- **zapi.ts**: WhatsApp message sending via Evolution API
- **supabase.ts**: Database CRUD operations

Separation of concerns. Reusable utilities. Type-safe interfaces.

