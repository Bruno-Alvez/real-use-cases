# Fintech CRM Platform

CRM platform for real estate and vehicle financing that automates lead qualification via WhatsApp, distributes qualified prospects to consultants automatically, and surfaces everything through a real-time Kanban interface.

## Problem

The brokerage operated on a fully manual lead pipeline. Prospects submitted inquiries through web forms, a coordinator reviewed submissions, manually assigned them to consultants via spreadsheet or messaging, and consultants then reached out, often hours later. By that point, many leads had already moved on or contacted a competitor.

The form data itself was the second problem. Fields were optional, submissions were incomplete, and no one was screening for intent or financial readiness before a consultant invested time. Consultants regularly spent their mornings working through cold, low-probability contacts before reaching anyone genuinely interested in financing.

The third problem was coordination overhead. With no shared view of pipeline state, consultants duplicated outreach, managers lacked visibility into who was working what, and distribution was never truly fair because it depended on whoever happened to be watching the spreadsheet at the right moment.

## Solution

The platform replaces the manual layer with three integrated components that hand off to each other automatically.

An AI agent running on WhatsApp engages every inbound lead immediately, regardless of the time of day. It collects the information a consultant would otherwise spend the first call gathering: intent, financing amount, property or vehicle type, income range, and timeline. Using Groq's LLM, it classifies each lead as HOT, WARM, or COLD and attaches a structured profile to the record before any human sees it. Consultants receive leads that are already qualified, already documented, and already warm from a real conversation.

Qualified leads flow into a round-robin scheduler that assigns them to available consultants in O(1) time. The assignment happens in under two seconds on average, without any coordinator involvement. Distribution variance across the team stays under 7% weekly, which matters for morale in commission-driven environments.

Consultants work from a real-time Kanban board that reflects pipeline state across the entire team simultaneously. When a lead moves from qualification to negotiation, every connected client sees the update within 87 milliseconds. An integrated WhatsApp chat panel sits alongside the board, so consultants continue conversations in the same interface without switching tools or losing context.

## Key Features

**AI lead qualification via WhatsApp, 24/7.** The agent handles the full intake conversation: greeting, data collection, qualification questions, and temperature classification. It reaches 83-87% data collection completion depending on lead engagement and classifies at 75-82% accuracy compared to human expert judgment. When the Groq API is unavailable, a rule-based fallback keeps classification running without interruption.

**Dual Baileys WhatsApp sessions.** The AI agent and consultant chat run as completely separate WhatsApp sessions. This prevents state collisions, allows each session to operate under different message handling logic, and means a consultant reconnecting their session does not interfere with ongoing AI conversations, or vice versa.

**Round-robin lead distribution.** Assignment is automatic and immediate. Redis tracks consultant availability and rotation index. A new assignment completes in an average of 1.3 seconds (p95: 2.1s), with no manual step and no coordinator as a bottleneck.

**Real-time Kanban board.** Socket.io broadcasts board mutations to all connected clients. Updates land in an average of 87ms (p95: 142ms). The system held 47 concurrent users at peak without observable degradation.

**Integrated WhatsApp chat for consultants.** Consultants read and reply to WhatsApp conversations directly inside the CRM. Message history is persisted to PostgreSQL for audit and continuity. Delivery rates measured at 98.7-99.4% across the production period.

**Role-based access control and per-consultant analytics.** Managers see team-wide pipeline state and conversion metrics. Consultants see their own queue, conversation history, and performance dashboard. Permissions are enforced at the API layer through Fastify middleware.

**Asynchronous processing via BullMQ.** Webhook payloads are acknowledged immediately and enqueued for processing. AI classification and lead routing happen in the background, keeping the system inside WhatsApp's five-second acknowledgment window regardless of Groq API latency.

## Stack

**Backend**: Node.js 20, Fastify, Prisma, PostgreSQL, Baileys, Groq, Socket.io, BullMQ, Redis
**Frontend**: React 19, TypeScript, Tailwind CSS, React Query
**Infrastructure**: Supabase (database), Render (backend), Vercel (frontend)
