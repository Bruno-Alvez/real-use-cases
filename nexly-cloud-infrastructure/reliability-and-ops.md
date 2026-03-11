# Reliability and Ops

## Alarm design

The observability strategy is built around known failure modes rather than generic thresholds. Each alarm covers a specific failure scenario with a defined action.

| Alarm | Metric | Threshold | Evaluation | Action | Rationale |
|---|---|---|---|---|---|
| Error rate | Lambda Errors | >0 | 1 period of 5 min | SNS Critical | Retry is built in; any error reaching the alarm is an anomaly |
| Duration | Lambda Duration | >240s | 1 occurrence | SNS Critical | Timeout is 300s; alarm at 80% to allow investigation before timeout |
| Zero records | Custom metric | =0 | 1 period of 24h | SNS Warning | API may be down or pipeline silently broken |
| Token refresh failed | Log pattern | `BLING_TOKEN_REFRESH_FAILED` | 1 occurrence | SNS Critical | Blocks all three Bling Lambdas until resolved |
| Circuit breaker | Log pattern | `CIRCUIT_BREAKER_OPEN` | 1 occurrence | SNS Critical | API instability requires manual investigation |

Total: 28 alarms.
- 9 Lambdas × 2 alarms (error + duration) = 18
- 8 ETL Lambdas × 1 alarm (zero records) = 8
- 3 Bling Lambdas × 1 alarm (token refresh) = 3 (via metric filter on log pattern)
- 1 global circuit breaker alarm

## Alert routing

```
CloudWatch Alarm (ALARM state)
  └── SNS Topic: nexly-etl-alertas
        ├── Email: bruno.santos@nexlygroup.com
        ├── Email: joao.passos@nexlygroup.com
        └── Lambda: slack_notifier
              └── Formats Slack Blocks message
              └── Posts to #infra-nexly channel
```

The `slack_notifier` Lambda receives the SNS event, extracts the alarm name and state, and formats a structured message with the Lambda name, error description, and a direct link to the CloudWatch log stream for the failing invocation. The Slack webhook URL is stored in Secrets Manager.

## Incident record: Bling token refresh race condition

**Date:** During initial deployment validation (before production traffic)

**Symptom:** Two Lambdas scheduled close together (`sync_pedidos_bling` at 06:00 and `sync_objetos_bling` also at 06:00) both attempted to refresh an expired Bling token simultaneously. The second refresh used an already-invalidated token and received a 401 from the Bling OAuth2 endpoint. The stored `refresh_token` in Secrets Manager was then in an inconsistent state because the first Lambda had updated it but the second Lambda had also attempted an update with stale credentials.

**Detection:** During integration testing before go-live. Both Lambdas failed, the `BLING_TOKEN_REFRESH_FAILED` log pattern was observed in CloudWatch, and the alarm triggered.

**Root cause:** The OAuth2 refresh token is single-use. Concurrent refreshes are mutually exclusive by design, but the initial implementation had no coordination mechanism.

**Resolution:** Implemented a distributed lock via DynamoDB with a conditional write and 60-second TTL. A Lambda holding the lock completes the refresh and writes the new token. Any other Lambda that cannot acquire the lock waits 5 seconds and reads the already-refreshed token. The lock TTL ensures a crashed Lambda cannot hold it indefinitely.

**Lesson:** Distributed systems that depend on single-use tokens need explicit coordination. The issue was caught in testing rather than production because the deployment validation intentionally ran all Bling functions in rapid succession. This is a useful testing pattern for any system with shared mutable state.

## State locking incident

**Symptom:** During early CI/CD setup, a PR pipeline and a manual `terraform apply` ran simultaneously. The state file was corrupted and `terraform plan` began producing incorrect output.

**Detection:** The plan showed resources being destroyed that had not been changed. The discrepancy was immediately visible before any apply ran.

**Resolution:** Enabled DynamoDB state locking. Terraform now acquires a lock before any state-modifying operation and releases it on completion. Concurrent runs fail fast with a clear error rather than silently corrupting state.

## Operational runbook (abbreviated)

**A Lambda is in ERROR state:**
1. Check CloudWatch logs for the specific invocation
2. Determine if the error is transient (API timeout, rate limit) or persistent
3. For transient errors: the next scheduled run will retry; monitor the next execution
4. For persistent errors (schema mismatch, auth failure): fix the root cause and trigger manual invocation via AWS console or `aws lambda invoke`

**Zero records alarm fires:**
1. Check if the source API is reachable
2. Check if the cursor timestamp is correct in the target table
3. If cursor is too far in the future (data migration artifact), reset via SQL
4. Trigger manual invocation

**Bling token expired:**
1. Check if `BLING_TOKEN_REFRESH_FAILED` log pattern appears
2. If DynamoDB lock is stuck (TTL not expired), manually delete the lock item
3. Trigger `sync_pedidos_bling` manually; the refresh will run as part of the invocation
4. If the refresh token itself is invalid (expired after 30 days of non-use), re-authorize via Bling console and update the secret manually
