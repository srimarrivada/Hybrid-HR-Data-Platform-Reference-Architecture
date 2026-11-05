# Governance and Operations

## Overview

This document explains the governance and operations model across both platforms of the Hybrid HR Data Platform Reference Architecture. Products named (e.g., Power BI, Databricks, Delta Lake) are examples; the patterns apply to equivalent technologies.

Key areas:

- [Data quality](#data-quality)
- [Access control and security tiers](#access-control)
- [Data retention and privacy](#data-retention-and-privacy)
- [Lineage](#lineage)
- [CI/CD](#cicd)
- [Monitoring](#monitoring)
- [Incident management](#incident-management)
- [Backup and recovery](#backup-and-recovery)

---

## Data Quality

The two platforms use different DQ strategies because they serve different audiences. → [D5](../adr/0005-blocking-vs-alerting-dq.md)

### Platform 1: Blocking DQ

Platform 1 supports enterprise-wide BI and sensitive data, so a quality gate **blocks publishing** when quality drops below the threshold. Critical rules block regardless of the overall score.

Typical checks:

- Referential integrity
- Completeness
- Duplicate detection
- SCD2 validation
- Source-to-target reconciliation
- Volume anomalies
- Required-field validation

Blocking DQ is appropriate when:

- Data is consumed by a broad business audience
- Published data becomes the enterprise source of truth
- Incorrect data could impact executive reporting
- Sensitive or regulated HR data is involved
- Downstream dashboards depend on trusted daily publish cycles

If the gate fails, the previous warehouse version stays live. → [F2](../failure-modes.md#f2-data-quality-gate-below-threshold)

### Platform 2: Alert-Oriented DQ

Platform 2 supports integration-specific analytics, so DQ checks are **logged and alerted but don't block** the pipeline.

Typical checks:

- Unresolved worker keys
- Unresolved recruiter keys
- Unresolved session keys
- Freshness / SLA breaches
- Volume checks
- Business rule thresholds

Alert-oriented DQ is appropriate when:

- The workload is integration-specific
- Operational visibility should continue
- Issues can be investigated without stopping every cycle
- Data volumes are smaller
- The dashboard audience is limited or operational
- Some issues are expected temporarily due to timing

### Reconciliation

> Reconciliation should fix timing gaps. It should not hide bad source data.

How it works for an expected timing gap:

1. A new hire record arrives before the warehouse worker dimension is refreshed.
2. The record gets the unknown-member worker key, temporarily.
3. Reconciliation resolves the key after the warehouse publish.
4. Keys that stay unresolved beyond the expected window trigger an alert.

How different cases are treated:

- **New hire, unresolved worker key:** expected; reconcile later.
- **Existing employee, unresolved key:** usually a source issue; treat as DQ.
- **External applicant:** never reconciled to the worker dimension.

→ [Design Principle 11](../design-principles.md#11-reconciliation-fixes-timing-not-bad-data)

---

## Access Control

The architecture uses layered security. → [D11](../adr/0011-defense-in-depth-security.md)

Typical controls:

- Schema-level separation
- Row-level security
- Role-based access control
- Column masking
- Least-privilege access
- Workspace separation
- Audit logging

Sensitive HR data stays in the governed warehouse unless there is a clear business and governance reason to expose it elsewhere. Restricted data is never exposed through federation. → [D4](../adr/0004-federation-over-replication.md)

### Security Tiers

A typical HR analytics platform separates data into three broad tiers.

| Tier | Typical content | Typical audience |
|---|---|---|
| General analytics | Headcount, movement, leave, hiring, termination, training participation | Broad HR and leadership audience |
| Restricted | Demographics, diversity fields, sensitive worker attributes, restricted leadership analytics | Small, approved groups |
| Highly restricted | Compensation, bonus, payroll, national identifiers, legal-sensitive data | Very small, named groups; kept in the source (e.g., DirectQuery) |

---

## Data Retention and Privacy

- Each source and layer has a defined **retention period** (raw capture, ODS history, warehouse history, lakehouse Bronze).
- History growth is planned from day one, with archiving or compaction rather than unbounded growth. → [D2](../adr/0002-temporal-ods-family.md)
- Raw layers that may hold personal data (e.g., Bronze) are restricted to platform engineers. → [D12](../adr/0012-bronze-captures-transient-sources.md)
- A documented **deletion procedure** handles personal-data removal requests across all layers, including history and raw capture.
- Downstream feeds receive only the fields their contract requires (data minimization).

---

## Lineage

Lineage is captured across the full platform.

**Lineage covers:**

- Source systems
- ODS tables
- Curated stores
- Warehouse dimensions and facts
- Lakehouse Bronze / Silver / Gold layers
- Reporting views
- BI datasets

**Typical sources of lineage:**

- Source-to-target mappings
- Schema registry
- Catalog lineage (e.g., Unity Catalog)
- Table history
- ETL audit logs
- DQ history

**Lineage supports:**

- Impact analysis
- Root-cause analysis
- Change governance
- Auditability
- Data trust

---

## CI/CD

Both platforms use controlled, reversible deployment. Dependencies, DQ rules and source configurations are reviewed like code. → [D3](../adr/0003-dependency-tiered-batch.md)

### Platform 1 CI/CD

```text
Source control
    ↓
Build / validation
    ↓
Development
    ↓
Stage / UAT
    ↓
Production
```

Deployment characteristics:

- Planned release cycles, with a hotfix path
- Manual approval gates and production sign-off
- Rollback procedures
- Release notes
- Deployment validation
- One commit promoted to all temporal stores (no store-specific forks) → [D2](../adr/0002-temporal-ods-family.md)

Platform 1 deployment is more tightly controlled because it supports enterprise-wide reporting and sensitive data.

### Platform 2 CI/CD

```text
Git repository
    ↓
Development catalog
    ↓
Stage catalog
    ↓
Production catalog
```

Deployment characteristics:

- Notebook and job promotion between environments
- Environment-specific configuration
- Secret scopes per environment
- Table version rollback where applicable (e.g., Delta)
- Jobs deployed between cycles
- Shorter release cycles, with hotfix support
- Validation using controlled job runs
- Idempotency tests: each load run twice and the results compared → [D9](../adr/0009-idempotent-loads.md)

Platform 2 deploys more often because integration analytics changes faster than the core warehouse.

---

## Monitoring

Monitoring covers both technical failures and data quality failures.

| Area | Platform 1 | Platform 2 |
|---|---|---|
| Jobs | Scheduler job status, workflow success / failure | Job runs, notebook / task failures |
| Logs | Batch audit logs, ETL error logs | API failure logs, file capture success / failure |
| Data quality | DQ score history, publish status | DQ check results, worker key reconciliation status |
| Consumption | BI refresh status, file feed delivery history | SQL compute performance, dashboard freshness |
| Cross-platform | Source volumes, runtime trends | Federation latency and source load |

Early-warning signals from the [risk register](../risks.md) are part of standard monitoring, not separate checks.

---

## Incident Management

Recommended incident flow:

```text
Alert
    ↓
Ops team investigation
    ↓
Log review / DQ review
    ↓
Root cause classification
    ↓
Source fix or pipeline fix
    ↓
Re-run / reconciliation / manual correction
    ↓
Incident closure (and register / ADR update if needed)
```

### Incident Classification

| Category | Examples |
|---|---|
| Source issue | Missing file, bad source value, API outage, schema drift |
| Pipeline issue | Job failure, job timeout, notebook error, stored procedure failure |
| DQ issue | Referential integrity failure, unresolved key |
| BI issue | Refresh failure, dashboard timeout |
| Delivery issue | File delivery failure |
| Security issue | Access denied, incorrect role mapping |
| Federation issue | Slow query, connection failure, source load issue |

For detection, automatic behavior and recovery steps for each major scenario, see [Failure Modes](../failure-modes.md).

---

## Backup and Recovery

Recovery paths are designed before failure happens. → [Design Principle 14](../design-principles.md#14-design-for-recovery)

| Platform 1: restore or republish trusted batch data | Platform 2: replay and table versioning |
|---|---|
| Full / differential / log database backups | Table time travel (e.g., Delta Lake) |
| Prior successful published warehouse version | Restore prior Gold table version |
| Reprocess from ODS / history | Re-run from Bronze |
| Restart from last successful state | Replay source files / events |
| BI cache available between refreshes | Re-run a failed job cycle or a specific integration window |

Recovery is tested through scheduled restore and replay drills. → [NFR-09](../nfrs.md)

For specific failure scenarios and recovery steps, see [Failure Modes](../failure-modes.md).

---

## Operational Design Principles

1. DQ should be visible and actionable.
2. Blocking DQ is used only where the cost of publishing bad data is high.
3. Alerting DQ is acceptable for smaller operational lakehouse workloads.
4. Logs should support root-cause analysis.
5. Recovery is designed before failure happens.
6. Source data is preserved where replay is required.
7. Security is enforced at multiple layers.
8. Deployment is controlled and reversible.
9. Monitoring covers both technical failures and data quality failures.
10. Federation is monitored for latency and source-system impact.

---

## Related Documents

- [Architecture Overview](architecture-overview.md)
- [Federation Pattern](federation-pattern.md)
- [Consumption Pattern](consumption-pattern.md)
- [Failure Modes](../failure-modes.md)
- [Risks](../risks.md)
- [Non-Functional Requirements](../nfrs.md)
- [ADR Index](../adr/README.md)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*