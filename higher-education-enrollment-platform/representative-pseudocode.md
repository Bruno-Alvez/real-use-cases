# Representative Pseudocode

This is intentionally pseudocode. It shows the engineering shape of the solution without exposing production source code.

## Backend boundary for sensitive actions

```ts
// NestJS-style pseudocode
@Patch('/backoffice/enrollments/:id/move')
@UseGuards(JwtAuthGuard, PermissionsGuard)
@RequiredPermission('enrollment:update')
async moveEnrollmentStep(@CurrentUser() user, @Param('id') id, @Body() input) {
  const enrollment = await enrollmentRepo.findById(id)

  domain.assertTransitionAllowed(enrollment.status, input.nextStep)
  domain.assertUserCanOperate(user, enrollment)

  await tx.run(async trx => {
    await enrollmentRepo.updateStep(trx, id, input.nextStep)
    await auditLogRepo.append(trx, {
      actorId: user.id,
      action: 'enrollment.step_moved',
      entityId: id,
      metadata: { from: enrollment.step, to: input.nextStep }
    })
  })

  return { ok: true }
}
```

## Server-side pricing

```ts
// TypeScript pseudocode
function calculateEnrollmentPrice(input: PricingInput): PricingResult {
  const base = pricingTable.get(input.courseId, input.shift)
  const campaign = campaignService.findActiveFor(input.courseId, input.shift, input.now)
  const voucher = voucherService.findValidForCpf(input.cpf, input.voucherCode)

  const price = priceEngine.apply({ base, campaign, voucher })

  return {
    nominalValue: base.nominalValue,
    finalValue: price.finalValue,
    appliedRules: price.rules,
    expiresAt: price.deadline
  }
}
```

## Migration-safe production rollout

```text
legacy system remains available
new system receives controlled traffic
users validate operational flows
new enrollments move to new backend
legacy data remains readable during transition
cutover happens only after validation window
```
