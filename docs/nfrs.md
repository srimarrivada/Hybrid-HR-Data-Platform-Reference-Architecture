# Non-Functional Requirements

These requirements describe how well the platform must work, not what it does. Each one states how it is verified and which decision addresses it. Targets are expressed qualitatively; an implementation would set specific values for its own context.

| ID | Category | Requirement | How it's verified | Addressed by |
|---|---|---|---|---|
| NFR-01 | Freshness | Enterprise data current by business start; operational data within a few hours | Freshness label on every dashboard; publish-time monitoring against the agreed window | [D3](adr/0003-dependency-tiered-batch.md), [D6](adr/0006-bi-tool-by-audience.md) |
| NFR-02 | Capture | Transient source data captured before it is deleted, overwritten or encrypted | File-arrival checks; Bronze row counts reconciled to source deliveries | [D12](adr/0012-bronze-captures-transient-sources.md) |
| NFR-03 | Reliability | Any load is safe to re-run; no data gaps after failures | Idempotency test in CI (run twice, compare); watermark audit after failed runs | [D9](adr/0009-idempotent-loads.md) |
| NFR-04 | Quality | Enterprise publish blocked below the quality threshold; critical rules always block | DQ score history; publish log shows no publish below threshold | [D5](adr/0005-blocking-vs-alerting-dq.md) |
| NFR-05 | Security | Sensitive data tiered, minimized and audited; no restricted data through federation | Periodic access reviews; audit log of restricted reads; federated view column review | [D11](adr/0011-defense-in-depth-security.md) |
| NFR-06 | Performance | Interactive dashboards respond in seconds; trend visuals read aggregates, not detail | Dashboard query-time monitoring; aggregate hit rate in BI | [D6](adr/0006-bi-tool-by-audience.md), [D13](adr/0013-fact-design-patterns.md) |
| NFR-07 | Consistency | One owning platform per KPI; conformed dimensions read, not copied | KPI catalog with owners; no conformed dimension tables in the lakehouse | [D1](adr/0001-two-platforms-not-one.md), [D4](adr/0004-federation-over-replication.md) |
| NFR-08 | Operability | Every job monitored, logged and restartable from the failed step | Run audit for every batch; restart tested as part of release | [D3](adr/0003-dependency-tiered-batch.md), [Governance & operations](architecture/governance-and-operations.md) |
| NFR-09 | Recoverability | Replay from raw capture; documented backup and restore | Scheduled restore and replay drills | [Failure modes](failure-modes.md) |
| NFR-10 | Auditability | Lineage from source to dashboard; every change to history is traceable | Catalog lineage review; SCD2 history checks | [D8](adr/0008-change-detection-strategy.md), [D10](adr/0010-kimball-dimensional-model.md) |
| NFR-11 | Privacy & retention | Raw and historical data kept only as long as justified; personal data deletable on request | Retention policy per source and layer; deletion procedure tested | [D2](adr/0002-temporal-ods-family.md), [D12](adr/0012-bronze-captures-transient-sources.md) |
| NFR-12 | Maintainability | New sources, rules and dependencies added through configuration, not code changes | Onboarding a test source needs config only; changes reviewed like code | [D3](adr/0003-dependency-tiered-batch.md), [D5](adr/0005-blocking-vs-alerting-dq.md) |
| NFR-13 | Scalability | Workforce growth or new integrations don't require redesign | Runtime and query trends tracked over time; capacity reviewed when trends rise | [D3](adr/0003-dependency-tiered-batch.md), [D7](adr/0007-selective-medallion.md) |

## Related
- [Business Context](business-context.md)
- [Source Assessment](source-assessment.md)
- [Risks](risks.md)
- [ADR Index](adr/README.md)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*