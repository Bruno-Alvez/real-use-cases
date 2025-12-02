# Challenges

## Department-Based Permissions

**Problem**: Need granular permissions per department. RLS policies must enforce access control.

**Solution**: Department-based RLS policies. Permission checks in service layer. User context from JWT.

```sql
CREATE POLICY "Department isolation"
ON vouchers FOR SELECT
USING (department_id IN (
    SELECT department_id FROM collaborators WHERE user_id = auth.uid()
));
```

RLS enforces at database level. Service layer validates permissions. Double protection.

## Voucher Code Uniqueness

**Problem**: Voucher codes must be unique. Need collision prevention.

**Solution**: Database unique constraint. Retry logic on collision. UUID-based generation.

```typescript
import { randomBytes } from 'crypto';

async generateUniqueCode(): Promise<string> {
    let attempts = 0;
    while (attempts < 10) {
        const random = randomBytes(4).toString('hex').toUpperCase();
        const codigo = `VOUCHER-${Date.now()}-${random}`;
        const exists = await this.voucherRepository.findByCode(codigo);
        if (!exists) return codigo;
        attempts++;
    }
    throw new AppError('Failed to generate unique code', 500);
}
```

Unique constraint in database. Retry mechanism. Fallback error handling.

## Kanban Real-Time Updates

**Problem**: Multiple users updating Kanban simultaneously. Need real-time synchronization.

**Solution**: Supabase real-time subscriptions. Optimistic updates. Conflict resolution.

```typescript
useEffect(() => {
    const subscription = supabase
        .channel('candidates')
        .on('postgres_changes', { event: 'UPDATE', schema: 'public', table: 'candidates' }, 
            (payload) => {
                updateCandidateInState(payload.new);
            })
        .subscribe();
    
    return () => subscription.unsubscribe();
}, []);
```

Real-time subscriptions. Optimistic UI updates. Server state as source of truth.

## Voucher Expiration

**Problem**: Vouchers expire. Need automatic status updates.

**Solution**: Supabase cron job. Daily batch update. Status field update.

```sql
CREATE OR REPLACE FUNCTION update_expired_vouchers()
RETURNS void AS $$
BEGIN
    UPDATE vouchers
    SET status = 'expirado'
    WHERE status = 'ativo'
    AND data_expiracao < CURRENT_DATE;
END;
$$ LANGUAGE plpgsql;
```

Automated expiration. Daily cron execution. Status field updated.

## Campaign Voucher Generation

**Problem**: Campaigns need bulk voucher generation. Performance concerns.

**Solution**: Batch processing. Chunked inserts. Progress tracking.

```typescript
async generateCampaignVouchers(campaignId: string, campaignData: CampaignData) {
    const vouchers = [];
    for (let i = 0; i < campaignData.quantity; i++) {
        vouchers.push(this.createVoucherData(campaignId, campaignData));
    }
    
    // Insert in chunks of 100
    for (let i = 0; i < vouchers.length; i += 100) {
        const chunk = vouchers.slice(i, i + 100);
        await this.voucherRepository.bulkInsert(chunk);
    }
}
```

Chunked processing. Progress tracking. Error handling per chunk.

