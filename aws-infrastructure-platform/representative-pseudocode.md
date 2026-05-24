# Representative Pseudocode

This is intentionally pseudocode. It shows the engineering shape of the solution without exposing production source code.

## Reusable Terraform module composition

```hcl
module "orders_sync" {
  source = "../../modules/lambda-etl"

  function_name = "client-orders-sync"
  runtime       = "python3.12"
  timeout       = 120
  memory_size   = 512

  secrets_read = [
    aws_secretsmanager_secret.database.arn,
    aws_secretsmanager_secret.external_api.arn
  ]

  alarm_topic_arn = aws_sns_topic.production_alerts.arn
}

module "orders_schedule" {
  source = "../../modules/eventbridge-schedule"

  name          = "orders-sync-every-3-hours"
  schedule      = "rate(3 hours)"
  target_arn    = module.orders_sync.lambda_arn
}
```

## Least-privilege IAM pattern

```hcl
statement {
  actions   = ["secretsmanager:GetSecretValue"]
  resources = var.secrets_read
}

statement {
  actions   = ["cloudwatch:PutMetricData"]
  resources = ["*"]
  condition {
    test     = "StringEquals"
    variable = "cloudwatch:namespace"
    values   = [var.metric_namespace]
  }
}
```

## Approval-gated production apply

```yaml
# GitHub Actions-style pseudocode
on:
  pull_request:
    jobs: [terraform_fmt, terraform_validate, terraform_plan]
  push_to_main:
    jobs:
      - terraform_plan
      - wait_for_environment_approval
      - terraform_apply
```
