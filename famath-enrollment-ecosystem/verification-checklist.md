# Verification Checklist

## Architecture

- [ ] Shared Supabase database schema documented (`database/schema.sql`, `database/supabase/supabase_schema_vouchers_campaigns_final.sql`)
- [ ] RLS policies defined for admin access control (`database/supabase/supabase_schema_collaborators.sql`)
- [ ] Database triggers for automatic status updates (`database/triggers/auto_update_enrollment_status.sql`)
- [ ] Component boundaries clearly defined (Admin, Portal, Agent)
- [ ] Legacy SQL Server sync boundary documented (Planned: not implemented)

## Integrations

- [ ] Asaas API integration implemented (`backend/src/services/checkoutService.ts`)
- [ ] Z-API webhook endpoint implemented (`src/index.ts`)
- [ ] OpenAI integration for conversational agent (`src/services/openai.ts`)
- [ ] ViaCEP integration with retry (`src/tools/viacep.ts`)
- [ ] Legacy SQL Server sync mentioned (Planned: not implemented)

## Reliability

- [ ] Timeouts configured for external API calls (Agent: Z-API 20s, ViaCEP 10s, OpenAI 30s)
- [ ] Retry logic implemented (Agent: BullMQ 2 attempts, 5s backoff; ViaCEP: 2 attempts, 1s/2s backoff)
- [ ] Error handling and logging implemented (Agent: Pino; Portal: Morgan; Admin: Winston)
- [ ] Graceful shutdown implemented (Portal: 10s timeout)
- [ ] Portal: No retry logic for Asaas API calls (Known gap)
- [ ] Portal: No idempotency keys for checkout creation (Known gap)
- [ ] Agent: No webhook signature verification (Known gap)
- [ ] Agent: No dead-letter queue for failed jobs (Known gap)

## Security

- [ ] Password hashing implemented (PBKDF2, 100k iterations)
- [ ] Log sanitization implemented (Agent: CPF, phone, email masked)
- [ ] RLS policies enforce access control
- [ ] Service role key usage documented (Agent bypasses RLS)
- [ ] Portal and Admin: No log sanitization (Known gap)
- [ ] Agent: No webhook signature verification (Known gap)

## Evidence Files

- **Admin**: `deep-dives/admin/architecture.md`, `backend/src/services/VoucherService.ts`, `backend/src/services/CampaignService.ts`, `database/supabase/supabase_schema_collaborators.sql`
- **Portal**: `deep-dives/portal/architecture.md`, `backend/src/services/checkoutService.ts`, `backend/src/services/pricingService.ts`, `database/triggers/auto_update_enrollment_status.sql`
- **Agent**: `deep-dives/agent/architecture.md`, `src/agents/conversationalAgent.ts`, `src/queue.ts`, `src/services/openai.ts`, `src/tools/supabase.ts`
