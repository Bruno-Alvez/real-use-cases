# Overview

## Context

Nexly Group is a technology consultancy delivering backend systems and data products for B2B clients. By early 2026, the team was operating AWS infrastructure entirely through the console, configuring services by hand and tracking parameters in shared documents. Each client deployment was a bespoke manual process that took anywhere from 4 to 8 hours, required constant availability from the person who had done it before, and left no audit trail of what had been changed or why.

The immediate trigger for this project was onboarding a second production client. It became clear that repeating the same artisanal setup was not viable. Beyond time, the manual model had no disaster recovery path, no consistent environment between staging and production, and credentials spread across Lambda environment variables with no rotation or access control.

## Problem

The infrastructure had six concrete failure modes that were affecting the business:

**No reproducibility.** Recreating any environment required someone to remember what had been configured, which invariably led to differences between dev and prod. Bugs that were present in production could not be replicated locally because the environments were not the same.

**No audit trail.** There was no record of who changed what and when. Compliance audits were impossible. Rolling back a bad configuration meant manually reversing steps from memory.

**Vendor lock-in via N8N.** All data pipelines ran in N8N SaaS, a proprietary automation platform with no version control, no observability, and limited error handling. Migrating away would require rebuilding every workflow. The tool accounted for R$ 450/month and delivered zero visibility into pipeline failures.

**No observability.** Pipeline failures were discovered manually, sometimes days later. MTTR exceeded 24 hours because there was no alerting. Data decisions were being made on stale or missing data without anyone knowing.

**Always-on compute.** EC2 instances and Render deployments ran 24/7 for workloads that executed for less than 30 seconds at a time, a few times per day. The workload pattern was ideal for serverless, but no one had modeled the cost difference.

**Disaster recovery was impossible.** If the AWS account had a problem, or a misconfiguration took down a resource, recovery meant rebuilding from scratch. There was no documented state of what the infrastructure looked like.

## Solution

I designed a Terraform-based IaC platform with reusable modules organized by domain, a CI/CD pipeline via GitHub Actions that validates every change on PR and requires approval for production deploys, and a migration of all compute from always-on instances to AWS Lambda with EventBridge scheduling.

The design principle was simple: infrastructure state lives in code, not in someone's head. A new engineer joining the team should be able to understand the entire AWS footprint by reading the repository, and a new client environment should require changing a single configuration file before running `terraform apply`.

The first production target for this platform was the Silva Nutrition ecosystem, which had nine Lambda functions, eight EventBridge schedules, twenty-eight CloudWatch alarms, and four Secrets Manager entries, all provisioned from scratch with this infrastructure.

## Scope

- Terraform module library covering Lambda, EventBridge, CloudWatch alarms, and Secrets Manager
- S3 remote state with DynamoDB state locking for concurrent execution safety
- GitHub Actions CI/CD with plan-on-PR and approval-gated apply on main
- IAM configuration with least-privilege roles per Lambda function
- AWS Identity Center for team SSO
- Billing alarms and cost monitoring
- Full observability for the first production deployment (Silva Nutrition): 28 CloudWatch alarms, structured logging, SNS-to-Slack alerting
