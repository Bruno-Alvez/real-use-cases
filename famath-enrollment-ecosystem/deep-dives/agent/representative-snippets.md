# Representative Code Snippets

## 1. Queue/Job Handler

**File:** `src/queue.ts`

```typescript
export const messageWorker = new Worker(
  BULLMQ_CONFIG.QUEUE_NAME,
  async (job) => {
    const data = ProcessedMessageSchema.parse(job.data);

    logger.info(
      {
        jobId: job.id,
        phone: data.from,
        event: 'job_started',
      },
      'Processing message job'
    );

    await processMessage(data.from, data.message.text);

    logger.info({ jobId: job.id, phone: data.from }, 'Message job completed');
  },
  {
    connection: getRedisBullMQ(),
    concurrency: BULLMQ_CONFIG.CONCURRENCY,
  }
);
```

**Context:** BullMQ worker processes enqueued messages. Validates data with Zod, logs start/end, calls orchestrator.

## 2. Webhook Ingestion

**File:** `src/index.ts`

```typescript
fastify.post('/webhook/whatsapp', async (request, reply) => {
  try {
    const body = request.body as unknown;
    logger.debug({ rawBody: body }, 'Webhook raw payload received');

    const webhookData = WebhookDataSchema.parse(body);

    if (webhookData.fromMe) {
      logger.debug({ phone: webhookData.from }, 'Ignoring message from bot');
      return reply.code(200).send({ status: 'ignored', reason: 'fromMe' });
    }

    if (!webhookData.message.text || webhookData.message.text.trim() === '') {
      logger.debug({ phone: webhookData.from }, 'Ignoring empty message');
      return reply.code(200).send({ status: 'ignored', reason: 'empty' });
    }

    logger.info(
      {
        phone: webhookData.from,
        messageLength: webhookData.message.text.length,
        event: webhookData.event,
        messagePreview: webhookData.message.text.slice(0, 100),
      },
      'Webhook received - queuing message'
    );

    await messageQueue.add('process-message', webhookData, {
      jobId: `${webhookData.from}-${Date.now()}`,
    });

    reply.code(200).send({ status: 'queued' });
  } catch (error) {
    logger.error(
      { 
        error,
        errorMessage: error instanceof Error ? error.message : String(error),
        rawBody: request.body 
      }, 
      'Webhook validation error'
    );

    reply.code(400).send({ 
      error: 'Invalid webhook payload', 
      details: error instanceof Error ? error.message : String(error) 
    });
  }
});
```

**Context:** Endpoint receives WhatsApp webhooks. Validates payload, ignores bot/empty messages, enqueues job, responds quickly.

## 3. Retry/Backoff Usage

**File:** `src/tools/viacep.ts`

```typescript
export async function fetchAddressFromViaCep(
  cep: string, 
  retries = 2
): Promise<ViaCepResponse | null> {
  const cleaned = cep.replace(/\D/g, '');

  if (cleaned.length !== 8) {
    throw new ViaCepError('Invalid CEP format', cep);
  }

  let lastError: unknown;
  
  for (let attempt = 0; attempt <= retries; attempt++) {
    try {
      if (attempt > 0) {
        logger.info({ cep: cleaned, attempt: attempt + 1 }, 'Retrying ViaCEP request');
        await new Promise(resolve => setTimeout(resolve, 1000 * attempt));
      }
      
      const response = await fetch(`https://viacep.com.br/ws/${cleaned}/json/`, {
        signal: AbortSignal.timeout(10000),
      });

      if (!response.ok) {
        logger.error({ 
          cep: cleaned, 
          status: response.status, 
          statusText: response.statusText 
        }, 'ViaCEP HTTP error');
        throw new ViaCepError(`HTTP ${response.status}: ${response.statusText}`, cep);
      }

      const data = await response.json() as { erro?: boolean; [key: string]: unknown };

      if (data.erro === true) {
        logger.warn({ cep: cleaned }, 'CEP not found in ViaCEP');
        return null;
      }

      const validated = ViaCepResponseSchema.parse(data);
      logger.info(
        { cep: cleaned, city: validated.localidade, state: validated.uf },
        'ViaCEP lookup successful'
      );

      return validated;
    } catch (error) {
      lastError = error;
      
      if (error instanceof ViaCepError && 
          (error.message.includes('Invalid') || error.message.includes('CEP format'))) {
        throw error;
      }
      
      if (attempt === retries) break;
      
      const isRetriable = error instanceof Error && 
        (error.name === 'AbortError' || 
         error.message.includes('timeout') ||
         error.message.includes('network'));
      
      if (!isRetriable) {
        throw error instanceof ViaCepError ? error : 
          new ViaCepError('Error fetching address from ViaCEP', cep, error);
      }
      
      logger.warn({ cep: cleaned, attempt: attempt + 1, error }, 'ViaCEP request failed, will retry');
    }
  }
  
  if (lastError instanceof ViaCepError) throw lastError;
  
  if (lastError instanceof Error && 
      (lastError.name === 'AbortError' || lastError.message.includes('timeout'))) {
    logger.error({ cep: cleaned, error: lastError }, 'ViaCEP timeout error after retries');
    throw new ViaCepError('Timeout fetching CEP from ViaCEP (please try again)', cep, lastError);
  }

  logger.error({ cep: cleaned, error: lastError }, 'ViaCEP API error after retries');
  throw new ViaCepError('Error fetching address from ViaCEP after multiple attempts', cep, lastError);
}
```

**Context:** Manual retry with progressive backoff (1s, 2s). Doesn't retry validation errors, only timeouts/network errors.

## 4. Integration Client Wrapper

**File:** `src/tools/zapi.ts`

```typescript
export async function sendWhatsAppMessage(
  phone: string, 
  text: string
): Promise<void> {
  const env = getEnv();

  if (process.env.TEST_MODE === 'true') {
    logger.info(
      { phone: sanitizePhone(phone), messageLength: text.length },
      'BOT (TEST MODE): Would send message'
    );
    return;
  }

  try {
    const url = `${env.ZAPI_URL}/instances/${env.ZAPI_INSTANCE_ID}/token/${env.ZAPI_TOKEN}/send-text`;
    
    const authToken = env.ZAPI_AUTH_TOKEN || env.ZAPI_TOKEN;
    
    const headers: Record<string, string> = {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${authToken}`
    };
    
    if (env.ZAPI_CLIENT_TOKEN) {
      headers['Client-Token'] = env.ZAPI_CLIENT_TOKEN;
    }
    
    const response = await fetch(url, {
      method: 'POST',
      headers,
      body: JSON.stringify({
        phone: phone,
        message: text,
      }),
      signal: AbortSignal.timeout(20000),
    });

    if (!response.ok) {
      const errorText = await response.text();
      let errorBody;
      try {
        errorBody = JSON.parse(errorText);
      } catch {
        errorBody = errorText;
      }

      logger.error(
        {
          phone: sanitizePhone(phone),
          status: response.status,
          statusText: response.statusText,
          errorBody,
        },
        'Z-API HTTP error'
      );
      
      throw new ZApiError(
        `HTTP ${response.status}: ${JSON.stringify(errorBody)}`,
        phone
      );
    }

    const responseData = await response.json() as { messageId?: string; [key: string]: unknown };
    
    logger.info(
      { 
        phone: sanitizePhone(phone), 
        messageLength: text.length,
        messageId: responseData.messageId || 'unknown',
      },
      'WhatsApp message sent via Z-API'
    );
  } catch (error) {
    if (error instanceof ZApiError) {
      throw error;
    }

    logger.error(
      {
        phone: sanitizePhone(phone),
        error,
        errorName: error instanceof Error ? error.name : 'Unknown',
        errorMessage: error instanceof Error ? error.message : String(error),
      },
      'Z-API network error'
    );

    throw new ZApiError('Failed to send WhatsApp message via Z-API', phone, error);
  }
}
```

**Context:** Z-API client with 20s timeout, configurable headers, structured error handling, test mode.

## 5. Database Transaction Boundary

**File:** `src/tools/supabase.ts`

```typescript
export async function createEnrollment(enrollmentData: {
  protocol_number: string;
  student_id: string;
  course_id: string;
  entry_modality_id: string;
  chosen_shift: string;
  status?: string;
  current_step?: string;
}): Promise<Enrollment> {
  const supabase = getSupabaseAdmin();

  const { data, error } = await supabase
    .from('enrollments')
    .insert({
      ...enrollmentData,
      status: enrollmentData.status || 'pending',
      current_step: enrollmentData.current_step || 'Aguardando Documentos',
    })
    .select()
    .single();

  if (error) {
    logger.error(
      {
        studentId: enrollmentData.student_id,
        courseId: enrollmentData.course_id,
        error,
      },
      'Error creating enrollment'
    );
    throw error;
  }

  logger.info(
    { enrollmentId: data.id, protocolNumber: enrollmentData.protocol_number },
    'Enrollment created'
  );

  return data as Enrollment;
}
```

**Context:** Write operation in Supabase. No explicit transaction (Supabase doesn't easily support multi-table transactions). Student and enrollment creation are separate operations.

## 6. State Machine Update

**File:** `src/agents/enrollmentFlow.ts`

```typescript
export async function handleCpfInput(
  phone: string, 
  messageText: string
): Promise<void> {
  const redis = getRedis();
  const cpf = cleanCpf(messageText);

  if (!validateCpf(cpf)) {
    await sendWhatsAppMessage(
      phone,
      'Invalid CPF format. Please check if you entered all 11 digits correctly.'
    );
    return;
  }

  const tempDataKey = `${REDIS_KEY_PREFIXES.TEMP_DATA}${phone}`;
  await redis.setex(tempDataKey, REDIS_TTL.TEMP_DATA, JSON.stringify({ cpf }));

  await redis.setex(
    `${REDIS_KEY_PREFIXES.STATE}${phone}`,
    REDIS_TTL.STATE,
    ConversationState.CHECKING_CANDIDATE
  );

  try {
    const candidate = await getCandidateByCpf(cpf);
    // ... rest of flow
  } catch (error) {
    logger.error({ 
      phone: sanitizePhone(phone), 
      cpf: sanitizeCpf(cpf), 
      error 
    }, 'Error checking candidate');

    await redis.setex(
      `${REDIS_KEY_PREFIXES.STATE}${phone}`,
      REDIS_TTL.STATE,
      ConversationState.INITIAL
    );

    await sendWhatsAppMessage(
      phone,
      'Something went wrong on our end. Please wait a few minutes and try again.'
    );
  }
}
```

**Context:** State update in Redis with TTL. On error, state is reset to INITIAL. `setex` operations are separate (not transactional), but acceptable for this use case with concurrency 1.

## 7. Protocol Generation

**File:** `src/tools/protocol.ts`

```typescript
export async function generateProtocol(): Promise<string> {
  const supabase = getSupabase();

  const { data, error } = await supabase
    .from('enrollments')
    .select('protocol_number')
    .like('protocol_number', `FAM${PROTOCOL_FORMAT.YEAR}%`)
    .order('protocol_number', { ascending: false })
    .limit(1)
    .single();

  let nextNumber = 1;

  if (!error && data?.protocol_number) {
    const match = data.protocol_number.match(/FAM\d{4}(\d{4})$/);
    if (match) {
      nextNumber = parseInt(match[1] || '0', 10) + 1;
    }
  } else if (error && error.code !== 'PGRST116') {
    logger.warn({ error }, 'Error fetching last protocol, starting from 1');
  }

  return PROTOCOL_FORMAT.FORMAT(nextNumber);
}
```

**Context:** Protocol generation searches last number in database and increments. Race condition possible if two jobs run simultaneously, but concurrency is 1, so it doesn't happen.

## 8. Password Hashing (PBKDF2)

**File:** `src/tools/protocol.ts`

```typescript
export function hashPassword(password: string): string {
  const salt = randomBytes(16);
  
  const iterations = 100000;
  const keyLength = 32;
  const digest = 'sha256';
  
  const hash = pbkdf2Sync(password, salt, iterations, keyLength, digest);
  
  const combined = Buffer.concat([salt, hash]);
  
  return combined.toString('base64');
}
```

**Context:** Password hash with PBKDF2 (SHA-256, 100k iterations, 16-byte salt). Format: Base64(salt + hash). Compatible with web system.
