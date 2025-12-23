# Fintech CRM Platform

CRM platform for real estate and vehicle financing with AI-powered lead qualification via WhatsApp. Automates initial contact, data collection, and routes qualified leads to consultants using round-robin distribution.

## What This System Does

The platform integrates with WhatsApp to provide 24/7 automated lead qualification through an AI agent. When a lead contacts via WhatsApp, the AI agent engages in conversation, collects necessary information, and classifies the lead. Qualified leads are automatically distributed to consultants using a round-robin algorithm. Consultants can manage leads through a real-time Kanban board and communicate directly via integrated WhatsApp chat.

## How to Read This Case Study

Start with **overview.md** to understand the problem domain, solution approach, and key features. **architecture.md** explains the system components, dual WhatsApp session architecture, and how data flows between the AI agent, consultants, and the database.

**key-flows.md** describes critical flows: lead qualification via WhatsApp, round-robin distribution, real-time Kanban synchronization, and consultant chat interactions. **technical-decisions.md** covers major choices: dual Baileys sessions, LLM selection, round-robin algorithm, WebSocket architecture.

**challenges.md** documents complex problems: webhook timeout prevention, conversation state management, LLM response latency, message deduplication, and human handoff scenarios. **representative-snippets.md** shows code examples for webhook handling, orchestrator logic, intent classification, and state management.

**results.md** presents production outcomes, metrics observed, and measurement guidance.

## Documentation Structure

- **overview.md** – System description, problem statement, solution approach, technology stack, and implemented vs planned features
- **architecture.md** – Component boundaries, dual WhatsApp session architecture, data flow, failure handling, and observability
- **key-flows.md** – Core flows: WhatsApp webhook reception, lead qualification, round-robin distribution, Kanban sync, consultant chat
- **technical-decisions.md** – Major decisions: monolithic architecture, BullMQ for async processing, dual-mode architecture, Redis for state, LLM selection
- **challenges.md** – Complex problems: webhook timeouts, conversation state, LLM latency, message deduplication, human handoff, secure protocol generation
- **representative-snippets.md** – Code examples for webhook handling, orchestrator, intent classification, CPF validation, Redis state management
- **results.md** – Production outcomes, metrics observed, and measurement guidance

## Key Technical Highlights

- **Dual Baileys WhatsApp sessions** – Separate sessions for consultants and AI agent to prevent conflicts
- **AI-powered lead qualification** – Groq LLM for intent classification with fallback rules
- **Round-robin distribution** – Fair lead allocation algorithm with consultant availability tracking
- **Real-time Kanban board** – WebSocket synchronization for live updates across clients
- **Conversation state management** – Redis-based state persistence for multi-turn conversations
- **Message deduplication** – Prevents duplicate processing from webhook retries

## Technology Stack

**Backend**: Node.js 20, Fastify, Prisma, PostgreSQL, Baileys, Groq, Socket.io, BullMQ, Redis  
**Frontend**: React 19, TypeScript, Tailwind CSS, React Query  
**Infrastructure**: Supabase, Render, Vercel

