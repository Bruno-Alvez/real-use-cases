# Challenges

## Webhook Timeout Prevention

**Problem**: Z-API webhooks timeout if processing takes too long. Need immediate response.

**Solution**: Queue-based architecture. Webhook handler returns 200 immediately. Processing happens asynchronously in BullMQ worker.

```typescript
fastify.post('/webhook/whatsapp', async (request, reply) => {
    await messageQueue.add('process-message', webhookData);
    return reply.code(200).send({ status: 'queued' });
});
```

Immediate response prevents timeouts. Jobs processed asynchronously. Retries on failure.

## Conversation State Management

**Problem**: Need to track conversation state across multiple messages. State must persist between requests.

**Solution**: Redis with TTL. State stored per phone number. 24-hour expiration.

```typescript
const stateKey = `${REDIS_KEY_PREFIXES.STATE}${phone}`;
await redis.setex(stateKey, 86400, ConversationState.COLLECTING_NAME);
```

Fast lookups. Automatic expiration. Separate temp data storage.

## Intent Classification Reliability

**Problem**: User messages ambiguous. Need accurate intent classification for routing.

**Solution**: OpenAI with low temperature. Clear system prompt. Fallback to enrollment.

```typescript
const response = await openai.chat.completions.create({
    model: 'gpt-4o-mini',
    temperature: 0.3, // Deterministic
    max_tokens: 10,
});
```

Low temperature for consistency. Clear prompts. Fallback strategy.

## Data Validation

**Problem**: User input needs validation (CPF, email, CEP). Invalid data causes errors.

**Solution**: Zod schemas for validation. Regex validators. Clear error messages.

```typescript
export function validateCpf(cpf: string): boolean {
    const clean = cpf.replace(/\D/g, '');
    if (clean.length !== 11) return false;
    // CPF validation algorithm
    return isValidCpf(clean);
}
```

Multiple validation layers. User-friendly error messages. Retry mechanism.

## External API Reliability

**Problem**: ViaCEP and other APIs can fail. Need graceful degradation.

**Solution**: Timeout and retry logic. Error handling with user messages. Fallback to manual input.

```typescript
try {
    const address = await fetchAddressFromViaCep(cep);
    if (address) {
        await sendWhatsAppMessage(phone, `Endereço encontrado: ${address.street}`);
    }
} catch (error) {
    await sendWhatsAppMessage(phone, 'CEP não encontrado. Digite o endereço manualmente.');
}
```

Timeout handling. Clear error messages. Manual fallback.

## Message Deduplication

**Problem**: Z-API can send duplicate messages. Need to prevent duplicate processing. Timestamp-based IDs can collide.

**Solution**: Composite job ID with phone, timestamp, and message hash. BullMQ deduplication with TTL.

```typescript
const messageHash = crypto
    .createHash('sha256')
    .update(webhookData.message.text)
    .digest('hex')
    .slice(0, 8);

await messageQueue.add('process-message', webhookData, {
    jobId: `${webhookData.from}-${Date.now()}-${messageHash}`,
    jobIdTtl: 3600000, // 1 hour
});
```

Unique job IDs with collision resistance. BullMQ prevents duplicates within TTL. Idempotent handlers. Handles network retries and duplicate webhooks.

