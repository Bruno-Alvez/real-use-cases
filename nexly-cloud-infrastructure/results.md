# Results

## Cost reduction

The total operational cost dropped from R$ 8,230/month to R$ 415/month, a 95% reduction. The breakdown shows where the savings came from:

| Resource | Before | After | Saving |
|---|---|---|---|
| Compute (EC2 always-on) | R$ 180/month | R$ 15/month (Lambda pay-per-request) | R$ 165/month |
| N8N SaaS (data pipelines) | R$ 450/month | R$ 0 (migrated to Lambda) | R$ 450/month |
| Engineering OpEx | R$ 7,600/month (40h × R$ 200/h for manual deploys and troubleshooting) | R$ 400/month (2h × R$ 200/h) | R$ 7,200/month |
| Observability | R$ 0 (nothing) | R$ 0 (CloudWatch free tier) | R$ 0 |
| **Total** | **R$ 8,230/month** | **R$ 415/month** | **R$ 7,815/month (-95%)** |

The engineering OpEx reduction is the largest line item. Before IaC, every deployment required someone to navigate the AWS console manually, which averaged 4 to 8 hours per client environment setup and an estimated 40 hours per month across routine changes, debugging configuration differences, and troubleshooting silently broken pipelines.

## Deployment time

| Operation | Before | After | Technique |
|---|---|---|---|
| New client environment | 4-8 hours | 15 minutes | Terraform modules, single tfvars change |
| Add a Lambda function | 2-4 hours | 10 minutes | `terraform apply` incremental |
| Rollback a configuration | Impossible | 5 minutes | `git revert` + PR + apply |
| Disaster recovery | 2-4 weeks | 15 minutes | `terraform apply` in target region |

## Observability

Before this project, the observability coverage was zero. There were no alerts, no dashboards, and no structured logs. Pipeline failures were discovered manually when someone noticed that data looked stale.

After:
- 28 CloudWatch alarms covering error rates, execution duration, zero-record conditions, Bling token refresh failures, and circuit breaker events
- MTTD (Mean Time To Detect) dropped from >24 hours to <5 minutes
- MTTR (Mean Time To Recover) dropped from >24 hours to <5 minutes (most issues resolve on the next scheduled run after alert)
- 9 CloudWatch log groups with structured JSON, queryable via CloudWatch Insights

## Governance

| Metric | Before | After |
|---|---|---|
| Audit trail | None | Full Git log (who, when, what, why) |
| Environment consistency | Frequent dev/prod drift | Zero (same code, same state) |
| Concurrent execution safety | Not applicable | DynamoDB state locking prevents corruption |
| Credential management | Environment variables, plaintext in DB | Secrets Manager with IAM-scoped access |

## Production uptime

Nine Lambda functions have been running in production continuously since February 2026. Zero incidents caused by infrastructure configuration. All failures to date have been external (WooCommerce API rate limits, Bling API downtime), detected within 5 minutes by CloudWatch alarms, and resolved without manual intervention via the retry and circuit breaker logic.
