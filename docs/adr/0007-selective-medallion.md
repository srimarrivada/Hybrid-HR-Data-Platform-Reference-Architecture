# ADR-0007: Medallion Only Where It Adds Value

- **Status:** Accepted
- **Scope:** Platform 2
- **Diagram marker:** D7

## Context
Platform 2 runs several integrations. The default "medallion for everything" approach would build Bronze, Silver and Gold layers for each one, whether or not anyone uses the result.

## Decision
An integration gets a Gold layer only if **all** of these hold:
1. It builds history that is not kept anywhere else.
2. It has a real analytical audience.
3. It answers questions no existing source can.
4. The value justifies the build and run effort.

Applying this, most integrations **build**. A system-to-system contact sync is a deliberate **skip**:
- It keeps current state only.
- Its data already exists in the warehouse worker dimension.
- It has no analytical audience.

The skipped integration is still covered by job monitoring, run logs, status tracking and a reprocess job. → [D9](0009-idempotent-loads.md)

Where an integration does build, the model is shaped to its questions:
- **Step dimension** (ordered stages) for conversion funnels and time-in-stage.
- **Dual-key lookup** when different source systems identify the same business entity differently.
- **Phased scope:** deliver what the integration can support now, and record deferred scope explicitly (e.g., add outcome analytics once that data is available).

### Example Decision Matrix

| Integration | Builds history? | Analytical audience? | Unique questions? | Worth the effort? | Result |
|---|---|---|---|---|---|
| New hire events | Yes | Yes (operations) | Yes | Yes | **Build** |
| Recruiting pipeline | Yes (source is transient) | Yes | Yes | Yes | **Build** |
| Training enrollment | Yes | Yes | Yes | Yes | **Build** (phased) |
| Contact data sync | No (current state only) | No | No (in warehouse) | No | **Skip** (monitor only) |

## Alternatives Considered
| Option | Why Not Chosen |
|---|---|
| Medallion for every integration | Cost, unused models and extra objects to govern. |
| Bronze + Silver only, no Gold | Still adds build and run cost; doesn't answer analytical questions. Useful only as a staging step toward Gold. |
| No analytical layer at all | Loses history and the questions only this data can answer. |

## Consequences
**Positive**
- Effort goes where it creates value.
- Fewer objects to maintain and govern.
- The decision and its reasoning are recorded, not implicit.

**Negative / Trade-offs**
- Skipped integrations have no analytics until reconsidered.
- The decision needs periodic review as audiences change.

## Guardrails
- A decision matrix is kept for each integration and reviewed when its use changes.
- **Skip does not mean unmonitored:** operational monitoring is always required. → [F3](../failure-modes.md#f3-integration-record-or-api-call-fails)
- Transient sources are captured in Bronze on arrival even if no Gold layer is built, so a later build can still use their history. → [D12](0012-bronze-captures-transient-sources.md)

## Revisit When
- A new audience or question appears (e.g., delivery success analytics).
- Deferred scope becomes possible, such as outcome data becoming available.

## Related
- [Platform 2](../architecture/platform-2-lakehouse.md)
- [Design Principle 5](../design-principles.md#5-model-only-where-analytical-value-exists)
- [Source Assessment](../source-assessment.md)
- [D1: Two platforms, Not One](0001-two-platforms-not-one.md)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*