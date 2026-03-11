# Technical Decisions

## Terraform over AWS CDK

CDK is an imperative framework that generates CloudFormation. Terraform is declarative and manages its own state.

The key difference that drove the decision was state management. Terraform tracks exactly what resources exist in each environment and what would change with a given plan, without needing to diff a CloudFormation stack or trust that a `cdk deploy` and the actual AWS state are in sync. For a consultancy managing multiple client environments, that explicit state model reduces the risk of silent drift.

CDK requires writing TypeScript or Python to describe infrastructure, which adds cognitive overhead for engineers who are primarily reading and writing application code in those languages, not infrastructure logic. HCL is narrow enough that it rarely surprises you.

The other consideration was vendor lock-in. Terraform modules work across AWS, GCP, and Azure. If a client ever moves partially or entirely to a different provider, the module patterns transfer. CDK generates CloudFormation which is AWS-only.

**Trade-off:** CDK offers more programmatic flexibility for complex conditional logic. For the current workload, that flexibility was not needed. Modules and `for_each` handle the variation across environments.

## Terraform over CloudFormation

CloudFormation is the native AWS IaC tool. It is deeply integrated and requires no external state management. The reasons to move to Terraform were portability (same argument as CDK), a cleaner syntax for complex resources, and better tooling for plan review. CloudFormation change sets exist but are harder to read than `terraform plan` output, which matters when you are posting the plan as a PR comment for review.

## Lambda over EC2

EC2 runs continuously. Lambda runs on demand. The workloads in question (ETL syncs, RFV recalculation, cluster computation) execute for 10 to 180 seconds, a few times per day. An always-on t3.small at R$ 180/month for workloads that are idle 99% of the time is not a defensible infrastructure choice.

Lambda cold starts are under one second for all but the container-based `recalc_clusters` function, which is acceptable for scheduled workloads that are not user-facing.

**Trade-off:** Lambda has a 15-minute maximum execution time. If a sync function is processing a backfill of historical data, it needs to be designed with cursor-based pagination so it can continue across multiple invocations. The alternative, a single long-running process on EC2, is simpler to reason about but costs more and requires managing the compute lifecycle.

## Lambda over AWS Glue

Glue is designed for distributed Spark workloads processing millions of records. The Silva Nutrition data warehouse has approximately 72,000 records, growing by 100 to 500 per day. Using Glue for this is like using a freight truck to deliver groceries.

| Dimension | Lambda | Glue |
|---|---|---|
| Cold start | <1 second | 45-60 seconds |
| Monthly cost | R$ 15 | R$ 180-450 |
| Minimum viable volume | Any | 1M+ records/day |
| Spark dependency | Not required | Required |

Glue becomes the right choice when transformations require distributed processing or when data volume justifies the startup overhead. At current scale, it would add cost and latency without any benefit.

## EventBridge over CloudWatch Events

EventBridge is CloudWatch Events v2. Same underlying service, more features, better event bus capabilities for future integration needs. The migration cost was zero and there was no reason to use the legacy interface.

## SNS fan-out over SES for alerting

The alerting design requires sending the same notification to multiple destinations: two email addresses and a Slack channel. SES sends email. SNS is a fan-out service that routes a single message to multiple protocol-specific subscriptions.

Using SNS means the alarm connects to one SNS topic, and the routing logic lives in subscriptions, not in the alarm configuration. Adding a new notification destination (a third email, a PagerDuty endpoint, a different Slack channel) is a subscription change, not an alarm change.

The Slack delivery goes through a Lambda subscriber rather than a direct SNS-to-Slack integration because it needs formatting: the raw SNS payload becomes a structured Slack Blocks message with the Lambda name, the error, and a direct link to the CloudWatch log stream.

## DynamoDB for distributed lock

The Bling OAuth2 integration uses a refresh token that is single-use: after a token refresh, the old refresh token is immediately invalidated. If two Lambda functions attempt to refresh the token at the same time, only one succeeds and the other corrupts the stored credential.

The solution is a distributed lock. When a Lambda needs to refresh the Bling token, it first attempts to acquire a lock via a conditional DynamoDB write. If the write succeeds, the lock is held. If it fails because the item already exists, another function holds the lock and the caller waits and then reads the already-refreshed token from Secrets Manager.

DynamoDB was already in the stack for Terraform state locking. Using it for application-level distributed locking avoided introducing a new service (Redis, for example) for a single use case. The lock has a TTL of 60 seconds so a crashed Lambda cannot hold it indefinitely.

## S3 state backend with versioning

Terraform state contains the current record of all managed resources. Losing or corrupting it means losing the ability to manage those resources without manual reconciliation. S3 with versioning enabled means every state change is recorded and any version can be restored. The state file is also encrypted at rest.

## GitHub Actions over dedicated CI/CD tools

No separate tool was introduced. GitHub Actions is already available, integrates natively with pull request comments for plan output, supports environment-gated deployments with required reviewers, and has no additional cost at the current usage level.
