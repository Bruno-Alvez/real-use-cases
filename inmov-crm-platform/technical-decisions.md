# Technical Decisions

## Framework Selection: Fastify over Express

Fastify chosen for performance and built-in validation. Benchmark shows 2-3x faster request handling than Express. Schema-based validation reduces boilerplate and catches errors early.

```javascript
fastify.post('/api/leads', {
  schema: {
    body: {
      type: 'object',
      required: ['name', 'phone'],
      properties: {
        name: { type: 'string', minLength: 1 },
        phone: { type: 'string', pattern: '^\\+?[1-9]\\d{1,14}$' }
      }
    }
  }
}, createLead);
```

Built-in JSON schema validation eliminates need for separate validation middleware. Type coercion and sanitization handled automatically.

## Dual WhatsApp Sessions Architecture

Separate Baileys instances for consultants and Lívia agent. Decision driven by:

1. **Isolation**: AI agent failures don't affect consultant sessions
2. **Scalability**: Each session can be scaled independently
3. **Security**: Different phone numbers and authentication states
4. **Maintenance**: Updates to one session don't require restarting the other

```javascript
// Consultant session
const baileysService = BaileysService.getInstance();
await baileysService.initialize(); // Uses ./baileys_sessions/consultant

// Lívia session
const liviaBaileysService = LiviaBaileysService.getInstance();
await liviaBaileysService.initialize(); // Uses ./baileys_sessions/livia
```

Session persistence via `useMultiFileAuthState` ensures connections survive server restarts without re-authentication.

## Prisma ORM Selection

Prisma chosen over raw SQL or TypeORM for:

1. **Type Safety**: Generated TypeScript types from schema
2. **Migration System**: Version-controlled database changes
3. **Developer Experience**: Intuitive query API, excellent tooling
4. **Performance**: Query optimization and connection pooling built-in

```javascript
const lead = await prisma.lead.create({
  data: {
    name: leadData.name,
    phone: leadData.phone,
    consultant: {
      connect: { id: consultantId }
    }
  },
  include: {
    consultant: {
      select: { id: true, name: true, email: true }
    }
  }
});
```

Migration files tracked in git, enabling reproducible database state across environments.

## Groq API for LLM

Groq selected over OpenAI for:

1. **Latency**: Faster inference times (200-500ms vs 1-3s)
2. **Cost**: Lower pricing for high-volume usage
3. **Rate Limits**: More generous limits for concurrent requests
4. **Model Selection**: Access to Llama models with competitive quality

```javascript
const groq = new Groq({ apiKey: process.env.GROQ_API_KEY });

const completion = await groq.chat.completions.create({
  messages: [
    { role: 'system', content: systemPrompt },
    { role: 'user', content: userMessage }
  ],
  model: 'llama-3.1-70b-versatile',
  temperature: 0.7,
  max_tokens: 500
});
```

Fallback to rule-based classification implemented for reliability when API unavailable.

## Round-Robin vs Load-Based Assignment

Simple round-robin chosen over complex load balancing:

1. **Simplicity**: Easier to understand and debug
2. **Fairness**: Equal distribution over time
3. **Predictability**: Consultants know when next lead arrives
4. **Performance**: O(1) selection, no complex calculations

**Trade-off Analysis**:
- **Round-robin**: O(1) complexity, predictable, fair over time. Doesn't account for consultant capacity or response times.
- **Load-based**: Better utilization, accounts for capacity. Requires O(n) queries, complex state management, potential race conditions.

Load-based algorithms considered but rejected due to:
- Additional database queries for workload calculation (N+1 problem)
- Complexity in handling consultant availability (real-time state)
- Edge cases with inactive consultants (stale data)
- Current scale doesn't justify complexity (<50 consultants)

**Future consideration**: Hybrid approach with cached workload metrics, updated asynchronously.

```javascript
selectOptimalConsultant(leadData, analysis) {
  const selectedConsultant = this.consultants[this.currentIndex];
  this.currentIndex = (this.currentIndex + 1) % this.consultants.length;
  return selectedConsultant;
}
```

Stats tracking added for future enhancement without changing core algorithm.

## WebSocket over SSE for Real-time Updates

Socket.io (WebSocket) chosen as primary, SSE as fallback:

1. **Bidirectional**: Client can send events (typing indicators, presence)
2. **Lower Latency**: Full-duplex connection
3. **Room Management**: Built-in room/namespace support for multi-tenancy
4. **Automatic Reconnection**: Handles connection drops gracefully

```javascript
this.io.on('connection', (socket) => {
  socket.on('join_room', (roomId) => {
    socket.join(roomId);
    this.rooms.set(roomId, this.rooms.get(roomId) || new Set());
    this.rooms.get(roomId).add(socket.id);
  });
  
  socket.on('lead_updated', (data) => {
    socket.to(data.roomId).emit('kanban_lead_updated', data);
  });
});
```

SSE endpoint maintained for clients behind restrictive firewalls that block WebSocket.

## In-Memory Chat Index

Baileys chat index maintained in memory rather than database:

1. **Performance**: Instant access without database queries
2. **Real-time Updates**: Chat list updates immediately on message receipt
3. **Baileys Integration**: Matches Baileys' internal data structures

**Trade-off Analysis**: 
- **Pros**: O(1) lookup, no database load, matches Baileys patterns
- **Cons**: Lost on restart, not shared across instances, memory growth
- **Decision**: Acceptable for current scale (<1000 concurrent chats). For scale, would migrate to Redis-backed index with TTL.

Mitigated by:
- Persisting critical conversations to database
- Rebuilding index from database on startup
- LRU eviction for memory management
- Acceptable for non-critical chat list UI

```javascript
this.chatIndex.set(jid, {
  id: jid,
  name: contactName,
  lastMessage: messageText,
  timestamp: new Date().toISOString(),
  unreadCount: 0
});
```

## PostgreSQL over MongoDB

PostgreSQL chosen for:

1. **Relational Data**: Leads, consultants, conversations have clear relationships
2. **ACID Compliance**: Critical for financial data integrity
3. **Prisma Support**: Excellent Prisma integration
4. **Supabase**: Managed PostgreSQL with built-in features (auth, storage, real-time)

JSON fields used where appropriate for flexible data (conversation context, metadata) while maintaining relational structure for core entities.

## React Query for Server State

React Query selected over Redux or Context API for server state:

1. **Caching**: Automatic request deduplication and caching
2. **Background Updates**: Stale-while-revalidate pattern
3. **Optimistic Updates**: Built-in support for optimistic UI updates
4. **DevTools**: Excellent debugging experience

```typescript
const { data: leads, isLoading } = useQuery({
  queryKey: ['leads', filters],
  queryFn: () => apiService.getLeads(filters),
  staleTime: 30000,
  refetchOnWindowFocus: true
});
```

Context API still used for global UI state (auth, theme) where appropriate.

## Session Persistence Strategy

WhatsApp sessions persisted to filesystem rather than database:

1. **Baileys Requirement**: `useMultiFileAuthState` expects filesystem
2. **Performance**: Faster than database for session reads/writes
3. **Compatibility**: Works with Baileys' internal session management

Persistent disk mounted in production (Render) at `/data/baileys_sessions` to survive container restarts.

```javascript
const { state, saveCreds } = await useMultiFileAuthState(
  process.env.WHATSAPP_SESSION_DIR || './baileys_sessions'
);
```

Backup strategy: Periodic sync to cloud storage (S3) for disaster recovery.

## Error Handling Strategy

Centralized error handling with custom error classes:

```javascript
class ValidationError extends Error {
  constructor(message) {
    super(message);
    this.statusCode = 400;
    this.name = 'ValidationError';
  }
}

fastify.setErrorHandler((error, request, reply) => {
  if (error instanceof ValidationError) {
    return reply.status(400).send({ error: error.message });
  }
  
  logger.error('Unhandled error:', error);
  return reply.status(500).send({ error: 'Internal server error' });
});
```

Benefits:
- Consistent error response format
- Proper HTTP status codes
- Error logging centralized
- Client-friendly error messages

## Authentication: JWT over Sessions

JWT tokens chosen for stateless authentication:

1. **Scalability**: No server-side session storage required
2. **Microservices**: Tokens work across multiple services
3. **Mobile Ready**: Easy to implement in mobile apps
4. **Security**: Tokens signed and can include expiration

```javascript
const token = jwt.sign(
  { userId: consultant.id, email: consultant.email, role: consultant.role },
  process.env.JWT_SECRET,
  { expiresIn: '7d' }
);
```

Refresh token pattern not implemented initially - tokens have 7-day expiration. Trade-off: Simpler implementation vs enhanced security. For production at scale, would implement refresh tokens with rotation and revocation.

**Security Considerations**:
- Token stored in httpOnly cookie (planned) to prevent XSS
- Short-lived access tokens (15min) with refresh tokens (7 days) for better security
- Token revocation list in Redis for immediate invalidation

