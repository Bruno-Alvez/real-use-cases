# Architecture

## System Overview

Monorepo structure with backend API (Fastify) and frontend web application (React). Backend exposes RESTful APIs and WebSocket connections for real-time updates. PostgreSQL database managed through Prisma ORM. Dual WhatsApp integration via Baileys library with isolated sessions for consultants and AI agent.

## Backend Architecture

### Application Server

Fastify application with modular route registration. Core services initialized at startup: Prisma connection, WebSocket server, Baileys sessions (consultant and Lívia), and health check monitoring.

```typescript
const fastify = Fastify({
  logger: logger,
  trustProxy: true,
  bodyLimit: 10 * 1024 * 1024 // 10MB
});

await fastify.register(helmet, {
  contentSecurityPolicy: isProduction ? {
    directives: {
      defaultSrc: ["'self'"],
      connectSrc: ["'self'", process.env.FRONTEND_URL!]
    }
  } : false
});

await fastify.register(rateLimit, {
  max: isProduction ? 200 : 2000,
  timeWindow: 60_000
});
```

### Service Layer

Services follow single responsibility principle with clear boundaries:

- **BaileysService**: Manages WhatsApp connection for consultants, handles message routing, maintains chat index in memory
- **LiviaBaileysService**: Isolated WhatsApp session for AI agent, processes incoming messages and triggers LiviaAgent
- **LiviaAgent**: Core AI agent logic, conversation state management, LLM integration via Groq
- **LeadService**: Lead processing, validation, AI classification, round-robin assignment
- **RoundRobinService**: Consultant selection algorithm with workload balancing
- **WebSocketService**: Real-time event broadcasting for notifications and Kanban updates
- **NotificationService**: Multi-channel notification delivery (WebSocket, SSE, in-app)

### Data Layer

Prisma ORM with PostgreSQL. Schema includes:

- **Consultant**: User accounts with roles and permissions
- **Lead**: Core lead entity with AI classification fields, financial data, tracking metadata
- **Conversation**: Lívia conversation state with stage tracking
- **ConversationMessage**: Message history for AI conversations
- **WhatsAppConversation**: Consultant WhatsApp conversations linked to leads
- **Interaction**: Audit trail of all lead interactions

```javascript
model Lead {
  id               String   @id @default(cuid())
  name             String
  phone            String?
  status           String   @default("NEW")
  ai_score         Int?
  ai_temperature   String? // "HOT" | "WARM" | "COLD"
  consultant_id    String?
  consultant       Consultant? @relation(fields: [consultant_id], references: [id])
  conversations   Conversation[]
  created_at       DateTime @default(now())
}
```

### WhatsApp Integration

Two independent Baileys instances with separate session directories:

1. **Consultant Session**: `baileys_sessions/consultant/` - Handles human consultant conversations
2. **Lívia Session**: `baileys_sessions/livia/` - Handles automated AI agent conversations

Each session maintains its own authentication state, connection status, and message handlers. Sessions persist across server restarts using multi-file auth state.

```typescript
class BaileysService {
  private sock: WASocket | null = null;
  private chatIndex = new Map<string, ChatInfo>();
  private sessionPath: string;

  constructor() {
    this.sessionPath = process.env.WHATSAPP_SESSION_PATH || './baileys_sessions';
  }

  async initialize(): Promise<void> {
    const { state, saveCreds } = await useMultiFileAuthState(this.sessionPath);
    
    this.sock = makeWASocket({
      auth: state,
      connectTimeoutMs: 60_000,
      keepAliveIntervalMs: 30_000
    });

    this.sock.ev.on('creds.update', saveCreds);
    this.sock.ev.on('connection.update', this.handleConnectionUpdate);
    this.sock.ev.on('messages.upsert', this.handleMessages);
  }
}
```

### AI Agent Architecture

LiviaAgent processes messages through a state machine with stages: INTRO → QUALIFICATION → SIMULATION → CONVERSION → TRANSFERRED. Each stage has specific prompts and validation rules.

```typescript
class LiviaAgent {
  async processMessage(phone: string, text: string): Promise<AgentResponse> {
    const conversation = await this.getOrCreateConversation(phone);
    await this.persistMessage(conversation.id, text, 'CLIENT');

    const [response] = await Promise.all([
      this.generateResponse(conversation, text),
      this.simulateTyping(phone, text.length)
    ]);

    if (response.message) {
      await Promise.all([
        this.persistMessage(conversation.id, response.message, 'LIVIA'),
        this.sendMessage(phone, response.message)
      ]);
    }

    await this.transitionStage(conversation.id, response.stage, response.data);
    return response;
  }
}
```

### Real-time Communication

WebSocket server (Socket.io) handles bidirectional communication for:
- Kanban board updates (lead status changes, assignments)
- New lead notifications
- WhatsApp message delivery status
- Consultant presence indicators

SSE endpoint provides unidirectional stream for notification delivery when WebSocket is unavailable.

## Frontend Architecture

React application with TypeScript. Component structure:

- **Pages**: Route-level components (LeadsPage, WhatsAppChatPage, AnalyticsPage)
- **Components**: Reusable UI components (Kanban board, chat interface, notification center)
- **Services**: API client, WebSocket client, notification service
- **Contexts**: AuthContext for user state management
- **Hooks**: Custom hooks for data fetching and real-time subscriptions

State management uses React Query for server state and Context API for global UI state. WebSocket client maintains persistent connection with automatic reconnection.

```typescript
class ApiService {
  private baseURL: string;
  
  async request<T>(endpoint: string, options: RequestInit = {}): Promise<T> {
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
}
```

## Data Flow

### Lead Creation Flow

1. Webhook receives lead data (form submission or WhatsApp message)
2. LeadService validates and structures data
3. AI classification via Groq API (with fallback to rule-based classification)
4. RoundRobinService selects consultant
5. Lead created in database with assignment
6. Notification sent via WebSocket to assigned consultant
7. Kanban board updated in real-time for all connected clients

### WhatsApp Message Flow (Lívia)

1. LiviaBaileysService receives message event
2. Routes to LiviaAgent.processMessage()
3. Agent retrieves or creates conversation record
4. Message saved to database
5. LLM generates response via Groq API
6. Response sent via Baileys
7. Conversation state updated (stage progression, data collection)
8. If qualification complete, lead created and transferred to consultant

### Lead Assignment Flow

1. Qualified lead ready for assignment
2. RoundRobinService.initialize() loads active consultants
3. RoundRobinService.assignLead() selects next consultant in rotation
4. Consultant stats updated (workload, last assigned timestamp)
5. Lead.consultant_id updated in database
6. Notification created and broadcast via WebSocket
7. Consultant receives real-time notification in UI

## Security

- JWT authentication with role-based access control
- CORS restricted to frontend domain in production
- Rate limiting (200 req/min production, 2000 dev)
- Input validation via Joi schemas
- Helmet.js for security headers
- Password hashing with bcrypt
- Environment variable protection (no secrets in code)

## Deployment

Backend deployed on Render with Docker, persistent disk for WhatsApp sessions. Frontend deployed on Vercel with environment variables for API endpoints. Database hosted on Supabase with automated backups. Session persistence ensures WhatsApp connections survive deployments.

**Scalability Considerations**:
- Current: Single backend instance handles ~500 concurrent connections
- Horizontal scaling: Stateless API enables multiple instances. WhatsApp sessions require sticky sessions or shared session storage (Redis)
- Database: Connection pool sized for expected load (25 connections per instance)
- Monitoring: Prometheus metrics track connection pool usage, WebSocket connections, message processing latency

