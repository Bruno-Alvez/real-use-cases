# Fintech CRM Platform - Overview

CRM for real estate and vehicle financing with AI-powered lead qualification via WhatsApp. Automates initial contact, data collection, and routes qualified leads to consultants.

## Problem

Manual lead processing caused delays and lost opportunities. Web forms generated cold leads with incomplete data. Consultants spent time on unqualified prospects.

## Solution

Three-part system: AI agent qualifies leads via WhatsApp using Groq LLM, dual Baileys sessions handle consultant and AI conversations separately, round-robin distributes qualified leads automatically.

## Key Features

- AI qualification via WhatsApp (24/7)
- Round-robin lead distribution
- Real-time Kanban board with WebSocket sync
- Integrated WhatsApp chat for consultants
- Analytics dashboard per consultant
- Role-based permissions

## Stack

**Backend**: Node.js 20, Fastify, Prisma, PostgreSQL, Baileys, Groq, Socket.io  
**Frontend**: React 19, TypeScript, Tailwind, React Query  
**Infra**: Supabase, Render, Vercel

## Implemented vs Planned

**Implemented**
- AI-powered lead qualification via WhatsApp
- Dual Baileys WhatsApp sessions (consultant and AI agent)
- Round-robin lead distribution algorithm
- Real-time Kanban board with WebSocket synchronization
- Integrated WhatsApp chat interface for consultants
- Analytics dashboard per consultant
- Role-based access control
- Lead classification with LLM (Groq) and fallback rules
- Conversation state management and persistence
- Message deduplication and error recovery

**Planned**
- Advanced analytics and reporting
- Multi-language support for AI agent
- Integration with additional communication channels
- Automated follow-up campaigns
- Lead scoring refinement based on conversion data

