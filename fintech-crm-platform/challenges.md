# Challenges

## WhatsApp Session Management

**Problem**: Baileys sessions disconnect unexpectedly. Two separate sessions add complexity.

**Solution**: Isolated session directories with `useMultiFileAuthState`. Auto-reconnect with exponential backoff. Persistent disk in production.

```typescript
private handleConnectionUpdate(update: ConnectionUpdate): void {
  if (update.connection === 'close') {
    const shouldReconnect = 
      update.lastDisconnect?.error?.output?.statusCode !== DisconnectReason.loggedOut;
    
    if (shouldReconnect && this.reconnectAttempts < MAX_RECONNECT) {
      this.reconnectAttempts++;
      setTimeout(() => this.initialize(), 5000 * this.reconnectAttempts);
    }
  }
}
```

Sessions persist across restarts. Auto-reconnect handles network issues.

## LLM Response Reliability

**Problem**: Groq API failures break conversation flow.

**Solution**: Fallback chain: LLM → JSON parse → rule-based. Timeout handling. Default responses if AI unavailable.

```typescript
async generateResponse(conversation: Conversation, message: string): Promise<AgentResponse> {
  try {
    const prompt = this.buildPrompt(conversation, message);
    const completion = await this.groqService.generateResponse(prompt);
    return this.parseResponse(completion);
  } catch (error) {
    logger.error('LLM error, using fallback', { error, stage: conversation.currentStage });
    return this.getFallbackResponse(conversation.currentStage);
  }
}
```

System stays operational during outages. Fallback maintains conversation quality.

## Real-time Synchronization

**Problem**: Multiple consultants viewing same Kanban board need real-time updates. WebSocket connections can drop. State divergence on reconnect.

**Solution**: Socket.io with automatic reconnection. Event sourcing pattern: client requests state delta on reconnect. Server maintains room membership and event log.

```typescript
socket.on('reconnect', async () => {
  const lastEventId = socket.handshake.query.lastEventId;
  const delta = await getStateDelta(roomId, lastEventId);
  socket.emit('state_sync', delta);
});

async function getStateDelta(roomId: string, lastEventId: string): Promise<StateDelta> {
  const events = await eventLog.getEventsAfter(roomId, lastEventId);
  return {
    events,
    fullState: events.length > 100 ? await getFullState(roomId) : null
  };
}
```

**Outcome**: Updates propagate instantly. Reconnection transparent. State consistency maintained. Handles network partitions gracefully.

## Lead Assignment Fairness

**Problem**: Simple round-robin can be unfair if consultants have different workloads or availability. Need to balance distribution while keeping algorithm simple.

**Solution**: Round-robin with statistics tracking. Current implementation uses simple rotation, but stats collected for future enhancement. Consultant availability checked before assignment.

```javascript
assignLead(leadData, analysis) {
  const availableConsultants = this.consultants.filter(c => 
    this.isConsultantAvailable(c)
  );
  
  if (availableConsultants.length === 0) {
    availableConsultants = this.consultants; // Fallback to all
  }
  
  const selected = availableConsultants[this.currentIndex % availableConsultants.length];
  this.currentIndex++;
  return selected;
}
```

**Outcome**: Fair distribution over time. Stats available for future load-based improvements.

## Conversation State Management

**Problem**: AI agent needs to track conversation stage and collected data across multiple messages. State must persist and recover from failures.

**Solution**: Database-backed state with Prisma. Conversation record stores current stage, collected data as JSON, and message history. State machine enforces valid transitions.

```javascript
async updateConversationState(conversationId, newStage, collectedData) {
  const conversation = await this.prisma.conversation.findUnique({
    where: { id: conversationId }
  });
  
  // Validate stage transition
  const validTransitions = this.getValidTransitions(conversation.currentStage);
  if (!validTransitions.includes(newStage)) {
    throw new Error(`Invalid transition: ${conversation.currentStage} -> ${newStage}`);
  }
  
  return await this.prisma.conversation.update({
    where: { id: conversationId },
    data: {
      currentStage: newStage,
      collectedData: { ...conversation.collectedData, ...collectedData }
    }
  });
}
```

**Outcome**: Reliable state tracking. Conversations resume correctly after server restarts.

## Typing Indicator Simulation

**Problem**: AI responses appear instant, breaking human-like conversation flow. Need to simulate natural typing delays.

**Solution**: Calculate delay based on response length. Send typing indicator before delay, then send message after delay completes.

```javascript
calculateTypingDelay(message) {
  const baseDelay = 1000; // 1 second base
  const charDelay = 50; // 50ms per character
  const maxDelay = 3000; // 3 seconds max
  
  const calculated = baseDelay + (message.length * charDelay);
  return Math.min(calculated, maxDelay);
}

async processMessage(phoneNumber, message) {
  await this.sendTypingIndicator(phoneNumber);
  const delay = this.calculateTypingDelay(response);
  await new Promise(resolve => setTimeout(resolve, delay));
  await this.sendMessage(phoneNumber, response);
}
```

**Outcome**: More natural conversation flow. Users perceive agent as more human-like.

## Memory Management for Chat Index

**Problem**: In-memory chat index grows unbounded. Can cause memory leaks in long-running processes. O(n) eviction is inefficient.

**Solution**: LRU cache with O(1) operations. Linked list + hash map for efficient eviction. Persist important conversations to database.

```typescript
class LRUCache<K, V> {
  private capacity: number;
  private cache = new Map<K, V>();

  get(key: K): V | undefined {
    if (!this.cache.has(key)) return undefined;
    const value = this.cache.get(key)!;
    this.cache.delete(key);
    this.cache.set(key, value); // Move to end
    return value;
  }

  set(key: K, value: V): void {
    if (this.cache.has(key)) {
      this.cache.delete(key);
    } else if (this.cache.size >= this.capacity) {
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }
    this.cache.set(key, value);
  }
}
```

**Outcome**: Memory usage bounded. O(1) get/set operations. Critical conversations preserved in database. Scales to 10k+ chats.

## Error Recovery in Message Processing

**Problem**: Message processing can fail at multiple points (database, LLM, WhatsApp). Need to ensure messages aren't lost.

**Solution**: Save incoming messages immediately. Process asynchronously. Retry failed operations with exponential backoff.

```javascript
async handleMessages({ messages }) {
  for (const msg of messages) {
    try {
      // Save immediately
      await this.saveMessageToDatabase(msg);
      
      // Process asynchronously
      this.processMessageAsync(msg).catch(error => {
        logger.error('Message processing failed:', error);
        // Retry logic
        this.retryMessageProcessing(msg, error);
      });
    } catch (error) {
      logger.error('Failed to save message:', error);
      // Queue for later processing
      await this.queueMessageForRetry(msg);
    }
  }
}
```

**Outcome**: No message loss. Failed messages retried automatically.

## CORS and Authentication in Production

**Problem**: CORS misconfiguration can block legitimate requests. JWT tokens need proper validation.

**Solution**: Environment-based CORS configuration. Strict origin validation in production. JWT validation with proper error handling.

```javascript
await fastify.register(cors, {
  origin: process.env.NODE_ENV === 'production'
    ? [process.env.FRONTEND_URL].filter(Boolean)
    : ['http://localhost:3000', 'http://localhost:3001'],
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS', 'PATCH']
});
```

**Outcome**: Secure production configuration. Development flexibility maintained.

## Database Connection Pooling

**Problem**: Prisma connection pool can exhaust under high load. Need proper connection management.

**Solution**: Configure Prisma connection pool size. Implement connection health checks. Graceful degradation on connection failures.

```javascript
const prisma = new PrismaClient({
  datasources: {
    db: {
      url: process.env.DATABASE_URL
    }
  },
  log: process.env.NODE_ENV === 'development' ? ['query', 'error'] : ['error']
});

// Health check
async function checkDatabaseHealth() {
  try {
    await prisma.$queryRaw`SELECT 1`;
    return true;
  } catch (error) {
    logger.error('Database health check failed:', error);
    return false;
  }
}
```

**Outcome**: Stable database connections. Proper error handling for connection issues.

