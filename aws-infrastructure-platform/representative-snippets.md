# Representative Snippets

These snippets illustrate the architectural patterns used in this project. They are representative examples, not production source code.

## Terraform module: lambda-etl

The module encapsulates everything a Lambda ETL function needs: the function itself, its IAM role with least-privilege policies, its log group, and its standard alarms.

```hcl
# modules/lambda-etl/main.tf

resource "aws_lambda_function" "this" {
  function_name = var.function_name
  role          = aws_iam_role.this.arn
  handler       = var.handler
  runtime       = var.runtime
  timeout       = var.timeout
  memory_size   = var.memory_size
  filename      = data.archive_file.source.output_path
  layers        = var.layer_arns

  environment {
    variables = var.environment_variables
  }
}

resource "aws_iam_role" "this" {
  name = "${var.function_name}-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "lambda.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy" "secrets" {
  role = aws_iam_role.this.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["secretsmanager:GetSecretValue"]
      Resource = var.secret_arns  # Only the specific secrets this function needs
    }]
  })
}

resource "aws_cloudwatch_log_group" "this" {
  name              = "/aws/lambda/${var.function_name}"
  retention_in_days = 30
}

# Standard alarms included with every Lambda, no configuration needed
module "error_alarm" {
  source          = "../cloudwatch-alarm"
  alarm_name      = "${var.function_name}-errors"
  metric_name     = "Errors"
  namespace       = "AWS/Lambda"
  threshold       = 0
  comparison      = "GreaterThanThreshold"
  sns_topic_arn   = var.sns_alarm_topic_arn
  dimensions      = { FunctionName = aws_lambda_function.this.function_name }
}
```

## DynamoDB distributed lock

Used to prevent race conditions when multiple Lambda functions attempt to refresh the Bling OAuth2 token simultaneously.

```python
# layers/shared/python/lib/dynamodb_lock.py (representative)

class DistributedLock:
    def __init__(self, table_name: str, lock_key: str, ttl_seconds: int = 60):
        self.table = boto3.resource("dynamodb").Table(table_name)
        self.lock_key = lock_key
        self.ttl = int(time.time()) + ttl_seconds

    def acquire(self) -> bool:
        try:
            self.table.put_item(
                Item={"lock_key": self.lock_key, "ttl": self.ttl},
                ConditionExpression="attribute_not_exists(lock_key)"
            )
            return True
        except ClientError as e:
            if e.response["Error"]["Code"] == "ConditionalCheckFailedException":
                return False
            raise

    def release(self) -> None:
        self.table.delete_item(Key={"lock_key": self.lock_key})


def refresh_bling_token_safely(secrets_client, dynamo_client):
    lock = DistributedLock(table_name="nexly-distributed-locks", lock_key="bling-token-refresh")

    if lock.acquire():
        try:
            new_token = call_bling_oauth2_refresh()
            secrets_client.update_secret(SecretId="company/client-analytics/bling", SecretString=json.dumps(new_token))
            return new_token
        finally:
            lock.release()
    else:
        # Another Lambda holds the lock, it will complete the refresh.
        # Wait briefly and read the result.
        time.sleep(5)
        return secrets_client.get_secret_value(SecretId="company/client-analytics/bling")
```

## Circuit breaker pattern

All ETL Lambdas use this pattern. Five consecutive failures trigger a halt and a CloudWatch alarm.

```python
# Representative pattern used in all ETL handler functions

def process_with_circuit_breaker(items: list, process_fn: callable, logger) -> dict:
    consecutive_failures = 0
    results = {"processed": 0, "errors": 0}

    for item in items:
        try:
            process_fn(item)
            results["processed"] += 1
            consecutive_failures = 0  # Reset on success
        except Exception as e:
            consecutive_failures += 1
            results["errors"] += 1
            logger.warning({"event": "item_error", "item_id": item.get("id"), "error": str(e)})

            if consecutive_failures >= 5:
                logger.critical({"event": "CIRCUIT_BREAKER_OPEN", "consecutive_failures": consecutive_failures})
                raise RuntimeError("Circuit breaker triggered after 5 consecutive failures") from e

    return results
```

## Structured logging format

All Lambdas log JSON to CloudWatch. The same fields appear in every log line so CloudWatch Insights queries work consistently across all functions.

```python
# layers/shared/python/lib/logger.py (representative)

def log(level: str, event: str, **kwargs) -> None:
    entry = {
        "timestamp": datetime.utcnow().isoformat() + "Z",
        "level": level,
        "lambda": os.environ.get("AWS_LAMBDA_FUNCTION_NAME"),
        "event": event,
        **kwargs
    }
    print(json.dumps(entry))


# Usage in a sync Lambda
log("INFO", "sync_completed",
    records_processed=42,
    records_inserted=30,
    records_updated=12,
    execution_time_ms=1847,
    cursor_next="2026-02-16T10:00:00Z")
```

This makes the following CloudWatch Insights query work across all ETL Lambdas:

```
fields @timestamp, lambda, records_processed, execution_time_ms
| filter event = "sync_completed"
| stats sum(records_processed) by lambda
| sort sum_records_processed desc
```

## Rate limiting: WooCommerce

```python
# Representative rate limit handling

def fetch_page(session, url: str, params: dict) -> requests.Response:
    response = session.get(url, params=params, timeout=30)
    response.raise_for_status()

    remaining = int(response.headers.get("X-RateLimit-Remaining", 100))
    if remaining < 10:
        reset_in = int(response.headers.get("X-RateLimit-Reset", 60))
        log("WARNING", "rate_limit_approaching", remaining=remaining, sleeping_seconds=reset_in)
        time.sleep(reset_in)

    return response
```

## Terraform module instantiation (projects level)

This shows how a project file instantiates the shared modules. A new client environment reuses the same modules with different variable values.

```hcl
# projects/client-analytics/main.tf

module "sync_clientes" {
  source = "../../modules/lambda-etl"

  function_name    = "silva-sync-clientes"
  handler          = "handler.lambda_handler"
  runtime          = "python3.12"
  timeout          = 300
  memory_size      = 512
  source_code_path = "${path.module}/lambdas/sync_clientes"
  layer_arns       = [aws_lambda_layer_version.shared.arn]
  secret_arns      = [data.aws_secretsmanager_secret.postgres.arn, data.aws_secretsmanager_secret.woocommerce.arn]
  sns_alarm_topic_arn = aws_sns_topic.alerts.arn
}

module "schedule_sync_clientes" {
  source           = "../../modules/eventbridge-schedule"
  name             = "silva-sync-clientes-schedule"
  schedule         = "cron(0 */6 * * ? *)"
  target_lambda_arn = module.sync_clientes.function_arn
}
```
