# Key Flows

## New client onboarding

Before IaC, onboarding a new client meant manually navigating the AWS console, creating resources one by one, and hoping the configuration matched the previous environment. The process took 4 to 8 hours and produced no audit trail.

With the module-based Terraform setup, onboarding is:

1. Copy `projects/client-analytics/` to `projects/new-client/`
2. Update `terraform.tfvars` with client-specific values (function names, schedule crons, secret ARNs)
3. Add Lambda source code to `lambdas/`
4. Open a PR, which triggers `terraform plan` automatically
5. Review the plan output in the PR comment
6. Merge and approve the production deployment gate in GitHub Actions
7. `terraform apply` runs, creating all resources in the correct order

The entire flow from code to production takes 15 minutes. The plan review step catches mistakes before they reach AWS.

## PR validation flow (terraform-ci)

Every pull request triggers this pipeline:

```
PR opened or updated
  └── terraform fmt -check       (fails if formatting is inconsistent)
  └── terraform init              (initializes remote backend)
  └── terraform validate          (syntax and schema validation)
  └── terraform plan              (generates execution plan)
  └── Post plan as PR comment     (reviewers see exact changes)
```

The plan comment shows which resources will be created, modified, or destroyed, with the full Terraform output. Reviewers can see if a change would replace a Lambda function (causing a brief cold start) or if it would unexpectedly destroy a resource. Approving a PR is approving the plan, not just the code.

## Production deploy flow (terraform-cd)

```
Merge to main
  └── terraform plan              (regenerated to catch drift)
  └── Wait for manual approval    (required reviewer in GitHub environment)
  └── terraform apply             (creates/updates/destroys resources)
  └── Outputs published           (Lambda ARNs, log group names)
```

The approval gate has a 24-hour window. If no reviewer acts, the workflow expires and a new merge is required to trigger another attempt. This prevents stale approvals from executing days later.

## Rollback flow

Terraform rollback is a `git revert` followed by a PR and a new apply:

1. Identify the commit that introduced the problem
2. `git revert <commit>` to produce an inverse commit
3. Open a PR (plan will show the resource returning to its prior state)
4. Merge and approve
5. `terraform apply` restores the previous configuration

For Lambda code changes without infrastructure changes, a rollback is just deploying the previous artifact. Because Lambda versions are immutable, the prior version remains available and can be pointed to without a redeployment.

## Bling OAuth2 token refresh with distributed lock

The Bling API uses a single-use refresh token: using it produces a new access token and a new refresh token, invalidating the old one. If two Lambda functions attempt to refresh the token simultaneously, the second one uses an already-invalidated token and fails, corrupting the stored credential and breaking all Bling integrations until manual recovery.

The distributed lock flow:

```
Lambda needs Bling access token
  └── Load token from Secrets Manager cache
  └── If token valid: return token (no lock needed)
  └── If token expired:
        └── Attempt DynamoDB conditional write (acquire lock)
        └── If lock acquired:
              └── Call Bling /oauth2/token with refresh_token
              └── Store new access_token + refresh_token in Secrets Manager
              └── Release lock (delete DynamoDB item)
              └── Return new access token
        └── If lock not acquired (another Lambda holds it):
              └── Sleep 5 seconds
              └── Read token from Secrets Manager (already refreshed)
              └── Return token
```

The lock TTL is 60 seconds. A Lambda that crashes after acquiring the lock cannot hold it indefinitely.

## Circuit breaker activation

Each ETL Lambda tracks consecutive failures internally. After 5 consecutive errors processing records from an external API, the Lambda logs `CIRCUIT_BREAKER_OPEN` and raises an exception to stop execution. A CloudWatch metric filter detects that log pattern and triggers an SNS alarm within 5 minutes.

This prevents a Lambda from looping indefinitely against an API that is timing out or returning errors, accumulating retry costs and filling logs with noise. A human reviews the alarm and decides whether the issue is transient (the pipeline will recover on the next scheduled run) or requires intervention.
