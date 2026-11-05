# ADR-0006: BI Tool and Refresh Mode by Audience

- **Status:** Accepted
- **Scope:** Both
- **Diagram marker:** D6

## Context
Consumers differ in size, freshness needs and sensitivity:
- A **large governed HR audience** needs row-level security and some restricted data.
- A **small technical operations team** needs status data that is minutes old.
- **Business users of lakehouse data** need fast dashboards refreshed daily.

One tool and one refresh mode can't serve all three well.

## Decision
Choose the BI tool and refresh mode by audience:

| Audience | Pattern |
|---|---|
| Broad governed HR audience (Platform 1) | **Composite semantic model** (e.g., Power BI): imported dimensions and aggregates for speed; DirectQuery for large detail facts and highly restricted facts, so that data stays in the source. Row-level security on top of schema security. Refresh is **triggered by the warehouse publish**, not by a clock. |
| Operations team (Platform 2) | **Live SQL dashboard** (e.g., Databricks SQL) on Gold views through a SQL warehouse; no cache. |
| Lakehouse business users (Platform 2) | **Imported dashboards** on Gold reporting views, refreshed on a staggered schedule after the upstream cycles they depend on. Federated context is read once per refresh, not per click. → [D4](0004-federation-over-replication.md) |

Performance patterns that support this:
- **Aggregate facts** (e.g., headcount, events, hours by period) serve trend visuals, so dashboards don't scan detail rows. → [D13](0013-fact-design-patterns.md)
- **Snapshot grain is a deliberate trade-off, chosen per subject area.** For example, monthly periodic snapshots give enough trend value at a fraction of daily-snapshot storage, while daily detail stays available through transaction facts.

## Alternatives Considered
| Option | Why Not Chosen |
|---|---|
| Import mode for everything | Operations data would be stale; highly restricted data would be copied into the BI layer. |
| DirectQuery / live queries for everything | Heavy source load and slow dashboards for large audiences. |
| One live SQL tool for everyone | Weaker row-level security and visuals for the broad business audience. |

## Consequences
**Positive**
- Freshness, cost and security are matched to each audience.
- Highly restricted data stays in the governed source. → [D11](0011-defense-in-depth-security.md)
- Staggered refreshes avoid compute contention.
- If the warehouse publish is held, the dependent refreshes don't run, so dashboards keep the last good data. → [F2](../failure-modes.md#f2-data-quality-gate-below-threshold)

**Negative / Trade-offs**
- Two BI surfaces to support.
- KPI definitions could diverge between tools. → [R-03](../risks.md)
- Shared compute can still contend during refresh windows. → [R-06](../risks.md)

## Guardrails
- Each KPI has exactly one owning platform and definition, recorded in a KPI catalog.
- Business logic lives in semantic models and Gold views, not in individual dashboards.
- Raw tables are never exposed to business users.
- Every dashboard shows a freshness label.
- Refreshes are triggered by upstream readiness, not fixed clock times.

## Revisit When
- Concurrency on the live dashboard grows significantly.
- A shared semantic layer across both platforms becomes practical.
- An operational audience grows large enough to need row-level security and richer visuals.

## Related
- [Consumption Pattern](../architecture/consumption-pattern.md)
- [Design Principle 12](../design-principles.md#12-right-bi-tool-for-the-audience)
- [D4: Federation Over Replication](0004-federation-over-replication.md)
- [Risks](../risks.md)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*