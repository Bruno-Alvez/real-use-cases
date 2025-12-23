# Representative Snippets

## Snippet 1: Transaction Deduplication Check

**File:** `backend/app/services/recurring_service.py`

**Lines:** 36-84

```python
def _generate_transaction_key(
    self,
    date_str: str,
    amount: float,
    description: str,
    category_id: str,
    account_id: str
) -> str:
    """Generate unique key for transaction identification"""
    return f"{date_str}-{amount}-{description.strip().lower()}-{category_id}-{account_id}"

async def _transaction_exists(
    self,
    date_str: str,
    amount: float,
    description: str,
    category_id: str,
    account_id: str,
    processed_cache: Set[str]
) -> bool:
    """Check if transaction already exists"""
    key = self._generate_transaction_key(date_str, amount, description, category_id, account_id)
    
    # Check in cache first
    if key in processed_cache:
        return True
    
    # Check in database
    transactions = await self.transaction_repository.get_all(
        filters={
            "date": date_str,
            "category_id": category_id,
            "account_id": account_id
        },
        limit=100
    )
    
    for t in transactions:
        existing_key = self._generate_transaction_key(
            t.date,
            t.amount,
            t.description or "",
            t.category_id,
            t.account_id
        )
        if existing_key == key:
            return True
    
    return False
```

**Purpose:** Prevents duplicate transactions when processing recurring rules. Uses in-memory cache for same-request duplicates, falls back to database query.

## Snippet 2: Plan Limit Enforcement

**File:** `backend/app/services/plan_limits_service.py`

**Lines:** 102-122

```python
async def ensure_can_import_csv(self) -> None:
    """
    Valida se a company ainda pode fazer importação CSV neste mês.
    """
    features = await self._get_active_plan_features()
    max_imports = int(features.get("csv_imports_per_month", 2))

    # -1 = ilimitado
    if max_imports < 0:
        return

    now = datetime.utcnow()
    used = await self.csv_imports_repo.count_imports_in_month(
        year=now.year,
        month=now.month,
    )
    if used >= max_imports:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="CSV import limit reached for current plan this month",
        )
```

**Purpose:** Enforces monthly CSV import limits based on subscription plan. Checks current month usage before allowing import.

## Snippet 3: JWT Token Validation

**File:** `backend/app/core/security.py`

**Lines:** 16-94

```python
async def verify_token(
    credentials: HTTPAuthorizationCredentials = Depends(security)
) -> dict:
    """
    Verify JWT token from Supabase
    """
    import logging
    logger = logging.getLogger(__name__)
    
    token = credentials.credentials
    logger.info(f"🔐 Verifying token (length: {len(token)})")
    
    try:
        # Decode JWT token without verifying assinatura/audience
        from jose.utils import base64url_decode
        import json

        # JWT = header.payload.signature -> pegamos só o payload (segunda parte)
        parts = token.split(".")
        if len(parts) != 3:
            logger.error(f"❌ Invalid token format: expected 3 parts, got {len(parts)}")
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Invalid token format",
                headers={"WWW-Authenticate": "Bearer"},
            )

        payload_part = parts[1]
        # Ajustar padding do base64
        padded = payload_part + "=" * (-len(payload_part) % 4)
        decoded_bytes = base64url_decode(padded.encode("utf-8"))
        payload = json.loads(decoded_bytes.decode("utf-8"))
        
        # Extract user_id from token (Supabase uses 'sub' field)
        user_id = payload.get("sub")
        if not user_id:
            logger.error(f"❌ Token missing user ID. Payload: {payload}")
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Token missing user ID",
                headers={"WWW-Authenticate": "Bearer"},
            )
        
        # Check expiration (comparing UTC timestamps)
        exp = payload.get("exp")
        if exp:
            exp_datetime = datetime.fromtimestamp(exp, tz=timezone.utc)
            now = datetime.now(tz=timezone.utc)
            
            if exp_datetime < now:
                logger.error(f"❌ Token expired. Exp: {exp_datetime}, Now: {now}")
                raise HTTPException(
                    status_code=status.HTTP_401_UNAUTHORIZED,
                    detail="Token expired",
                    headers={"WWW-Authenticate": "Bearer"},
                )
        
        return {
            "sub": user_id,
            "email": payload.get("email", ""),
            "payload": payload
        }
```

**Purpose:** Validates JWT token structure and expiration. Extracts user_id from token payload. Note: Does not verify signature.

## Snippet 4: Company ID Resolution

**File:** `backend/app/core/security.py`

**Lines:** 143-206

```python
async def get_current_company_id(user_id: str = Depends(get_current_user_id)) -> str:
    """
    Get company_id for the current user
    
    Looks up the user's company via company_users table.
    For MVP, assumes 1 user = 1 company (owner role).
    Later can be extended for multi-company support.
    """
    import logging
    logger = logging.getLogger(__name__)
    
    logger.info(f"🏢 Getting company for user: {user_id}")
    
    client = get_supabase_client()
    
    try:
        # Get company_id from company_users table
        response = (
            client.table("company_users")
            .select("company_id")
            .eq("user_id", user_id)
            .limit(1)
            .execute()
        )
        
        if not response.data or len(response.data) == 0:
            logger.error(f"❌ User {user_id} has no associated company")
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail="User has no associated company. Please create a company first.",
            )
        
        company_id = response.data[0].get("company_id")
        if not company_id:
            logger.error(f"❌ Company ID not found in response for user {user_id}")
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail="Company ID not found for user",
            )
        
        logger.info(f"✅ Company ID: {company_id}")
        return company_id
```

**Purpose:** Resolves company_id from user_id via database query. Used as dependency in all authenticated endpoints for multi-tenant filtering.

## Snippet 5: CSV Import Row Processing

**File:** `backend/app/services/csv_import_service.py`

**Lines:** 80-157

```python
for idx, raw_row in enumerate(reader, start=2):  # linha 1 = header
    errors: List[str] = []

    date_str = (raw_row.get("date") or "").strip()
    type_str = (raw_row.get("type") or "").strip().lower()
    amount_str = (raw_row.get("amount") or "").strip().replace(",", ".")
    category_name = (raw_row.get("category") or "").strip().lower()
    account_name = (raw_row.get("account") or "").strip().lower()
    description = (raw_row.get("description") or "").strip() or None

    # Validar campos básicos
    if not date_str:
        errors.append("Missing date")
    if type_str not in ("income", "expense"):
        errors.append("type must be 'income' or 'expense'")
    try:
        amount = float(amount_str)
        if amount <= 0:
            raise ValueError()
    except Exception:
        errors.append("amount must be a positive number")

    # Converter data
    from datetime import datetime
    try:
        parsed_date = datetime.strptime(date_str, "%d/%m/%Y").date()
    except Exception:
        errors.append("date must be in format DD/MM/YYYY")
        parsed_date = None

    # Resolver categoria e conta
    category = category_by_name.get(category_name)
    account = account_by_name.get(account_name)

    if not category:
        errors.append(f"Category '{raw_row.get('category')}' not found")
    if not account:
        errors.append(f"Account '{raw_row.get('account')}' not found")

    if errors:
        failed_count += 1
        rows.append(
            ImportRow(
                row_number=idx,
                success=False,
                transaction_id=None,
                errors=errors,
            )
        )
        continue

    # Criar transação
    transaction = TransactionCreate(
        date=parsed_date,
        transaction_kind=type_str,
        amount=amount,
        category_id=category.id,
        account_id=account.id,
        description=description,
    )
    
    created = await self.transaction_service.create(transaction)
    success_count += 1
    rows.append(
        ImportRow(
            row_number=idx,
            success=True,
            transaction_id=created.id,
            errors=[],
        )
    )
```

**Purpose:** Processes CSV rows sequentially, validates each row, creates transactions for valid rows, collects errors for invalid rows. Returns partial results.

## Snippet 6: Storage Upload with Content-Type Detection

**File:** `backend/app/services/storage_service.py`

**Lines:** 131-232

```python
async def upload_avatar_from_bytes(self, user_id: str, filename: str, content: bytes) -> str:
    """
    Faz upload de um avatar a partir de bytes já lidos.
    Uses service role client to bypass RLS.
    """
    from datetime import datetime
    import mimetypes
    
    # Generate unique filename with timestamp
    file_ext = filename.split('.')[-1] if '.' in filename else 'jpg'
    timestamp = int(datetime.now().timestamp() * 1000)
    unique_filename = f"{timestamp}.{file_ext}"
    path = self.avatar_path(user_id, unique_filename)
    
    # Determine content type from file magic bytes (first bytes) - MUST be done before upload
    content_type = None
    
    # Check magic bytes to determine actual file type (most reliable method)
    if len(content) >= 4:
        if content[:4] == b'\x89PNG':
            content_type = 'image/png'
        elif content[:2] == b'\xff\xd8':
            content_type = 'image/jpeg'
        elif len(content) >= 12 and content[:4] == b'RIFF' and content[8:12] == b'WEBP':
            content_type = 'image/webp'
        elif len(content) >= 6 and content[:6] in (b'GIF87a', b'GIF89a'):
            content_type = 'image/gif'
    
    # If magic bytes didn't match, try extension-based detection
    if not content_type:
        guessed_type, _ = mimetypes.guess_type(filename)
        if guessed_type and guessed_type.startswith('image/'):
            content_type = guessed_type
    else:
            ext_to_type = {
                'png': 'image/png',
                'jpg': 'image/jpeg',
                'jpeg': 'image/jpeg',
                'webp': 'image/webp',
                'gif': 'image/gif'
            }
            content_type = ext_to_type.get(file_ext.lower(), 'image/jpeg')
    
    # Use Supabase REST API directly to ensure proper content-type handling
    supabase_url = settings.SUPABASE_URL
    service_key = settings.SUPABASE_SERVICE_ROLE_KEY
    
    upload_url = f"{supabase_url}/storage/v1/object/{AVATARS_BUCKET}/{path}"
    
    headers = {
        "Authorization": f"Bearer {service_key}",
        "Content-Type": content_type,
        "x-upsert": "true"
    }
    
    # Upload using httpx to ensure proper content-type header
    with httpx.Client(timeout=30.0) as client:
        response = client.put(
            upload_url,
            content=content,
            headers=headers
        )
        
        if response.status_code not in [200, 201]:
            error_msg = response.text
            logger.error(f"Supabase Storage API error: {response.status_code} - {error_msg}")
            raise Exception(f"Failed to upload avatar: {error_msg}")
    
    return path
```

**Purpose:** Uploads avatar files with content-type detection via magic bytes. Uses direct REST API instead of Python client due to content-type handling issues. Includes 30-second timeout.

## Snippet 7: Repository Query with Company Filter

**File:** `backend/app/repositories/transaction_repository.py`

**Lines:** 27-72

```python
async def get_all(
    self,
    skip: int = 0,
    limit: int = 100,
    filters: Optional[dict] = None
) -> List[TransactionResponse]:
    """
    Get all transactions for the company
    """
    query = self.client.table("transactions").select("*").eq("company_id", self.company_id)
    
    # Apply filters
    if filters:
        if "date_from" in filters:
            query = query.gte("date", filters["date_from"])
        if "date_to" in filters:
            query = query.lte("date", filters["date_to"])
        if "type" in filters:
            query = query.eq("type", filters["type"])
        if "category_id" in filters:
            query = query.eq("category_id", filters["category_id"])
        if "account_id" in filters:
            query = query.eq("account_id", filters["account_id"])
    
    # Order by date (most recent first)
    query = query.order("date", desc=True)
    
    # Apply pagination
    query = query.range(skip, skip + limit - 1)
    
    response = query.execute()
    
    if response.data:
        # Convert 'type' from database to 'transaction_kind' for schema
        transactions = []
        for item in response.data:
            if 'type' in item:
                item['transaction_kind'] = item.pop('type')
            transactions.append(TransactionResponse(**item))
        return transactions
    return []
```

**Purpose:** Repository method that always filters by `company_id` for multi-tenant isolation. Applies additional filters, pagination, and converts database field names to schema field names.

## Snippet 8: Recurring Rule Version Creation

**File:** `backend/app/services/recurring_service.py`

**Lines:** 163-221

```python
else:
    # Create new version for future transactions only
    current_date = datetime.now().date().isoformat()

    # Start from the existing rule data so fields que não foram alterados
    # (categoria, conta, dia do mês, etc.) sejam mantidos na nova versão.
    base_data = existing.model_dump()

    # Remove campos que são controlados pelo sistema/versionamento
    for system_field in [
        "id",
        "created_at",
        "effective_from_date",
        "original_rule_id",
        "superseded_by",
        "processed",
        "last_processed_date",
        "processing_lock",
        "version",
        "is_archived",
    ]:
        base_data.pop(system_field, None)

    # Dados enviados pelo frontend (apenas campos realmente modificados)
    partial_update = rule.model_dump(exclude={"mode"}, exclude_unset=True)

    # Merge: o que veio do frontend sobrescreve o valor atual;
    # o restante permanece igual ao existente.
    update_data = {**base_data, **partial_update}

    # Garantir company_id e user_id corretos
    update_data["company_id"] = self.company_id
    if self.user_id:
        update_data["user_id"] = self.user_id

    # Convert date objects to strings for JSON serialization
    if "start_date" in update_data and isinstance(update_data["start_date"], date):
        update_data["start_date"] = update_data["start_date"].isoformat()
    if "end_date" in update_data and isinstance(update_data["end_date"], date):
        update_data["end_date"] = update_data["end_date"].isoformat()

    new_version = await self.repository.create_rule_version(
        rule_id,
        update_data,
        current_date
    )
    
    # Update future transactions to use new rule
    if new_version:
        transactions = await self.transaction_repository.get_by_recurring_rule_id(rule_id)
        current_date_obj = datetime.now().date()
        for t in transactions:
            if t.date >= current_date_obj:
                from app.schemas.transaction import TransactionUpdate
                await self.transaction_repository.update(
                    t.id,
                    TransactionUpdate(recurring_rule_id=new_version.id)
                )
    
    return new_version
```

**Purpose:** Creates new rule version when updating "future only" transactions. Merges existing rule data with updates, creates new version, and links future transactions to new version.
