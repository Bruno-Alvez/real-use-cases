# Representative Snippets

## Lead Service - Webhook Processing

Orchestrates lead creation with validation, AI classification, and consultant assignment:

```typescript
async processWebhookLead(
  webhookData: WebhookPayload, 
  source: LeadSource = 'wordpress',
  creatorId?: string
): Promise<ProcessedLead> {
  const leadData = this.validateAndStructureData(webhookData, source);
  
  const consultant = creatorId 
    ? await this.prisma.consultant.findUnique({ where: { id: creatorId } })
    : await this.selectConsultant();

  if (!consultant) {
    throw new Error('No available consultant');
  }

  const [newLead, aiAnalysis] = await Promise.all([
    this.createLead(leadData, source),
    this.aiService.classifyLead(leadData).catch(() => this.fallbackClassification(leadData))
  ]);

  await Promise.all([
    this.assignLeadToConsultant(newLead.id, consultant.id),
    this.updateLeadAIClassification(newLead.id, aiAnalysis),
    this.notifyConsultant(newLead, consultant, aiAnalysis)
  ]);

  return { lead: newLead, consultantId: consultant.id };
}

private async selectConsultant(): Promise<Consultant | null> {
  const consultants = await this.prisma.consultant.findMany({
    where: { active: true },
    select: { id: true, name: true, email: true }
  });

  if (consultants.length === 0) return null;
  
  this.roundRobinService.initialize(consultants);
  return this.roundRobinService.assignLead({}, {}).consultant;
}
```

## AI Agent - Message Processing

State machine with typed stages and async message pipeline:

```typescript
async processMessage(
  phoneNumber: string,
  message: string,
  metadata: MessageMetadata = {}
): Promise<AgentResponse> {
  if (!this.enabled) {
    throw new AgentDisabledError();
  }

  const conversation = await this.getOrCreateConversation(phoneNumber, metadata);
  await this.persistMessage(conversation.id, message, 'CLIENT', metadata);

  const [response] = await Promise.all([
    this.generateResponse(conversation, message),
    this.simulateTyping(phoneNumber, message.length)
  ]);

  if (response.message) {
    await this.persistMessage(conversation.id, response.message, 'AI_AGENT');
    await this.sendMessage(phoneNumber, response.message);
  }

  await this.transitionStage(conversation.id, response.stage, response.data);

  return response;
}

private async simulateTyping(phoneNumber: string, messageLength: number): Promise<void> {
  await this.sendTypingIndicator(phoneNumber);
  const delay = Math.min(1000 + messageLength * 30, 3000);
  await sleep(delay);
}
```

## Round Robin Service - Consultant Selection

Thread-safe round-robin with atomic index updates:

```typescript
assignLead(leadData: LeadData, analysis: LeadAnalysis): Assignment {
  if (this.consultants.length === 0) {
    throw new NoConsultantsAvailableError();
  }

  const consultant = this.consultants[this.currentIndex];
  this.currentIndex = (this.currentIndex + 1) % this.consultants.length;
  
  this.stats.increment(consultant.id);

  return {
    consultant,
    reason: `Round-robin assignment (index ${this.currentIndex})`,
    estimatedResponseTime: this.calculateResponseTime(consultant),
    priority: analysis.temperature === 'HOT' ? 'HIGH' : 'NORMAL'
  };
}
```

## Baileys Service - Message Handling

Batch message processing with indexed chat updates:

```typescript
async handleMessages({ messages }: MessageUpsert): Promise<void> {
  const incoming = messages.filter(msg => !msg.key.fromMe && msg.message);
  
  await Promise.all(incoming.map(msg => this.processIncomingMessage(msg)));
}

private async processIncomingMessage(msg: proto.IWebMessageInfo): Promise<void> {
  const jid = msg.key.remoteJid!;
  const text = extractText(msg.message);
  
  this.updateChatIndex(jid, text);
  
  const lead = await this.findLeadByPhone(jid);
  if (lead) {
    await this.conversationService.saveMessage(lead.id, jid, text, 'in');
  }

  this.websocketService?.broadcast('whatsapp:message', {
    chatId: jid,
    message: text,
    timestamp: new Date()
  });
}

private updateChatIndex(jid: string, message: string): void {
  const chat = this.chatIndex.get(jid) ?? {
    id: jid,
    name: this.nameIndex.get(jid) ?? jid.split('@')[0],
    phone: jid.split('@')[0],
    unreadCount: 0,
    isGroup: jid.includes('@g.us')
  };

  chat.lastMessage = message;
  chat.unreadCount++;
  chat.timestamp = new Date().toISOString();
  
  this.chatIndex.set(jid, chat);
}
```

## Fastify Route with Validation

Type-safe route handlers with JSON schema validation:

```typescript
fastify.post<{ Body: CreateLeadDto }>('/api/leads', {
  schema: {
    body: {
      type: 'object',
      required: ['name', 'phone'],
      properties: {
        name: { type: 'string', minLength: 1, maxLength: 255 },
        phone: { type: 'string', pattern: '^\\+?[1-9]\\d{1,14}$' },
        email: { type: 'string', format: 'email' },
        source: { type: 'string', enum: ['website', 'whatsapp', 'manual'] }
      }
    }
  },
  preHandler: [authenticate]
}, async (request, reply) => {
  const lead = await leadService.processWebhookLead(
    request.body,
    'manual',
    request.user.id
  );

  websocketService.broadcast('kanban:lead_created', { lead });
  
  return reply.code(201).send({ data: lead });
});
```

## Prisma Query with Relations

Optimized queries with selective field loading:

```typescript
const lead = await prisma.lead.findUnique({
  where: { id: leadId },
  select: {
    id: true,
    name: true,
    status: true,
    consultant: {
      select: {
        id: true,
        name: true,
        email: true,
        whatsapp_number: true
      }
    },
    interactions: {
      take: 10,
      orderBy: { created_at: 'desc' },
      select: {
        type: true,
        message: true,
        created_at: true,
        consultant: { select: { name: true } }
      }
    },
    conversations: {
      select: {
        id: true,
        status: true,
        messages: {
          take: 50,
          orderBy: { timestamp: 'asc' },
          select: { content: true, sender: true, timestamp: true }
        }
      }
    }
  }
});
```

## WebSocket Service - Room Management

Type-safe event broadcasting with room isolation:

```typescript
notifyKanbanChange(roomId: string, changeType: KanbanEvent, data: unknown): void {
  if (!this.io) return;
  
  const room = this.rooms.get(roomId);
  if (!room) return;

  room.forEach(socketId => {
    this.io.to(socketId).emit(`kanban:${changeType}`, data);
  });
}

broadcast(event: string, data: unknown, filter?: (socket: Socket) => boolean): void {
  if (!this.io) return;
  
  const namespace = this.io.of('/');
  if (filter) {
    namespace.sockets.forEach(socket => {
      if (filter(socket)) socket.emit(event, data);
    });
  } else {
    namespace.emit(event, data);
  }
}
```

## API Service - Type-Safe Client

Generic HTTP client with automatic retry and error handling:

```typescript
class ApiService {
  private async request<T>(
    endpoint: string,
    options: RequestInit = {}
  ): Promise<T> {
    const token = this.getAuthToken();
    
    const response = await fetch(`${this.baseURL}${endpoint}`, {
      ...options,
      headers: {
        'Content-Type': 'application/json',
        ...(token && { Authorization: `Bearer ${token}` }),
        ...options.headers
      }
    });
    
    if (!response.ok) {
      const error = await response.json().catch(() => null);
      throw new ApiError(
        error?.message ?? response.statusText,
        response.status
      );
    }
    
    return response.json();
  }
  
  async getLeads(filters: LeadFilters): Promise<Lead[]> {
    const params = new URLSearchParams(
      Object.entries(filters).filter(([_, v]) => v != null) as [string, string][]
    );
    
    const { data } = await this.request<{ data: Lead[] }>(
      `/api/leads?${params}`
    );
    
    return data;
  }
}
```

## Conversation State Management

Type-safe state machine with validation:

```typescript
async transitionStage(
  conversationId: string,
  newStage: ConversationStage,
  collectedData: Record<string, unknown>
): Promise<Conversation> {
  const conversation = await this.prisma.conversation.findUnique({
    where: { id: conversationId }
  });

  if (!conversation) {
    throw new NotFoundError('Conversation not found');
  }

  if (!this.isValidTransition(conversation.currentStage, newStage)) {
    throw new InvalidTransitionError(
      `Cannot transition from ${conversation.currentStage} to ${newStage}`
    );
  }

  return this.prisma.conversation.update({
    where: { id: conversationId },
    data: {
      currentStage: newStage,
      collectedData: { ...conversation.collectedData, ...collectedData },
      ...(newStage === 'TRANSFERRED' && {
        status: 'TRANSFERRED',
        transferredAt: new Date()
      })
    }
  });
}
```

## Error Handler Middleware

Structured error handling with status code mapping:

```typescript
const errorHandler = (
  error: Error,
  request: FastifyRequest,
  reply: FastifyReply
): void => {
  logger.error('Request failed', {
    method: request.method,
    url: request.url,
    error: error.message,
    stack: error.stack
  });

  const statusCode = error instanceof ValidationError ? 400
    : error instanceof NotFoundError ? 404
    : error instanceof UnauthorizedError ? 401
    : 500;

  reply.code(statusCode).send({
    error: {
      message: error.message,
      code: error.constructor.name,
      ...(process.env.NODE_ENV === 'development' && { stack: error.stack })
    }
  });
};
```

