# ADR-0009: Idempotent Loads and Safe Re-Runs

- **Status:** Accepted
- **Scope:** Both
- **Diagram marker:** D9

## Context
Jobs fail and get re-run: nightly batch steps in Platform 1, and integration cycles and outbound calls in Platform 2. Sources also deliver the same file twice, and retries resend the same record. If a re-run duplicates or double-counts data, operations has to decide by hand whether re-running is safe, and recovery slows down.

## Decision
Every load must be **safe to re-run**: running it twice on the same input gives the same result as running it once.

**Both platforms**
- **State advances only after full success.** Watermarks, versions and publishes move forward only when a run completes, so a failed run automatically re-processes the same window. → [D3](0003-dependency-tiered-batch.md)
- **Upserts and MERGE** rather than blind inserts.
- **File / run logs** record every processed input, so a file delivered twice is rejected.
- **Additive merges** (target += source, e.g., running totals) are allowed only when a run log rejects inputs that were already processed.
- **Error thresholds** stop a run early instead of loading partial data; every batch and error is logged.

**Platform 2 additionally**
- Every integration record carries a status (**pending → success / failed**), so a re-run only processes what hasn't succeeded.
- Outbound calls carry an **idempotency key**, so retries can't create duplicates downstream.
- Raw inputs are kept in Bronze, so any window can be replayed without re-requesting it from the source. → [D12](0012-bronze-captures-transient-sources.md)

## Alternatives Considered
| Option | Why Not Chosen |
|---|---|
| Manual cleanup before re-runs | Slow, error-prone, depends on individuals. |
| Truncate-and-reload everything | Safe but expensive, and loses incremental history. |
| Rely on exactly-once delivery from the infrastructure | Rarely guaranteed end to end across files, APIs and downstream systems; the pipeline must protect itself. |

## Consequences
**Positive**
- Any job can be re-run without analysis.
- No data gaps after failures, and no double counting after retries. → [NFR-03](../nfrs.md)
- Faster, routine recovery. → [F1](../failure-modes.md#f1-nightly-run-fails-part-way), [F3](../failure-modes.md#f3-integration-record-or-api-call-fails)

**Negative / Trade-offs**
- Run logs, status tables and watermark state must be maintained.
- Additive patterns need extra design.
- Idempotency keys must be supported (or emulated) by downstream systems.

## Guardrails
- Code review asks of every new load: "What happens if this runs twice?"
- An idempotency test in CI for each pipeline: run twice, compare results.
- Change detection is itself idempotent: a second compare on the same input finds no new changes. → [D8](0008-change-detection-strategy.md)
- Any additive merge without a run log is rejected in review.

## Revisit When
- The platform moves to tooling with built-in exactly-once guarantees end to end (e.g., transactional table formats with checkpointed streaming), which could simplify run logs.
- A downstream system can't accept idempotency keys, which needs a compensating design.

## Related
- [Design Principle 9](../design-principles.md#9-every-load-is-safe-to-re-run)
- [D3: Dependency-Tiered Batch](0003-dependency-tiered-batch.md)
- [D8: Change Detection](0008-change-detection-strategy.md)
- [Failure Modes](../failure-modes.md)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*