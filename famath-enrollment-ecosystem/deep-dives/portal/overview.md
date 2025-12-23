# Overview

## What the System Does

The FAMATH enrollment system manages student registration for a higher education institution. It handles the complete enrollment lifecycle: student registration, document submission and validation, dynamic pricing calculation based on enrollment windows and campaigns, payment processing through Asaas payment gateway, and enrollment status tracking. The system generates sequential protocol numbers, validates required documents per entry modality, applies voucher discounts, and maintains an audit trail via timeline events.

## Integrations

- **Asaas API**: Payment gateway for checkout sessions (PIX and credit card), customer management, and payment status webhooks
- **Supabase**: PostgreSQL database with real-time capabilities, storage for document uploads, and authentication infrastructure
- **Render**: Backend API hosting and deployment
- **Vercel**: Frontend hosting and deployment

## Stack

**Backend:**
- Node.js 20 with TypeScript
- Express.js for HTTP server
- Supabase JS client for database operations
- Morgan for request logging
- Helmet for security headers
- express-rate-limit for API rate limiting
- Multer for file upload handling

**Frontend:**
- React 18 with TypeScript
- Vite for build tooling
- React Router for navigation
- Axios for HTTP client
- Tailwind CSS for styling
- Supabase JS client for direct database queries

**Database:**
- PostgreSQL (via Supabase)
- UUID extension for primary keys
- JSONB for flexible document metadata
- Database triggers for automatic status updates and protocol generation

**Infrastructure:**
- Docker multi-stage builds for backend
- Render.com for backend deployment
- Vercel for frontend deployment

## Implemented

- Student registration with password hashing (PBKDF2)
- Enrollment creation with protocol number generation (database trigger)
- Document upload and validation workflow
- Dynamic pricing engine (timezone-aware, campaign-based, pre-enrollment windows)
- Voucher system with expiration and usage tracking
- Asaas checkout session integration (PIX and credit card)
- Automatic enrollment status updates via database triggers
- Timeline/audit trail for enrollment events
- Password reset functionality
- Rate limiting and security headers
- Graceful shutdown handling

## Planned / Not Implemented

- Webhook endpoint for Asaas payment confirmations (payment status checked via polling)
- Retry logic with exponential backoff for external API calls
- Idempotency keys for checkout session creation
- Dead-letter queue for failed payment processing
- Structured logging with correlation IDs
- Metrics collection (Prometheus/StatsD)
- Distributed tracing
- Automated testing suite
- Admin dashboard for enrollment management
- Email notifications for status changes
