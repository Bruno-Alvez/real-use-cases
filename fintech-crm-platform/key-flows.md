# Key Flows

## Lead Qualification Flow (AI Agent)

WhatsApp message → conversation lookup → LLM response → state transition:

```typescript
async processMessage(phone: string, text: string): Promise<AgentResponse> {
  const conversation = await this.getOrCreateConversation(phone);
  await this.persistMessage(conversation.id, text, 'CLIENT');

  const [response] = await Promise.all([
    this.generateResponse(conversation, text),
    this.simulateTyping(phone, text.length)
  ]);

  if (response.message) {
    await Promise.all([
      this.persistMessage(conversation.id, response.message, 'AI_AGENT'),
      this.sendMessage(phone, response.message)
    ]);
  }

  await this.transitionStage(conversation.id, response.stage, response.data);
  return response;
}
```

Flow: Message received → saved → typing indicator → LLM call → response sent → state updated → transfer if qualified.

## Lead Creation and Assignment Flow

Webhook → validation → AI classification → round-robin assignment → notification:

```typescript
async processWebhookLead(data: WebhookPayload, source: LeadSource): Promise<ProcessedLead> {
  const leadData = this.validateAndStructureData(data, source);
  const consultant = await this.selectConsultant(data.creatorId);
  
  const [lead, analysis] = await Promise.all([
    this.createLead(leadData, source),
    this.aiService.classifyLead(leadData).catch(() => this.fallbackClassification(leadData))
  ]);

  await Promise.all([
    this.assignLeadToConsultant(lead.id, consultant.id),
    this.updateLeadAIClassification(lead.id, analysis),
    this.notifyConsultant(lead, consultant, analysis)
  ]);

  websocketService.broadcast('kanban:lead_created', { lead });
  return { lead, consultantId: consultant.id };
}
```

Parallel execution for classification and lead creation. WebSocket broadcast for real-time UI updates.

## Round-Robin Assignment Flow

Circular array selection with atomic index updates:

```typescript
assignLead(leadData: LeadData, analysis: LeadAnalysis): Assignment {
  if (this.consultants.length === 0) {
    throw new NoConsultantsAvailableError();
  }

  const consultant = this.consultants[this.currentIndex];
  this.currentIndex = (this.currentIndex + 1) % this.consultants.length;
  this.stats.increment(consultant.id);

  return {
    consultant,
    reason: `Round-robin (index ${this.currentIndex})`,
    estimatedResponseTime: this.calculateResponseTime(consultant)
  };
}
```

O(1) selection. Stats tracked for future load-based improvements.

## WhatsApp Message Handling (Consultant Session)

Batch processing with indexed updates and real-time broadcast:

```typescript
async handleMessages({ messages }: MessageUpsert): Promise<void> {
  const incoming = messages.filter(msg => !msg.key.fromMe && msg.message);
  
  await Promise.all(incoming.map(msg => this.processIncomingMessage(msg)));
}

private async processIncomingMessage(msg: proto.IWebMessageInfo): Promise<void> {
  const jid = msg.key.remoteJid!;
  const text = extractText(msg.message);
  
  this.updateChatIndex(jid, text);
  
  const lead = await this.findLeadByPhone(jid);
  if (lead) {
    await this.conversationService.saveMessage(lead.id, jid, text, 'in');
  }

  this.websocketService?.broadcast('whatsapp:message', {
    chatId: jid,
    message: text,
    timestamp: new Date()
  });
}
```

In-memory index for fast chat list updates. Database persistence only for leads.

## Kanban Board Update Flow

Status update → database → WebSocket broadcast → UI sync:

```typescript
async updateLeadStatus(id: string, status: LeadStatus): Promise<Lead> {
  const lead = await this.prisma.lead.update({
    where: { id },
    data: { status, updated_at: new Date() }
  });

  this.websocketService.broadcast('kanban:lead_updated', {
    lead,
    action: 'status_changed'
  });

  return lead;
}
```

Frontend uses React Query with optimistic updates. WebSocket ensures all clients sync.

## Conversation Transfer Flow (AI Agent → Consultant)

Qualification complete → lead creation → assignment → handoff message:

```typescript
async transferToConsultant(conversationId: string): Promise<Lead> {
  const conversation = await this.prisma.conversation.findUnique({
    where: { id: conversationId },
    include: { messages: true }
  });

  const lead = await this.leadService.createLeadFromConversation(conversation);
  const assignment = this.roundRobinService.assignLead({}, {});
  
  await Promise.all([
    this.leadService.assignLeadToConsultant(lead.id, assignment.consultant.id),
    this.prisma.conversation.update({
      where: { id: conversationId },
      data: {
        status: 'TRANSFERRED',
        consultantId: assignment.consultant.id,
        leadId: lead.id,
        transferredAt: new Date()
      }
    }),
    this.sendHandoffMessage(conversation.phoneNumber, assignment.consultant.id),
    this.notifyConsultant(lead, assignment.consultant)
  ]);

  return lead;
}
```

Full conversation history preserved. Consultant receives notification with context.

