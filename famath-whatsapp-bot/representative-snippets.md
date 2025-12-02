# Representative Snippets

## Fastify Webhook Handler

Clean webhook reception with validation:

```typescript
fastify.post('/webhook/whatsapp', async (request, reply) => {
    const webhookData = WebhookDataSchema.parse(request.body);
    
    if (webhookData.fromMe) return reply.code(200).send({ status: 'ignored' });
    if (!webhookData.message.text?.trim()) return reply.code(200).send({ status: 'ignored' });
    
    await messageQueue.add('process-message', webhookData);
    return reply.code(200).send({ status: 'queued' });
});
```

## Orchestrator

State machine routing:

```typescript
export async function processMessage(phone: string, messageText: string): Promise<void> {
    if (env.CONVERSATIONAL_MODE) {
        return await handleConversationalMessage(phone, messageText);
    }
    
    const state = await redis.get(`${STATE_PREFIX}${phone}`) || ConversationState.INITIAL;
    await routeToHandler(phone, state, messageText);
}
```

## Intent Classification

LLM-powered intent detection:

```typescript
export async function classifyIntent(messageText: string): Promise<Intent> {
    const response = await openai.chat.completions.create({
        model: 'gpt-4o-mini',
        messages: [
            { role: 'system', content: INTENT_PROMPT },
            { role: 'user', content: `Mensagem: "${messageText}"` },
        ],
        temperature: 0.3,
        max_tokens: 10,
    });
    
    return response.choices[0]?.message?.content?.trim().toLowerCase() as Intent;
}
```

## CPF Validation and Lookup

Input validation with database lookup:

```typescript
export async function handleCpfInput(phone: string, messageText: string) {
    const cpf = cleanCpf(messageText);
    if (!validateCpf(cpf)) {
        await sendWhatsAppMessage(phone, 'CPF inválido.');
        return;
    }
    
    const candidate = await getCandidateByCpf(cpf);
    if (candidate) {
        const enrollment = await getEnrollmentByCandidateId(candidate.id);
        if (enrollment) {
            await sendWhatsAppMessage(phone, `Protocolo: ${enrollment.protocol_number}`);
            return;
        }
    }
    
    await sendWhatsAppMessage(phone, 'Qual seu nome completo?');
    await redis.setex(`${STATE_PREFIX}${phone}`, TTL, ConversationState.COLLECTING_NAME);
}
```

## Redis State Management

State storage with TTL:

```typescript
const stateKey = `${REDIS_KEY_PREFIXES.STATE}${phone}`;
await redis.setex(stateKey, REDIS_TTL.STATE, ConversationState.COLLECTING_NAME);

const tempDataKey = `${REDIS_KEY_PREFIXES.TEMP_DATA}${phone}`;
await redis.setex(tempDataKey, REDIS_TTL.TEMP_DATA, JSON.stringify({ cpf, name }));
```

## BullMQ Queue Setup

Job queue configuration:

```typescript
const messageQueue = new Queue('messages', {
    connection: getRedisBullMQ(),
    defaultJobOptions: {
        attempts: 3,
        backoff: { type: 'exponential', delay: 2000 },
    },
});

messageQueue.process('process-message', async (job) => {
    const { from, message } = job.data;
    await processMessage(from, message.text);
});
```

## Address Validation

ViaCEP integration:

```typescript
export async function fetchAddressFromViaCep(cep: string): Promise<Address | null> {
    const cleanCep = cleanCep(cep);
    if (!validateCep(cleanCep)) return null;
    
    const response = await fetch(`https://viacep.com.br/ws/${cleanCep}/json/`);
    const data = await response.json();
    
    return data.erro ? null : {
        street: data.logradouro,
        neighborhood: data.bairro,
        city: data.localidade,
        state: data.uf,
    };
}
```

## Protocol Generation

Cryptographically secure protocol and password generation:

```typescript
import { randomBytes } from 'crypto';

export function generateProtocol(): string {
    const random = randomBytes(4).toString('hex').toUpperCase();
    return `FAM${random}`;
}

export function generatePassword(): string {
    return randomBytes(8).toString('base64').slice(0, 8).toUpperCase();
}

export async function hashPassword(password: string): Promise<string> {
    return await bcrypt.hash(password, 10);
}
```

