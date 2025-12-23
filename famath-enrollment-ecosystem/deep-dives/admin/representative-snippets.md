# Representative Code Snippets

## 1. Code Generation with Retry Logic

**File:** `backend/src/services/VoucherService.ts`

```typescript
private async generateUniqueCode(): Promise<string> {
  let attempts = 0;
  const maxAttempts = 10;

  while (attempts < maxAttempts) {
    const timestamp = Date.now().toString().slice(-6);
    const code = `DESC${new Date().getFullYear()}${timestamp}`;
    
    const exists = await this.voucherRepository.codeExists(code);
    if (!exists) {
      return code;
    }
    
    attempts++;
    await new Promise(resolve => setTimeout(resolve, 100));
  }

  throw new AppError('Failed to generate unique voucher code', 500);
}
```

**Context:** Generates human-readable voucher codes with retry logic. Fixed 100ms delay between attempts, no exponential backoff.

## 2. Business Rule Validation in Service Layer

**File:** `backend/src/services/CampaignService.ts`

```typescript
if (campaignUsage.length > 0) {
  logger.warn(`Campaign ${id} has ${campaignUsage.length} usage(s) registered`);
  
  if (updateData.codigo_promocional && 
      updateData.codigo_promocional !== existingCampaign.codigoPromocional) {
    throw new AppError(
      'Cannot change promotional code of a campaign that has been used', 
      400
    );
  }
  
  if (updateData.limite_total !== undefined && 
      updateData.limite_total < campaignUsage.length) {
    throw new AppError(
      `Cannot reduce total limit below ${campaignUsage.length}`, 
      400
    );
  }
}
```

**Context:** Prevents invalid state changes by querying campaign_usage table before allowing updates.

## 3. Database Query with Error Handling

**File:** `backend/src/repositories/VoucherRepository.ts`

```typescript
async findById(id: string): Promise<Voucher | null> {
  try {
    const { data, error } = await supabase
      .from('vouchers')
      .select('*')
      .eq('id', id)
      .single();

    if (error) {
      if (error.code === 'PGRST116') {
        return null;
      }
      logger.error('Error fetching voucher:', error);
      throw new Error(`Error fetching voucher: ${error.message}`);
    }

    return data ? this.mapToVoucher(data) : null;
  } catch (error) {
    logger.error('Error in VoucherRepository.findById:', error);
    throw error;
  }
}
```

**Context:** Handles Supabase-specific error codes (PGRST116 = not found) and maps database records to domain objects.

## 4. Multi-Step Update Without Transaction

**File:** `backend/src/repositories/CampaignRepository.ts`

```typescript
if (courseIds !== undefined) {
  await supabase
    .from('campaign_courses')
    .delete()
    .eq('campaign_id', id);

  if (courseIds.length > 0) {
    await this.associateCourses(id, courseIds);
  }
}
```

**Context:** Separate delete and insert operations. No transaction wrapper, so partial failures leave inconsistent state.

## 5. Database Trigger for Automatic Status Update

**File:** `database/supabase/supabase_schema_vouchers_campaigns_final.sql`

```sql
CREATE OR REPLACE FUNCTION mark_voucher_as_used()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE vouchers 
    SET status = 'utilizado', updated_at = NOW()
    WHERE id = NEW.voucher_id;
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER trigger_mark_voucher_as_used
    AFTER INSERT ON voucher_usage
    FOR EACH ROW EXECUTE FUNCTION mark_voucher_as_used();
```

**Context:** Automatically updates voucher status when usage record is created. Runs at database level, ensuring consistency.

## 6. Rate Limiting Middleware

**File:** `backend/src/middlewares/rateLimiter.ts`

```typescript
export const apiLimiter = rateLimit({
  windowMs: config.rateLimit.windowMs,
  max: config.rateLimit.maxRequests,
  message: {
    success: false,
    error: 'Too many requests. Try again later.',
    statusCode: 429
  },
  standardHeaders: true,
  legacyHeaders: false,
});
```

**Context:** Global rate limit using in-memory store. Applied to all API routes in `app.ts`.

## 7. Error Handler with Structured Logging

**File:** `backend/src/middlewares/errorHandler.ts`

```typescript
export const errorHandler = (
  err: Error | AppError,
  req: Request,
  res: Response,
  next: NextFunction
) => {
  if (err instanceof AppError) {
    logger.error('Operational Error:', {
      message: err.message,
      statusCode: err.statusCode,
      stack: err.stack,
      path: req.path,
      method: req.method
    });

    return res.status(err.statusCode).json({
      success: false,
      error: err.message,
      statusCode: err.statusCode
    });
  }

  logger.error('Unexpected Error:', {
    message: err.message,
    stack: err.stack,
    path: req.path,
    method: req.method
  });

  return res.status(500).json({
    success: false,
    error: 'Internal server error',
    statusCode: 500
  });
};
```

**Context:** Distinguishes between operational errors (AppError) and unexpected errors, logging both with request context.

## 8. Frontend State Management with useRef Guard

**File:** `frontend/src/components/Voucher/CampaignForm.tsx`

```typescript
const initializationRef = useRef<string | null>(null);

useEffect(() => {
  if (!campaign?.id || courses.length === 0) return;
  if (initializationRef.current === campaign.id) return;
  
  initializationRef.current = campaign.id;
  
  // Initialize form state...
}, [campaign?.id, courses.length]);
```

**Context:** Prevents infinite re-initialization loops by tracking which campaign ID has been initialized.
