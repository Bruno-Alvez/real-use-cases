This case study documents a CRM platform built for a fintech operating in real estate and vehicle financing. The core problem was a broken lead pipeline: web forms delivered cold, incomplete data, manual distribution created bottlenecks, and consultants burned hours on prospects that never converted. The solution replaces that entire manual layer with an AI agent that qualifies leads via WhatsApp 24/7, routes them automatically through a round-robin scheduler, and gives consultants a real-time Kanban board and integrated chat interface to close deals without switching tools. In production over six months, initial contact time dropped from two hours to eighteen minutes, conversion increased by 12%, and each consultant now handles 40% more leads without additional headcount.

## Outcomes

| Metric | Before | After |
|---|---|---|
| Initial contact time | ~2 hours | 18 minutes |
| Leads per consultant | baseline | +40% |
| Conversion rate | baseline | +12% |
| Manual processing overhead | baseline | -60% |
| AI qualification accuracy | n/a | 75-82% vs human expert |
| Lead assignment latency | manual | avg 1.3s (p95: 2.1s) |
| Message delivery rate | n/a | 98.7-99.4% |
| WebSocket update latency | n/a | avg 87ms (p95: 142ms) |

## How to Read This Case Study

| File | Contents |
|---|---|
| **overview.md** | Problem context, solution description, key features, and technology stack |
| **architecture.md** | Component boundaries, dual WhatsApp session design, data flow, and failure handling |
| **key-flows.md** | End-to-end flows: lead qualification, round-robin distribution, Kanban sync, consultant chat |
| **technical-decisions.md** | Rationale behind major choices: monolithic architecture, BullMQ, Baileys dual-session, LLM selection |
| **challenges.md** | Hard problems solved: webhook timeouts, conversation state, LLM latency, message deduplication, human handoff |
| **representative-snippets.md** | Annotated code for webhook handling, orchestrator logic, intent classification, and Redis state management |
| **results.md** | Production metrics with confidence ranges and measurement context |

## Key Technical Highlights

- **Dual Baileys sessions** separate the AI agent's conversation flow from consultant sessions entirely, preventing state collisions and allowing each to operate under different message handling rules
- **Groq LLM with rule-based fallback** classifies lead temperature and risk profile in an average of 387ms, falling back to deterministic rules during API degradation to keep qualification running
- **Round-robin distribution** runs in O(1) time with Redis-backed consultant availability state, assigning leads in an average of 1.3 seconds with under 7% weekly variance across the team
- **Socket.io real-time Kanban** pushes board updates to all connected clients in an average of 87ms, sustaining 47 concurrent users observed at peak without degradation
- **Conversation state in Redis** persists multi-turn dialogue context across webhook invocations, handling reconnects and retries transparently with message deduplication built into the ingestion path
- **BullMQ job queues** decouple webhook acknowledgment from AI processing, returning 200 to WhatsApp within the five-second timeout window while classification and routing proceed asynchronously

## Technology Stack

**Backend**: Node.js 20, Fastify, Prisma, PostgreSQL, Baileys, Groq, Socket.io, BullMQ, Redis
**Frontend**: React 19, TypeScript, Tailwind CSS, React Query
**Infrastructure**: Supabase (database), Render (backend), Vercel (frontend)
