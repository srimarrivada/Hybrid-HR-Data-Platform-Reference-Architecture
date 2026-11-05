# ADR-0013: Fact and Dimension Design Patterns

- **Status:** Accepted
- **Scope:** Platform 1 (and Platform 2 Gold)
- **Diagram marker:** D13

## Context
The dimensional model sets the overall approach. → [D10](0010-kimball-dimensional-model.md) Several recurring modeling problems still need a standard answer:
- The same business event is recorded by more than one source system.
- Facts collect many low-cardinality flags.
- Some relationships are many-to-many (e.g., a worker assigned to several cost centers or project teams).
- The same dimension plays several roles (e.g., hire date, termination date; worker, manager, recruiter).
- Some fact records arrive before their dimension member exists.

Without standard patterns, each subject area solves these differently, and totals or definitions drift.

## Decision
- **Consolidated facts with a source-type discriminator:** when several systems record the same business event (e.g., an absence recorded by both the HCM system and a time-tracking system), load them into one fact with a `source_type` column instead of one fact per source. Users get one place to query and can still filter by source.
- **Junk dimensions** for low-cardinality flags, so fact tables don't carry many flag columns.
- **Weighted bridge tables** for many-to-many relationships, with allocation weights that add up to 1 to prevent double counting.
- **Role-playing dimensions** (date, worker) instead of copying the dimension for each role.
- **Unknown-member rows** in every dimension, so facts that arrive before their dimension member are loaded with a placeholder key and reconciled later, not dropped. → [D5](0005-blocking-vs-alerting-dq.md)

## Alternatives Considered
| Option | Why Not Chosen |
|---|---|
| One fact per source system | Users must union facts themselves; KPI definitions drift between them. |
| Flags directly on facts | Wide facts, slower scans. |
| Many-to-many without weights | Totals double-count. |
| A separate physical copy of a dimension per role | Duplicate maintenance; copies drift apart. |
| Rejecting facts with unresolved keys | Data loss and gaps; timing issues treated as errors. |

## Consequences
**Positive**
- Simpler consumption: one fact per business event.
- Correct totals across many-to-many relationships.
- Leaner facts and consistent dimension usage.
- No facts lost to timing gaps.

**Negative / Trade-offs**
- Consolidated facts need a common grain and conformed codes across sources.
- Bridge weights must be maintained when memberships change.
- Unknown-member rows must be monitored, or they can hide real data quality issues.

## Guardrails
- A consolidated fact is allowed only when all sources share the same grain; otherwise, keep separate facts.
- The `source_type` values are a controlled list, documented in the catalog.
- Bridge weights are validated in DQ checks: for each worker, the weights add up to 1.
- Role-playing views have clear, role-specific names (e.g., `hire_date`, `manager_worker`).
- Unknown-member counts are tracked; keys that stay unresolved beyond the expected window raise a DQ alert.

## Revisit When
- A new source records the same event at a different grain.
- BI tools can handle many-to-many relationships natively with correct totals.
- Many unknown-member rows persist, which signals a source or timing problem.
- The warehouse moves to a lakehouse; these patterns carry over to Delta tables. → [D10](0010-kimball-dimensional-model.md)

## Related
- [D10: Kimball Dimensional Model](0010-kimball-dimensional-model.md)
- [D5: Blocking vs Alerting DQ](0005-blocking-vs-alerting-dq.md)
- [D4: Federation Over Replication](0004-federation-over-replication.md)
- [Platform 1](../architecture/platform-1-batch-warehouse.md)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*