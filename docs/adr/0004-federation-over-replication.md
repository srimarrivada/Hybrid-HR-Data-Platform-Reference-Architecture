# ADR-0004: Federation Over Replication

- **Status:** Accepted
- **Scope:** Seam
- **Diagram marker:** D4

## Context
Platform 2's Gold models need worker identity and organization context. These are conformed dimensions mastered in Platform 1. Copying them into the lakehouse would create a second version of the truth, a sync lag, and a reconciliation burden.

## Decision
Platform 2 reads approved conformed dimensions **in place from Platform 1** through a small number of governed **federated views** (e.g., Lakehouse Federation). No rows are copied.

Keep **two views of different weight**:
- A lightweight **identity view** (keys, name, status) for most lookups.
- A heavier **organization-context view** (business unit, level, location, job family), used only where organization context is needed, such as coverage analysis.

Two usage patterns:
1. **ETL-time key lookup (preferred):** resolve surrogate keys during the pipeline run and store them in Gold facts. Dashboards over those facts never trigger federation queries.
2. **Reporting-time join (limited):** used only for lightweight reporting views that need current organization context, read once per scheduled refresh rather than per click. → [D6](0006-bi-tool-by-audience.md)

## Alternatives Considered
| Option | Why Not Chosen |
|---|---|
| Nightly copy of dimensions into the lakehouse | Sync lag, duplicate pipelines, and drift between copies. |
| CDC stream of dimensions | Extra infrastructure and complexity for slowly changing data. |
| Call the HCM system's APIs directly | Rate limits, and the data is not conformed. |

## Consequences
**Positive**
- One version of the truth for worker and organization context.
- No replication pipelines to build or reconcile.
- Source-side security still applies.

**Negative / Trade-offs**
- Runtime dependency on Platform 1 availability and performance. → [R-01](../risks.md), [F4](../failure-modes.md#f4-federation-slow-or-enterprise-warehouse-unavailable)
- Adds load to the source and latency across the network.
- Reads before the warehouse publishes see the previous day's data.

## Guardrails
- Expose **views only**, never whole schemas; limit columns to what is needed.
- Current-row filters are centralized in the views, not repeated by every consumer.
- Read-only, least-privilege service account; secrets kept in a vault.
- No restricted or highly restricted data is exposed through federation. → [D11](0011-defense-in-depth-security.md), [R-04](../risks.md)
- Records whose worker can't be resolved yet get the **unknown-member key** and are reconciled after the warehouse publish. Keys that stay unresolved beyond the expected window raise a DQ alert. → [D5](0005-blocking-vs-alerting-dq.md), [D13](0013-fact-design-patterns.md)
- Monitor query latency, failures and source load.

## Revisit When (Materialize Instead)
Materialization is an explicit decision, never the default. Reconsider when:
- Query concurrency is high or latency targets are missed.
- Large unfiltered scans repeat.
- Source load becomes a concern.
- BI refresh windows grow too long.

If materialization is chosen, record it as a new ADR that names the views materialized and the refresh approach.

## Related
- [Federation Pattern](../architecture/federation-pattern.md)
- [D1: Two platforms, Not One](0001-two-platforms-not-one.md)
- [F4: Federation Slow or Enterprise Warehouse Unavailable](../failure-modes.md#f4-federation-slow-or-enterprise-warehouse-unavailable)
- [Risks](../risks.md)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*