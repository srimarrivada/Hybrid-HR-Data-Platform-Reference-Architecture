# ADR-0012: Bronze Captures Transient Source Data

- **Status:** Accepted
- **Scope:** Platform 2
- **Diagram marker:** D12

## Context
Some integrations exchange data that **doesn't persist anywhere**: files that are processed, encrypted and forwarded, or source records that are overwritten or expire. Without capture, history, trend analysis and replay are impossible, and the loss is permanent.

## Decision
Capture every inbound payload **on arrival** in Bronze, *before* transformation, encryption or forwarding:
- **Append-only**, with load time, source identifier and file name.
- **As received**, except for fields classified as restricted. Those are removed at capture, or the source is handled by Platform 1 instead, so restricted data never lands in the lakehouse. → [D11](0011-defense-in-depth-security.md)
- **Arrival checks** per source and schedule, with an alert when an expected delivery is missing or late. → [NFR-02](../nfrs.md)
- **File / run log**, so a payload delivered twice is captured once. → [D9](0009-idempotent-loads.md)
- **Schema validation** at capture; a schema change stops processing for that source only, and the raw payload is kept for replay after the mapping is updated.

Bronze capture applies even to integrations that get no Gold layer, so a later build can still use their history. → [D7](0007-selective-medallion.md)

## Alternatives Considered
| Option | Why Not Chosen |
|---|---|
| Process and forward with no capture | History and trends are lost permanently; failures can't be replayed. |
| Capture only after cleaning (Silver first) | A cleaning bug or rule change can't be corrected, because the original payload is gone. |
| Ask the source to resend when needed | Many sources can't or won't; transient exports may no longer exist. |

## Consequences
**Positive**
- Enables history the source doesn't keep.
- Supports replay, audit and source reconstruction. → [F3](../failure-modes.md#f3-integration-record-or-api-call-fails)
- Makes analytics possible for integrations that were "pass-through" only.

**Negative / Trade-offs**
- Retention and storage cost.
- Raw personal data in Bronze must be governed: access, retention and deletion. → [NFR-11](../nfrs.md)
- Field classification must happen before capture starts.

## Guardrails
- Bronze is restricted to platform engineers.
- Bronze is encrypted at rest, like every other layer.
- A retention period is set per source and enforced automatically.
- Every source's fields are classified before capture begins; restricted fields are never captured in Platform 2.
- The personal-data deletion procedure covers Bronze, not just modeled layers.
- Missing or late deliveries alert, rather than failing silently. → [R-02](../risks.md)

## Revisit When
- A source starts keeping reliable, queryable history of its own (capture urgency drops).
- Storage or retention cost grows faster than the value of the history.
- Regulations change what may be held in raw form.

## Related
- [Design Principle 7](../design-principles.md#7-capture-what-the-source-wont-keep)
- [Platform 2](../architecture/platform-2-lakehouse.md)
- [Source Assessment](../source-assessment.md)
- [D7: Medallion only where it adds value](0007-selective-medallion.md)
- [Risks](../risks.md)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*