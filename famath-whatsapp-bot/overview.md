# FAMATH WhatsApp Agent - Overview

WhatsApp conversational agent for student enrollment automation. Handles complete matriculation workflow through natural language conversations, integrating with Supabase database and Z-API for WhatsApp messaging.

## Problem

Manual enrollment process requires human agents. Students need to fill forms, validate data, and receive credentials. Process is time-consuming and error-prone.

## Solution

Conversational WhatsApp agent with two modes: LLM-powered conversational mode for natural interactions, and state machine mode for structured enrollment flows. Integrates with Supabase for data persistence and Z-API for WhatsApp messaging.

## Key Features

- Conversational enrollment flow via WhatsApp
- CPF validation and candidate lookup
- Address validation via ViaCEP
- Course and modality selection
- Enrollment status queries
- Human handoff mechanism
- Protocol and password generation

## Stack

**Backend**: Node.js 20, TypeScript, Fastify, BullMQ  
**Queue**: Redis (BullMQ)  
**Database**: Supabase (PostgreSQL)  
**AI**: OpenAI (intent classification, conversational mode)  
**WhatsApp**: Z-API  
**Logging**: Pino

