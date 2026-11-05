# ADR-0003: Dependency-Tiered Batch Orchestration

- **Status:** Accepted
- **Scope:** Platform 1
- **Diagram marker:** D3

## Context
The nightly pipeline contains many jobs across extraction, transformation, change detection, downstream feeds and warehouse loading. Fixed clock schedules either waste time through padding or fail silently when upstream jobs run late. A strictly sequential chain makes the critical path the sum of every job's duration.

## Decision
Orchestrate by **dependency graph**:
- A small number of anchor jobs start on a clock; every other job starts when its predecessors complete.
- Jobs are grouped into tiers, and independent jobs within a tier run in parallel.
- **Milestone gates** wait for a defined set of predecessors before a dependent stage starts.
- The graph is defined as **configuration** (e.g., a dependency list), not hard-coded.
- State advances only after full success: a post-load step moves the extraction window forward. → [D9](0009-idempotent-loads.md)
- **Warehouse load order follows tiers:** reference / utility → independent dimensions → core dimensions → dependent dimensions → bridges → facts → post-load (aggregates, audit). A failure in a core dimension holds back everything that references it.

## Alternatives Considered
| Option | Why Not Chosen |
|---|---|
| Fixed clock time per job | Needs padding for worst-case durations, and breaks when upstream runs late. |
| Single sequential chain | Simple, but gives the longest possible critical path. |
| Fully event-driven orchestration | Overkill for a nightly batch, and harder to audit as one nightly run. |

## Consequences
**Positive**
- Shorter critical path, because independent work runs in parallel.
- Completion time adapts to daily change volume instead of a fixed worst case.
- A failure automatically holds back its dependents, so no partial loads flow downstream.
- Runs can restart from the failed step. → [F1](../failure-modes.md#f1-nightly-run-fails-part-way)
- A failed night re-runs the same extraction window, so there are no data gaps.

**Negative / Trade-offs**
- The dependency graph must be maintained as the platform changes.
- A missing dependency creates a race condition. → [R-08](../risks.md)
- Parallelism raises load on sources and targets, so concurrency must be limited.

## Guardrails
- Dependency changes are reviewed like code, and checked in CI (every job's inputs have a declared predecessor).
- Concurrency caps respect source API connection limits.
- Pause / stop / retry / resume flags and a run audit for every batch.
- Critical-path duration is monitored, so slow jobs are found before they threaten the publish window.

## Revisit When
- Moving to a cloud orchestrator (e.g., Airflow or Databricks Workflows). The same model maps directly to DAGs.
- The graph becomes too large to reason about, which is a signal to split it into independently scheduled domains.

## Related
- [Platform 1](../architecture/platform-1-batch-warehouse.md)
- [Governance and Operations](../architecture/governance-and-operations.md)
- [D9: Idempotent Loads](0009-idempotent-loads.md)
- [F1: Nightly Run Fails Part-Way](../failure-modes.md#f1-nightly-run-fails-part-way)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*