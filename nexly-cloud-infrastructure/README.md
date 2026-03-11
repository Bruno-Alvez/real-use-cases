# Nexly Cloud Infrastructure

Infrastructure as Code standardization for Nexly Group's AWS environment. Replaced manual ClickOps operations with a modular, versioned, and reproducible platform that provisions a full production environment in 15 minutes.

**Period:** February 2026
**Status:** In production
**Role:** Sole architect and implementer

---

| Dimension | Before | After |
|---|---|---|
| Time to deploy | 4-8 hours | 15 minutes |
| Operational cost | R$ 8,230/month | R$ 415/month |
| Client onboarding | 2-4 weeks | 1 day |
| Disaster recovery | Impossible | 15-minute RTO |
| Audit trail | None | Full Git history |
| MTTR | >24 hours | <5 minutes |

---

## Case study files

- [Overview](overview.md) — context, problem, and solution approach
- [Architecture](architecture.md) — Terraform module design, AWS resources, CI/CD pipeline
- [Technical Decisions](technical-decisions.md) — Lambda vs EC2, Terraform vs CDK, state management
- [Key Flows](key-flows.md) — deploy flow, onboarding flow, token refresh with distributed lock
- [Representative Snippets](representative-snippets.md) — module structure, circuit breaker, structured logging
- [Reliability and Ops](reliability-and-ops.md) — CloudWatch alarms, MTTR, incident record
- [Results](results.md) — quantified outcomes
