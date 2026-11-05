# ADR-0001: Two platforms, Not One

- **Status:** Accepted
- **Scope:** Both
- **Diagram marker:** D1

## Context
An established nightly data warehouse is the trusted source for enterprise HR reporting. It is complete, reconciled and governed, and it holds the most sensitive data. New requirements have appeared that it was not built for: event-driven integrations (onboarding, recruiting sync, learning-platform exchange) that need data within hours rather than T+1, plus operational monitoring of those integrations.

## Decision
Keep the warehouse (**Platform 1**) as the producer of conformed data and the system of record for analytics. Add a cloud lakehouse (**Platform 2**) as a consumer for integration-specific workloads. Platform 2 **extends** the warehouse and never duplicates what the warehouse masters.

## Alternatives Considered
| Option | Why Not Chosen |
|---|---|
| Migrate everything to the lakehouse in one program | High risk and a long parallel run. Security, reconciliation and sensitive-data handling would all need re-certifying at once. |
| Add near-real-time loads to the warehouse | Competes with the nightly batch window. Relational ETL is a poor fit for event and API orchestration, and it couples integrations to the batch scheduler. |
| Point-to-point integrations with no analytics layer | No history, no monitoring, and logic duplicated in every integration. |

## Consequences
**Positive**
- Low migration risk: the trusted platform is untouched.
- Each workload runs on the platform best suited to it.
- Failures stay contained: independent schedulers, and a problem in one platform doesn't stop the other.
- Gives an incremental path to modernization instead of a big-bang migration.

**Negative / Trade-offs**
- Two platforms to operate, needing two skill sets. → [R-11](../risks.md)
- Risk of drift: duplicate KPIs or dimension copies if boundaries are not enforced. → [R-03](../risks.md)
- The boundary between the platforms must be explicit and governed.

## Guardrails
- Platform 2 never masters conformed dimensions; it reads them through federation. → [D4](0004-federation-over-replication.md)
- Restricted and highly restricted data stays in Platform 1. → [D11](0011-defense-in-depth-security.md)
- No hard job dependency across platforms. Lakehouse reconciliation and refreshes are *aligned* to the warehouse publish, but a late publish only delays enrichment; integrations keep flowing. → [F4](../failure-modes.md#f4-federation-slow-or-enterprise-warehouse-unavailable)
- Each KPI definition has exactly one owning platform.

## Revisit When
- Most consumers need sub-day freshness.
- The warehouse platform approaches end of support.
- Lakehouse governance is mature enough to hold restricted data, enabling a single governed platform.

One path toward a single governed lakehouse is shown in the [Evolution reference architecture](https://github.com/srimarrivada/Enterprise-Data-Platform-Evolution-Reference-Architecture).

## Related
- [Architecture Overview](../architecture/architecture-overview.md) 
- [Platform 1](../architecture/platform-1-batch-warehouse.md) 
- [Platform 2](../architecture/platform-2-lakehouse.md)
- [Risks](../risks.md)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*