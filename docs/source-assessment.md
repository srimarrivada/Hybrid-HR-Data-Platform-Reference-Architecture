# Source Assessment Framework

Every source is assessed before design. The answers drive capture, cadence, platform placement and modeling decisions. A source that keeps no history, for example, must be captured on arrival, or its history is lost for good.

## Source Categories

| Source Category | Access | Keeps History? | Capture Urgency | Person Identifier | Sensitivity | Platform | Design Consequence |
|---|---|---|---|---|---|---|---|
| HCM system of record | API | Yes (effective-dated) | Low | Worker ID (master) | High | 1 | Temporal ODS family → [D2](adr/0002-temporal-ods-family.md) |
| Payroll / benefits providers | Files (MFT) | Provider-side only | Medium | Worker ID or provider ID mapped to worker | High (restricted) | 1 | Domain ODS + file run log → [D9](adr/0009-idempotent-loads.md), [D11](adr/0011-defense-in-depth-security.md) |
| Recruiting platform | Files / API | **No** (transient exports) | **High** | Candidate ID; worker ID after hire | Medium | 2 | Capture to Bronze on arrival → [D12](adr/0012-bronze-captures-transient-sources.md) |
| Learning platform | Files / API | Partial | High | Learner ID matched to worker ID | Low | 2 | Bronze capture; phased scope → [D7](adr/0007-selective-medallion.md), [D12](adr/0012-bronze-captures-transient-sources.md) |
| Event platform | API | Partial | Medium | Registrant ID / email matched to worker | Low | 2 | Integration with status tracking and replay → [D9](adr/0009-idempotent-loads.md) |
| External contact vendor | Files | No | Low | Worker ID | Medium | 2 | Integration only, no Gold layer → [D7](adr/0007-selective-medallion.md) |

### Capture Urgency
- **High:** The source deletes or overwrites data, or exports are transient. Capture on arrival.
- **Medium:** History exists but outside our control (provider-side, limited retention). Capture each cycle and keep a run log.
- **Low:** The source keeps full, queryable history. It can be re-extracted on demand.

## Assessment Checklist

| # | Question | What the Answer Drives |
|---|---|---|
| 1 | Does the source keep history, and for how long? | Capture urgency; Bronze retention → [D12](adr/0012-bronze-captures-transient-sources.md) |
| 2 | Are records effective-dated (back-dated or future-dated changes)? | Temporal handling → [D2](adr/0002-temporal-ods-family.md) |
| 3 | How are changes identified (timestamps, full extracts, events)? | Change detection approach → [D8](adr/0008-change-detection-strategy.md) |
| 4 | What identifies a person, and how does it match the worker record? | Key resolution, unknown-member handling, reconciliation → [D4](adr/0004-federation-over-replication.md), [D5](adr/0005-blocking-vs-alerting-dq.md) |
| 5 | How sensitive is the data? | Platform placement; security tier; minimization → [D11](adr/0011-defense-in-depth-security.md) |
| 6 | What volume and cadence, and is delivery file, API or event? | Batch vs micro-batch; platform choice → [D1](adr/0001-two-platforms-not-one.md) |
| 7 | Who would analyze it, and what questions can only it answer? | Whether to build a Gold layer → [D7](adr/0007-selective-medallion.md) |
| 8 | What happens if a delivery is missing, late or duplicated? | Arrival checks; idempotency; replay → [D9](adr/0009-idempotent-loads.md), [Failure modes](failure-modes.md) |

## Related
- [Business Context](business-context.md)
- [Non-Functional Requirements](nfrs.md)
- [Platform 1](architecture/platform-1-batch-warehouse.md) 
- [Platform 2](architecture/platform-2-lakehouse.md)
- [ADR Index](adr/README.md)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*