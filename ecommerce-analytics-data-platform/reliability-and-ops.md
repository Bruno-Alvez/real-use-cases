# Reliability and Ops

## Alarm strategy

Twenty-eight CloudWatch alarms cover every production Lambda. The philosophy is that each alarm maps to a specific failure mode with a known remediation, not to generic thresholds that produce noise.

| Alarm type | Metric | Threshold | Evaluation | Action |
|---|---|---|---|---|
| Error rate | Lambda Errors | >0 | 5 minutes | SNS Critical |
| Duration | Lambda Duration | >240s | 1 occurrence | SNS Critical |
| Zero records | Custom metric | =0 | 24 hours | SNS Warning |
| Token refresh failed | Log pattern `BLING_TOKEN_REFRESH_FAILED` | 1 occurrence | Immediate | SNS Critical |
| Circuit breaker | Log pattern `CIRCUIT_BREAKER_OPEN` | 1 occurrence | Immediate | SNS Critical |

The error rate alarm is set to >0 rather than a percentage because all ETL functions have retry logic built in. If an error reaches the alarm, it has already survived three retry attempts with exponential backoff. That is not normal.

The zero-records alarm has a 24-hour evaluation window rather than 5 minutes because it is expected that a sync function might produce zero records in a given run (no data changed since the last cursor). The alarm fires when no sync has produced records for an entire day, which indicates either an API outage or a broken pipeline.

## Incident: WooCommerce Product Bundles data gap

**Date:** First week of migration validation

**Symptom:** After the initial data load, `fato_itens_vendas` contained 29,143 records instead of the expected 56,163. The FK constraint on `item_id` was generating violations and rolling back inserts for any line item where `product_id = 0`.

**Detection:** The migration script reported rollback errors with `ForeignKeyViolation` on the first load. Row count verification immediately showed the 47.7% shortfall.

**Root cause:** The WooCommerce Product Bundles plugin generates line items with `product_id = 0` for bundle container items (the parent entry that groups component products). These are virtual items with no real SKU. The FK constraint correctly rejected them because `0` is not a valid product ID in `dim_produtos`.

Two options were considered: loosen the FK constraint to allow `product_id = 0`, or parse the bundle metadata and reconstruct the actual component relationships. Loosening the constraint would have preserved a data modeling mistake. The bundle metadata contains the correct component product IDs; it just requires parsing the `_bundled_items` JSON field from order metadata.

**Resolution:** Fetched all 18,941 orders from WooCommerce API. For each order with bundle items, parsed the `_bundled_items` metadata field to extract component product IDs and quantities. Loaded 31,138 corrected records via UPSERT. Recalculated `cluster_base_rfv` because bundle purchasers had incorrect `produto_favorito` values.

**Time to resolve:** 4 hours from detection to corrected data in production.

**Lesson:** Integration-tested the FK constraints against real production data from the source API before the migration cutover. This incident would have been discovered post-cutover if the validation had been skipped.

## Incident: Bot attack records in customer table

**Date:** Identified during audit, before migration started

**Symptom:** The legacy `customers` table contained 91,608 records but the WooCommerce store had no record of approximately two-thirds of them. CPF analysis showed patterns consistent with automated registration: sequential numbers, invalid checksums, duplicate contact information.

**Root cause:** The store had been the target of three waves of bot registrations over its lifetime. The WooCommerce platform did not block them because no CAPTCHA or email verification was configured. The pipeline inserted them without validation because there was no quality gate.

**Resolution:** Cross-referenced the customer table against the WooCommerce API, which only returns customers that have placed at least one order or completed registration. 62,466 records with no corresponding WooCommerce activity were classified as fraudulent and excluded from the migration. 12,609 legitimate customers were migrated.

**Impact on downstream data:** RFV scores and segmentation in the legacy system were computed on a customer base that was 68% fraudulent. All segment sizes, acquisition metrics, and engagement ratios reported to the business prior to the migration were incorrect by approximately 200%.

**Lesson:** Any data pipeline that inserts records from an external source without a validation gate against the authoritative source of truth will accumulate garbage silently. Foreign keys enforce referential integrity but not semantic validity.

## Operational practices

**Deployment gate:** All infrastructure changes go through `terraform plan` review on PR and require manual approval before apply. No change reaches production without a written record.

**Idempotency guarantee:** All ETL writes use UPSERT. Any Lambda can be triggered manually at any time without risk of duplicate records.

**Credential rotation:** Bling OAuth2 tokens are refreshed automatically by the Lambda layer on expiry. PostgreSQL and WooCommerce credentials are static but stored in Secrets Manager with IAM-scoped access. Rotation requires a Secrets Manager update; no Lambda redeployment needed.

**On-call:** Alerts route to email and Slack simultaneously. For critical alarms (error rate, circuit breaker, token refresh), the expectation is acknowledgement within 30 minutes during business hours and within 2 hours outside of it. Most pipeline failures are self-healing on the next scheduled run.
