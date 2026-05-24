# Start Here

This repository is a portfolio of real engineering case studies. It is written for two audiences: recruiters who need a quick signal of scope, and engineering managers who want to evaluate technical judgment.

## Recommended Reading Paths

### 5-minute recruiter path

Read only:

1. [README](README.md)
2. [Higher Education Enrollment Platform](higher-education-enrollment-platform/README.md)
3. [E-commerce Analytics Data Platform](ecommerce-analytics-data-platform/README.md)
4. [AWS Infrastructure Platform](aws-infrastructure-platform/README.md)

This gives the shortest view of production scope: backend, data, cloud, security, migration, observability, and AI-assisted workflows.

### 15-minute engineering manager path

Read:

1. [Higher Education Enrollment Platform - Technical Decisions](higher-education-enrollment-platform/technical-decisions.md)
2. [E-commerce Analytics Data Platform - Architecture](ecommerce-analytics-data-platform/architecture.md)
3. [AWS Infrastructure Platform - Architecture](aws-infrastructure-platform/architecture.md)
4. [Internal Proposal Agent - Technical Decisions](internal-proposal-agent/technical-decisions.md)

This shows how problems were framed, what trade-offs were made, and where reliability/security boundaries were placed.

### Deep technical path

Read the `representative-pseudocode.md` files first, then open each `architecture.md`, `key-flows.md`, and `reliability-and-ops.md`.

## What This Repository Is Not

- It is not a collection of tutorial projects.
- It is not a code dump from client systems.
- It does not expose confidential source code, client names, credentials, infrastructure identifiers, or proprietary business data.

## What To Evaluate

Look for the engineering judgment behind each case:

- Where the system boundary was placed.
- How production risk was reduced.
- How data quality was restored.
- How failures were made visible.
- How AI was constrained by deterministic code and human review.
- How infrastructure moved from manual configuration to reproducible operations.
