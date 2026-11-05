# Architecture Risks and Mitigations

The main risks this architecture carries, how likely and serious each one is, how it would be noticed early, and how it is mitigated. Ratings are qualitative (H / M / L) and would be re-assessed for any real implementation.

| ID | Risk | Likelihood | Impact | Early warning | Mitigation | See also |
|---|---|---|---|---|---|---|
| R-01 | **Federation dependency:** Lakehouse relies on enterprise warehouse availability and performance | M | M | Rising federation latency; more unresolved worker keys | Keep last good Gold; unknown-member keys + reconciliation; resolve keys at ETL time; materialize hot views if needed | [D4](adr/0004-federation-over-replication.md), [F4](failure-modes.md#f4-federation-slow-or-enterprise-warehouse-unavailable) |
| R-02 | **Transient sources lose data before capture** | M | H | Missing or late file arrivals; Bronze counts below source deliveries | Capture on arrival; alert on missing files; replay from Bronze | [D12](adr/0012-bronze-captures-transient-sources.md) |
| R-03 | **KPI drift between platforms:** the same measure defined differently | M | H | Two dashboards showing different numbers for the same question | One owning platform per KPI; logic in semantic models and Gold views, not dashboards | [D1](adr/0001-two-platforms-not-one.md), [D6](adr/0006-bi-tool-by-audience.md) |
| R-04 | **Sensitive data exposed through federation or Bronze** | L | H | Federated view column changes; access to raw layers outside the platform team | Approved views only; restricted data never federated; Bronze restricted to engineers; column reviews | [D4](adr/0004-federation-over-replication.md), [D11](adr/0011-defense-in-depth-security.md) |
| R-05 | **Access model on the newer platform lags behind the warehouse** | M | H | Grants created ad hoc; users requesting access outside the process | Define grants before data lands; same role model on both platforms; regular access reviews | [D11](adr/0011-defense-in-depth-security.md) |
| R-06 | **Shared compute contention** during refresh windows | M | M | Refresh durations creeping up; queued queries | Stagger refreshes, triggered by upstream readiness; separate compute for ops vs BI | [D6](adr/0006-bi-tool-by-audience.md) |
| R-07 | **Unbounded history growth** | H | M | Storage and runtime trends rising steadily | Retention and archiving policy from day one; periodic history compaction | [D2](adr/0002-temporal-ods-family.md) |
| R-08 | **Dependency graph drift:** a new job added without its real dependencies | M | M | Intermittent failures or wrong data only on some nights | Dependencies reviewed like code; dependency checks in CI | [D3](adr/0003-dependency-tiered-batch.md), [F1](failure-modes.md#f1-nightly-run-fails-part-way) |
| R-09 | **Source schema drift** | M | M | Schema validation failures in Bronze / Silver | Schema checks at capture; load stops for that source only; replay after mapping update | [D12](adr/0012-bronze-captures-transient-sources.md) |
| R-10 | **Alert fatigue:** too many DQ warnings, so real issues are missed | M | M | Rising alert volume; alerts closed without action | Severity routing; tune thresholds; alert on trends, not single events | [D5](adr/0005-blocking-vs-alerting-dq.md) |
| R-11 | **Two platforms to run:** higher operating cost and skills spread across both | H | M | Incidents or releases bottlenecked on a few people | Shared standards, runbooks and CI/CD; cross-training; clear platform ownership | [D1](adr/0001-two-platforms-not-one.md) |
| R-12 | **Warehouse platform nears end of support** | L | H | Vendor support timelines; rising patching effort | Track the "revisit when" signals; plan a phased move toward a single governed lakehouse | [D1](adr/0001-two-platforms-not-one.md) |

## How Risks Are Managed
- Each risk has an **owner** and is reviewed at regular architecture reviews.
- **Early-warning signals** are part of standard monitoring, not separate checks.
- A risk that occurs becomes an incident; its root cause updates this register and, where needed, an ADR.

## Related
- [Failure Modes](failure-modes.md)
- [Non-Functional Requirements](nfrs.md)
- [ADR Index](adr/README.md)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*