# ADR-0002: Temporal ODS Family

- **Status:** Accepted
- **Scope:** Platform 1
- **Diagram marker:** D2

## Context
HR data is effective-dated. A change entered today can take effect in the **past** (a correction or retroactive change) or in the **future** (a scheduled hire, transfer or termination). A single "current" extract misses back-dated changes, and future-dated rows contaminate current state. Recovering from errors also requires a complete history.

## Decision
Use a **family of operational data stores** with the same schema and shared code, each loading a different time window:

| Store | Window | Role |
|---|---|---|
| **Current** (primary) | Latest changes since the last successful run | Runs all transformations; feeds change detection |
| Effective-dated | Look-back window for back-dated changes | Catches corrections; merged back into Current (insert-only) |
| Future-dated | Forward window | Pre-hire provisioning and planning; bypasses change detection until effective |
| History | All time, append-only snapshots | Point-in-time reconstruction |
| Full | Complete extract, on demand | Recovery and history reloads |

Implementation:
- **Set-based SQL for complex logic:** hierarchy traversal, classification rules and effective-date logic live in set-based SQL (e.g., stored procedures), not visual ETL mappings. They are easier to express, test and tune there. The ETL tool handles data movement and orchestration.
- **One code base:** code is committed once to Current and promoted to every store through CI/CD. Database-aware logic handles the few places where behavior differs by window.

## Alternatives Considered
| Option | Why Not Chosen |
|---|---|
| Single SCD2 staging table with valid-from / valid-to | Complex per-entity merge logic; hard to keep corrections and future changes apart; recovery means rebuilding everything. |
| Daily full snapshots only | Storage-heavy, and a snapshot can't distinguish a back-dated correction from a new change. |
| Rely on the source system's own history | Constrained by API limits and cost; no control over retention. |

## Consequences
**Positive**
- Back-dated corrections are captured without overwriting current state.
- Future-dated changes are available before they take effect.
- Point-in-time rebuild and replay are possible.
- Each time window is isolated, so a problem in one doesn't corrupt the others.

**Negative / Trade-offs**
- More databases to run and monitor.
- Code must be window-aware.
- History stores grow continuously and need a retention or archiving policy. → [R-07](../risks.md)

## Guardrails
- Merge-back is insert-only and idempotent; it never updates rows already in Current. → [D9](0009-idempotent-loads.md)
- Every transformation runs inside a transaction with consistent error handling, so a failure never leaves a partial update.
- Row-count and threshold gates between stores.
- One commit is promoted everywhere, with no store-specific code forks.
- Window sizes and retention periods are configuration, not code.

## Revisit When
- The source system provides a reliable effective-dated change feed.
- The platform moves to a table format with time travel and MERGE (e.g., Delta Lake), which could collapse several stores into fewer tables.

## Related
- [Platform 1](../architecture/platform-1-batch-warehouse.md)
- [D3: Dependency-Tiered Batch](0003-dependency-tiered-batch.md)
- [D8: Change Detection](0008-change-detection-strategy.md)
- [F5: Late or Back-Dated Source Changes](../failure-modes.md#f5-late-or-back-dated-source-changes)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*