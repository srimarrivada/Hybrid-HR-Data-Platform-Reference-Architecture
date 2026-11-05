# ADR-0005: Blocking vs Alerting Data Quality

- **Status:** Accepted
- **Scope:** Both
- **Diagram marker:** D5

## Context
The two platforms serve consumers with different contracts:
- **Enterprise reporting** has a broad audience making decisions from it. Wrong numbers are worse than slightly stale ones.
- **Operational integrations** feed downstream processes that cannot wait. Some unresolved keys are *expected* because of timing; for example, a new hire may not reach the warehouse until the next nightly load.

## Decision
- **Platform 1: blocking gate before publish.** A rule set produces a quality score. Below the threshold, or if any critical rule fails, publishing stops, the previous version stays live, and an alert raises a ticket. → [F2](../failure-modes.md#f2-data-quality-gate-below-threshold)
- **Platform 2: alerting checks.** Checks run every cycle and write to a shared DQ log. They alert but do not block, and a scheduled reconciliation resolves timing gaps after the warehouse publish.
- **Reconciliation follows business meaning, not just lookup failure:**
  - An unresolved new hire is expected; it gets the unknown-member key and is reconciled later. → [D13](0013-fact-design-patterns.md)
  - An external candidate has no worker record; no reconciliation is needed.
  - An unresolved learner should already exist, so it raises a DQ alert.
  - Any key that stays unresolved beyond the expected window is investigated as a source issue, not reconciled away.

## Alternatives Considered
| Option | Why Not Chosen |
|---|---|
| Block everywhere | Integrations would stall on expected timing gaps. |
| Alert everywhere | Bad data could reach executive dashboards. |
| Quarantine failing records and publish the rest (for the warehouse) | Partial data gives misleading totals in enterprise reporting; a complete previous version is safer than an incomplete new one. |

## Consequences
**Positive**
- Each audience gets the failure mode that suits it.
- Warehouse numbers stay trustworthy, and integrations keep flowing.

**Negative / Trade-offs**
- Thresholds need tuning over time.
- Dashboards showing yesterday's data need a visible freshness indicator.
- Too many alerts lead to alert fatigue. → [R-10](../risks.md)

## Guardrails
- Critical rules block regardless of the overall score.
- Severity routing: critical issues create tickets, warnings go to a channel.
- A DQ scorecard makes results visible.
- Dashboards always show how current their data is.
- Rules and thresholds are configuration, reviewed when they fire too often or not at all.

## Revisit When
- Consumers need finer control, e.g., publishing per subject area instead of all-or-nothing.
- Alert volume grows to the point where warnings are routinely ignored.

## Related
- [Governance and Operations](../architecture/governance-and-operations.md)
- [Design Principle 11](../design-principles.md#11-reconciliation-fixes-timing-not-bad-data)
- [D4: Federation Over Replication](0004-federation-over-replication.md)
- [F2: DQ Gate Below Threshold](../failure-modes.md#f2-data-quality-gate-below-threshold) 
- [F3: Integration Record Fails](../failure-modes.md#f3-integration-record-or-api-call-fails)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*