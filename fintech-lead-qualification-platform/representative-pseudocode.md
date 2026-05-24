# Representative Pseudocode

This is intentionally pseudocode. It shows the engineering shape of the solution without exposing production source code.

## Async webhook intake

```ts
// Fastify-style pseudocode
app.post('/webhooks/leads', async (request, reply) => {
  const payload = leadWebhookSchema.parse(request.body)

  await leadQueue.add('qualify-lead', {
    leadId: payload.leadId,
    message: payload.message,
    receivedAt: new Date().toISOString()
  }, {
    jobId: payload.messageId,
    attempts: 3,
    backoff: { type: 'exponential', delay: 1000 }
  })

  return reply.code(202).send({ queued: true })
})
```

## LLM-assisted classification with deterministic fallback

```ts
async function classifyLead(input: LeadProfile): Promise<LeadScore> {
  try {
    return await llmClassifier.classify(input)
  } catch (error) {
    logger.warn({ error }, 'LLM unavailable; using rule-based classifier')
    return ruleBasedClassifier.classify(input)
  }
}
```

## Round-robin assignment

```ts
async function assignQualifiedLead(lead: Lead) {
  const consultants = await consultantRepo.listActiveOrderedByLastAssignment()
  const selected = consultants.find(canReceiveLead)

  await transaction.run(async tx => {
    await leadRepo.assign(tx, lead.id, selected.id)
    await consultantRepo.markAssigned(tx, selected.id, new Date())
    await events.publish(tx, 'lead.assigned', { leadId: lead.id, consultantId: selected.id })
  })
}
```
