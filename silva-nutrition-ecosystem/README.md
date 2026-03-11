# Silva Nutrition Ecosystem

Full reconstruction of a production e-commerce data platform, migrating from a system with critical structural debt to an enterprise-grade data warehouse, analytics layer, and ETL pipeline, with zero downtime throughout.

**Period:** January-February 2026
**Status:** In production
**Role:** Sole architect and implementer

---

| Dimension | Before | After |
|---|---|---|
| Data quality score | 2/10 | 9/10 |
| Problematic records | 89,487 | 0 |
| Orders coverage | 26% | 100% |
| p95 API latency | >30s (timeout) | <2s |
| Tables with RLS | 23% (5/22) | 100% (18/18) |
| LGPD compliance | Critical violation | Fully compliant |
| Disaster recovery | Impossible | 15-minute RTO |

---

## Case study files

- [Overview](overview.md) — context, audit findings, and solution strategy
- [Architecture](architecture.md) — Strangler Fig migration, Star Schema design, Lambda ETL
- [Technical Decisions](technical-decisions.md) — migration strategy, data modeling, infrastructure choices
- [Key Flows](key-flows.md) — migration flow, incremental sync, Product Bundles incident
- [Representative Snippets](representative-snippets.md) — UPSERT patterns, RLS policies, circuit breaker
- [Reliability and Ops](reliability-and-ops.md) — 28 CloudWatch alarms, MTTR, bot attack response
- [Results](results.md) — quantified outcomes across data, application, and infrastructure layers
