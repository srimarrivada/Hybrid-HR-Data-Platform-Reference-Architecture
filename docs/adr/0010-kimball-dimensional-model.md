# ADR-0010: Kimball Dimensional Model for the Enterprise Warehouse

- **Status:** Accepted
- **Scope:** Platform 1 (and Platform 2 Gold)
- **Diagram marker:** D10

## Context
A large HR audience needs self-service reporting with consistent definitions, accurate point-in-time history and fast queries, replacing scattered one-off reports. The same worker, organization and date context must mean the same thing in every subject area.

## Decision
Use a **Kimball star schema / constellation**: conformed dimensions shared across subject areas, with facts chosen by the kind of question each one answers.

**Dimensions**
- **Conformed dimensions** (worker, organization, job, location, date) are shared across all subject areas and owned by Platform 1.
- **SCD Type 2** where history matters; Type 1 for corrections. → [D8](0008-change-detection-strategy.md)
- **Role-playing dimensions** (e.g., several date roles; worker as employee, manager or recruiter).

**Facts**

| Fact type | Answers | HR example |
|---|---|---|
| Transaction | What happened, when? | Hires, terminations, movements |
| Periodic snapshot | What was the state at period end? | Headcount by month |
| Accumulating snapshot | How far has a process progressed? | Hiring pipeline stages |
| Factless | Did an event or coverage occur? | Training eligibility vs enrollment |
| Aggregate | Fast trend answers | Headcount trend by business unit |

**Bridges** handle many-to-many relationships. Detailed patterns (consolidated facts, junk dimensions, weighted bridges, unknown-member rows) are in [D13](0013-fact-design-patterns.md).

The same approach is used for Platform 2 Gold models, which reuse Platform 1's conformed dimensions through federation. → [D4](0004-federation-over-replication.md)

## Alternatives Considered
| Option | Why Not Chosen |
|---|---|
| 3NF enterprise model | Hard for business users and BI tools to query; many joins. |
| Data Vault | Strong for auditability across many fast-changing sources, but still needs a dimensional presentation layer, so it adds a layer of build and run cost the source landscape doesn't justify. |
| Reporting directly on raw views | No history, inconsistent KPIs, slow queries. |

## Consequences
**Positive**
- Intuitive for business users and BI tools.
- Consistent KPIs across subject areas through conformed dimensions.
- Fast queries, with aggregates for trend visuals. → [NFR-06](../nfrs.md)
- Accurate point-in-time history. → [NFR-10](../nfrs.md)

**Negative / Trade-offs**
- ETL build effort for dimensions, facts and history.
- Schema changes ripple through dimensions and facts.
- Conformed dimensions need clear ownership, or subject areas start creating their own copies.

## Guardrails
- **Declare the grain first:** every fact states its grain in one sentence before any columns are added.
- **Bus matrix:** a matrix of facts × conformed dimensions is kept current and reviewed for every new subject area.
- **One owner per conformed dimension** (Platform 1); no local copies in other subject areas or in the lakehouse.
- **Surrogate keys** in facts, with unknown-member rows for late-arriving dimensions. → [D13](0013-fact-design-patterns.md)
- **Model changes pass the DQ gate** (referential integrity, SCD2 consistency) before publish. → [D5](0005-blocking-vs-alerting-dq.md)
- Business logic lives in the model and semantic layer, not in dashboards. → [D6](0006-bi-tool-by-audience.md)

## Revisit When
- Source count and change rate grow enough that a Data Vault integration layer behind the star schema becomes worthwhile.
- The warehouse moves to a lakehouse, where the same dimensional model can be kept on Delta tables.

## Related
- [Platform 1](../architecture/platform-1-batch-warehouse.md)
- [D13: Fact and Dimension Design Patterns](0013-fact-design-patterns.md)
- [D2: Temporal ODS Family](0002-temporal-ods-family.md)
- [Business Context](../business-context.md)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*