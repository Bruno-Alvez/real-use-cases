# Key Flows

## Voucher Creation Flow

Request → validation → code generation → creation:

```typescript
async createVoucher(voucherData: CreateVoucherData, userId: string) {
    this.validateVoucherData(voucherData);
    const codigo = await this.generateUniqueCode();
    const dataExpiracao = this.calculateExpirationDate(voucherData.validade_dias);
    
    return await this.voucherRepository.create({
        ...voucherData,
        codigo,
        data_expiracao,
        criado_por: userId,
    });
}
```

Flow: Request → service validation → unique code generation → expiration calculation → repository → response.

## Enrollment Status Update Flow

Kanban drag → status update → database:

```typescript
const handleDragEnd = (result: DropResult) => {
    if (!result.destination) return;
    
    const candidateId = result.draggableId;
    const newStatus = result.destination.droppableId;
    
    await updateCandidateStatus(candidateId, newStatus);
    await refreshCandidates();
};
```

Drag & drop triggers status update. Database updated. UI refreshed.

## Permission Check Flow

Request → user context → permission validation → access:

```typescript
const hasPermission = (user: User, permission: string): boolean => {
    if (!user.department) return false;
    return user.department.permissions.includes(permission);
};

if (!hasPermission(user, PERMISSIONS.MANAGE_VOUCHERS)) {
    throw new AppError('Unauthorized', 403);
}
```

User context extracted from JWT. Permission checked against department. Access granted or denied.

## Voucher Filtering Flow

Filters → repository query → results:

```typescript
async findAll(filters?: VoucherFilters): Promise<Voucher[]> {
    let query = supabase.from('vouchers').select('*');
    
    if (filters?.cpf) query = query.ilike('cpf_aluno', `%${filters.cpf}%`);
    if (filters?.status) query = query.eq('status', filters.status);
    if (filters?.dataInicio) query = query.gte('data_criacao', filters.dataInicio);
    if (filters?.search) {
        query = query.or(
            `codigo.ilike.%${filters.search}%,cpf_aluno.ilike.%${filters.search}%`
        );
    }
    
    const { data } = await query;
    return this.mapToVoucherArray(data || []);
}
```

Dynamic query building. Multiple filter support. Search across fields.

## Campaign Creation Flow

Campaign data → validation → creation → voucher generation:

```typescript
async createCampaign(campaignData: CampaignData) {
    this.validateCampaignData(campaignData);
    
    const campaign = await this.campaignRepository.create(campaignData);
    
    if (campaignData.auto_generate_vouchers) {
        await this.generateCampaignVouchers(campaign.id, campaignData);
    }
    
    return campaign;
}
```

Campaign created. Optional automatic voucher generation. Batch processing.

