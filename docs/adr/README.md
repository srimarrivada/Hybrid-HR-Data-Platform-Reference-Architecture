# Architecture Decision Records

Each record captures one significant design decision: its context, the options considered, the choice made, its consequences, and the signals that would cause it to be revisited. Records follow a lightweight ADR format and describe a **generalized reference architecture**, not any specific organization's system.

Decision numbers (D1–D13) match the gold markers on the [detailed diagram](../diagrams/svg/hybrid-hr-platform-reference-architecture.drawio.svg).

| ADR | Marker | Decision | Scope | Status |
|---|---|---|---|---|
| [0001](0001-two-platforms-not-one.md) | D1 | Two platforms, not one | Both | Accepted |
| [0002](0002-temporal-ods-family.md) | D2 | Temporal ODS family | Platform 1 | Accepted |
| [0003](0003-dependency-tiered-batch.md) | D3 | Dependency-tiered batch orchestration | Platform 1 | Accepted |
| [0004](0004-federation-over-replication.md) | D4 | Federation over replication | Seam | Accepted |
| [0005](0005-blocking-vs-alerting-dq.md) | D5 | Blocking vs alerting data quality | Both | Accepted |
| [0006](0006-bi-tool-by-audience.md) | D6 | BI tool and refresh mode by audience | Both | Accepted |
| [0007](0007-selective-medallion.md) | D7 | Medallion only where it adds value | Platform 2 | Accepted |
| [0008](0008-change-detection-strategy.md) | D8 | Change detection by content compare | Platform 1 (+ Platform 2) | Accepted |
| [0009](0009-idempotent-loads.md) | D9 | Idempotent loads and safe re-runs | Both | Accepted |
| [0010](0010-kimball-dimensional-model.md) | D10 | Kimball dimensional model | Platform 1 (+ Platform 2 Gold) | Accepted |
| [0011](0011-defense-in-depth-security.md) | D11 | Defense-in-depth security | Both | Accepted |
| [0012](0012-bronze-captures-transient-sources.md) | D12 | Bronze captures transient source data | Platform 2 | Accepted |
| [0013](0013-fact-design-patterns.md) | D13 | Fact and dimension design patterns | Platform 1 (+ Platform 2 Gold) | Accepted |

## Status Values
- **Proposed:** under discussion, not yet applied.
- **Accepted:** applied in the architecture.
- **Superseded:** replaced by a later ADR. The old record stays for history and links to its replacement.

## Adding a Decision
1. Copy [0000-template.md](0000-template.md) and take the next number.
2. Fill in Context, Alternatives Considered, Consequences, Guardrails and Revisit When.
3. Add a row to this table, and add a marker to the diagram if the decision applies to a specific component.
4. If it replaces an existing decision, set the old one's status to *Superseded by [ADR-NNNN](NNNN-title.md)* and update its row here.

## Related
- [Design Principles](../design-principles.md)
- [Architecture Overview](../architecture/architecture-overview.md)
- [Failure Modes](../failure-modes.md)
- [Risks](../risks.md)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*