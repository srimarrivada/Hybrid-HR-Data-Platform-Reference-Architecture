# ADR-0008: Change Detection by Content Compare

- **Status:** Accepted
- **Scope:** Platform 1 (pattern reusable in Platform 2)
- **Diagram marker:** D8

## Context
Source extracts carry load stamps and other volatile columns that change on every run even when the business data hasn't. Timestamp-based change detection therefore reports false changes, and it misses real ones when source timestamps are unreliable. Downstream feeds and history need to see **only true changes**.

## Decision
Detect changes by **comparing content**, not timestamps:
- **Set-based compare** (e.g., `EXCEPT` between staging and the curated store) for moderate-width entities.
- **Row hash** (a checksum over the business columns) for wide dimensions, so a single comparison replaces dozens of column checks.
- **Volatile columns are always excluded** (load stamps, extraction dates).
- **Changed keys are routed:**
  - New key → insert.
  - Existing key, correction-type attribute → update in place (SCD Type 1).
  - Existing key, history-tracked attribute → close the current version and insert a new one (SCD Type 2). → [D10](0010-kimball-dimensional-model.md)
- **Deletes are detected explicitly:** keys present in the curated store but missing from a full extract are flagged as inactive (soft delete), never physically removed, so history stays intact.

Back-dated corrections from the effective-dated store pass through the same compare, so they are handled like any other change. → [D2](0002-temporal-ods-family.md), [F5](../failure-modes.md#f5-late-or-back-dated-source-changes)

## Alternatives Considered
| Option | Why Not Chosen |
|---|---|
| Source last-modified timestamps | Unreliable; volatile columns cause false positives. |
| Full reload every night | Simple but expensive, and it loses the "what changed" signal downstream consumers need. |
| Database-native CDC on the source | Not available for SaaS sources accessed through APIs. |

## Consequences
**Positive**
- Accurate change sets.
- Downstream feeds receive only real changes.
- History stays clean and auditable. → [NFR-10](../nfrs.md)

**Negative / Trade-offs**
- Compare cost grows with table width and volume.
- The list of excluded columns must be maintained.
- Delete detection needs a full extract (or a source delete signal) for the entity.

## Guardrails
- The excluded-column list is configuration, reviewed when the schema changes.
- Hash algorithm, column order and null handling are fixed and documented. Changing any of them makes every row look changed.
- Use a hash with a negligible collision risk for the data volume; compare full columns when a hash matches but correctness is critical.
- Row counts are reconciled before and after the compare.
- The compare is idempotent: running it twice on the same input produces no new changes. → [D9](0009-idempotent-loads.md)

## Revisit When
- The platform moves to a table format with native MERGE and change data feed (e.g., Delta Lake).
- Sources start providing a reliable change or delete feed.

## Related
- [Platform 1](../architecture/platform-1-batch-warehouse.md)
- [D2: Temporal ODS Family](0002-temporal-ods-family.md)
- [D10: Kimball Dimensional Model](0010-kimball-dimensional-model.md)
- [Design Principle 8](../design-principles.md#8-handle-time-explicitly)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*