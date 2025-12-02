# Key Flows

## Webhook Reception Flow

Evolution API → Fastify webhook → validation → queue:

```typescript
fastify.post('/webhook/whatsapp', async (request, reply) => {
    const webhookData = WebhookDataSchema.parse(request.body);
    
    if (webhookData.fromMe) return reply.code(200).send({ status: 'ignored' });
    if (!webhookData.message.text?.trim()) return reply.code(200).send({ status: 'ignored' });
    
    await messageQueue.add('process-message', webhookData);
    return reply.code(200).send({ status: 'queued' });
});
```

Flow: Webhook → Zod validation → duplicate check → queue → immediate 200 response.

## Message Processing Flow

Queue job → orchestrator → handler → response:

```typescript
messageQueue.process('process-message', async (job) => {
    const { from, message } = job.data;
    await processMessage(from, message.text);
});

async function processMessage(phone: string, messageText: string) {
    if (env.CONVERSATIONAL_MODE) {
        return await handleConversationalMessage(phone, messageText);
    }
    
    const state = await redis.get(`${STATE_PREFIX}${phone}`) || ConversationState.INITIAL;
    await routeToHandler(phone, state, messageText);
}
```

Async processing prevents webhook timeouts. State machine routes to appropriate handler.

## Enrollment Flow

CPF → validation → lookup → data collection → enrollment:

```typescript
export async function handleCpfInput(phone: string, messageText: string) {
    const cpf = cleanCpf(messageText);
    if (!validateCpf(cpf)) {
        await sendWhatsAppMessage(phone, 'CPF inválido. Tente novamente.');
        return;
    }
    
    const candidate = await getCandidateByCpf(cpf);
    
    if (candidate) {
        const enrollment = await getEnrollmentByCandidateId(candidate.id);
        if (enrollment) {
            await sendWhatsAppMessage(phone, `Protocolo: ${enrollment.protocol_number}`);
            return;
        }
        await showCourses(phone);
        return;
    }
    
    await sendWhatsAppMessage(phone, 'Qual seu nome completo?');
    await redis.setex(`${STATE_PREFIX}${phone}`, TTL, ConversationState.COLLECTING_NAME);
}
```

CPF validation. Candidate lookup. Branching logic based on existence. State transitions.

## Conversational Mode Flow

LLM-powered natural conversation:

```typescript
export async function handleConversationalMessage(phone: string, messageText: string) {
    const history = await getConversationHistory(phone);
    
    const response = await openai.chat.completions.create({
        model: 'gpt-4o-mini',
        messages: [
            { role: 'system', content: MARIA_THEREZA_PERSONA },
            ...history,
            { role: 'user', content: messageText },
        ],
        temperature: 0.7,
    });
    
    const reply = response.choices[0]?.message?.content;
    await sendWhatsAppMessage(phone, reply);
    await saveToHistory(phone, messageText, reply);
}
```

Conversation history maintained. Persona-driven responses. Natural language understanding.

## Intent Classification Flow

Message → LLM classification → routing:

```typescript
export async function classifyIntent(messageText: string): Promise<Intent> {
    const response = await openai.chat.completions.create({
        model: 'gpt-4o-mini',
        messages: [
            { role: 'system', content: INTENT_CLASSIFICATION_PROMPT },
            { role: 'user', content: `Mensagem: "${messageText}"` },
        ],
        temperature: 0.3,
        max_tokens: 10,
    });
    
    const intent = response.choices[0]?.message?.content?.trim().toLowerCase();
    return ['enrollment', 'status_query', 'general_question', 'handoff'].includes(intent)
        ? intent as Intent
        : 'enrollment';
}
```

Low temperature for deterministic classification. Fallback to enrollment on error.

## Address Validation Flow

CEP → ViaCEP API → address data:

```typescript
export async function fetchAddressFromViaCep(cep: string): Promise<Address | null> {
    const cleanCep = cleanCep(cep);
    if (!validateCep(cleanCep)) return null;
    
    const response = await fetch(`https://viacep.com.br/ws/${cleanCep}/json/`);
    const data = await response.json();
    
    if (data.erro) return null;
    
    return {
        street: data.logradouro,
        neighborhood: data.bairro,
        city: data.localidade,
        state: data.uf,
    };
}
```

CEP validation. External API call. Error handling. Address data returned.

