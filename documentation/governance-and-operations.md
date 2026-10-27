# Governance and Operations

## Overview

This document explains the governance and operations model for the hybrid HR data platform reference architecture.

The architecture includes governance controls across both the batch warehouse and cloud lakehouse platforms.

Key areas:

- Data quality
- Access control
- Lineage
- CI/CD
- Monitoring
- Incident management
- Backup and recovery
- Security

---

## Data Quality

The two platforms use different DQ strategies based on their roles in the architecture.

---

## Platform 1: Blocking DQ

Platform 1 supports enterprise-wide BI and sensitive data, so DQ rules may block publishing when quality drops below threshold.

Typical checks:

- Referential integrity
- Completeness
- Duplicate detection
- SCD2 validation
- Source-to-target reconciliation
- Volume anomalies
- Required-field validation

### Why Blocking DQ Is Used

Blocking DQ is appropriate when:

- Data is consumed by a broad business audience
- Published data becomes the enterprise source of truth
- Incorrect data could impact executive reporting
- Sensitive or regulated HR data is involved
- Downstream dashboards depend on trusted daily publish cycles

In this pattern, the warehouse is published only after quality checks pass.

---

## Platform 2: Alert-Oriented DQ

Platform 2 supports integration-specific analytics, so DQ checks are usually alert-oriented and non-blocking.

Typical checks:

- Unresolved worker keys
- Unresolved recruiter keys
- Unresolved session keys
- SLA breaches
- Volume checks
- Business rule thresholds

### Why Alert-Oriented DQ Is Used

Alert-oriented DQ is appropriate when:

- The workload is integration-specific
- Operational visibility should continue
- Issues can be investigated without stopping every cycle
- Data volumes are smaller
- The dashboard audience is limited or operational
- Some issues are expected temporarily due to timing

DQ results are logged and alerted, but the pipeline is not automatically blocked for every issue.

---

## Reconciliation

Reconciliation is used only when unresolved keys are expected due to timing.

Example:

1. New hire record arrives before worker dimension refresh.
2. Unknown worker key is allowed temporarily.
3. Reconciliation resolves the key after the warehouse refresh.
4. Long-unresolved keys trigger an alert.

This avoids masking true data quality issues.

### Reconciliation Principle

Reconciliation should fix timing gaps.

It should not hide bad source data.

Examples:

- New Hire unresolved worker keys may be expected temporarily.
- Existing employee unresolved keys usually indicate a source issue.
- External applicants should not be reconciled to worker dimensions.

---

## Access Control

The reference architecture uses layered security.

Typical controls:

- Schema-level separation
- Row-level security
- Role-based access control
- Column masking
- Least-privilege access
- Workspace separation
- Audit logging

Sensitive HR data should remain in the governed warehouse unless there is a clear business and governance reason to expose it elsewhere.

---

## Security Tiers

A typical HR analytics platform separates data into three broad tiers.

### General Analytics

Used for broadly available HR analytics such as:

- Headcount
- Movement
- Leave
- Hiring
- Termination
- Training participation

### Restricted Data

Used for sensitive HR attributes such as:

- Demographics
- Diversity fields
- Sensitive worker attributes
- Restricted leadership analytics

### Highly Restricted Data

Used for highly sensitive subject areas such as:

- Compensation
- Bonus
- Payroll
- National identifiers
- Legal-sensitive data

---

## Lineage

Lineage should be captured across the full platform.

Lineage areas include:

- Source systems
- ODS tables
- Curated stores
- Warehouse dimensions / facts
- Lakehouse Bronze / Silver / Gold layers
- Reporting views
- BI datasets

Representative lineage tools:

- Source-to-target mapping
- Schema registry
- Catalog lineage
- Table history
- ETL audit logs
- DQ history

Lineage supports:

- Impact analysis
- Root-cause analysis
- Change governance
- Auditability
- Data trust

---

## CI/CD

Both platforms should support controlled deployment.

---

## Platform 1 CI/CD

Typical pattern:

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

- Monthly or planned release cycles
- Hotfix path
- Manual approval gates
- Rollback procedures
- Release notes
- Deployment validation
- Production sign-off

Platform 1 deployment is usually more controlled because it supports enterprise-wide reporting and sensitive data.

---

## Platform 2 CI/CD

Typical pattern:

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

- Notebook promotion
- Environment-specific configuration
- Secret scopes per environment
- Delta rollback where applicable
- Jobs deployed between cycles
- Shorter release cycles
- Hotfix support
- Validation using controlled job runs

Platform 2 deployment may be more frequent because integration analytics often changes faster than the core warehouse.

---

## Monitoring

Monitoring should cover:

- Job status
- Runtime duration
- Failed tasks
- Error logs
- DQ checks
- SLA status
- BI refresh status
- Source volume
- Federation latency
- Dashboard freshness

---

## Monitoring by Platform

### Platform 1 Monitoring

Typical monitoring includes:

- Scheduler job status
- Workflow success/failure
- Batch audit logs
- ETL error logs
- DQ score history
- Publish status
- Power BI refresh status
- File feed delivery history

### Platform 2 Monitoring

Typical monitoring includes:

- Databricks job runs
- Notebook failures
- DQ check results
- Worker key reconciliation status
- File capture success/failure
- API failure logs
- SQL Warehouse performance
- Dashboard freshness

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
Incident closure
```

Common incident categories:

- Source feed missing
- API failure
- Schema drift
- DQ failure
- Unresolved worker key
- BI refresh failure
- Federation performance issue
- File delivery failure
- Job timeout
- Security/access issue

---

## Incident Classification

Typical categories include:

| Category | Example |
|---|---|
| Source issue | Missing file, bad source value, API outage |
| Pipeline issue | Job failure, notebook error, stored procedure failure |
| DQ issue | Referential integrity failure, unresolved key |
| BI issue | Refresh failure, dashboard timeout |
| Security issue | Access denied, incorrect role mapping |
| Federation issue | Slow query, connection failure, source load issue |

---

## Backup and Recovery

Both platforms should have clear recovery patterns.

---

## Platform 1 Recovery

Typical recovery methods:

- SQL backups
- Full / differential / log backups
- Prior successful published warehouse state
- Reprocessing from ODS/history
- BI cache availability between refreshes
- Restart from last successful state

Platform 1 recovery is usually designed around restoring or republishing trusted batch data.

---

## Platform 2 Recovery

Typical recovery methods:

- Delta Lake time travel
- Re-run from Bronze
- Replay source files/events
- Restore prior Gold table version
- Re-run failed job cycle
- Reprocess a specific integration window

Platform 2 recovery is usually designed around replayability and lakehouse table versioning.

---

## Operational Design Principles

1. DQ should be visible and actionable.
2. Blocking DQ should be used only where the cost of publishing bad data is high.
3. Alerting DQ is acceptable for smaller operational lakehouse workloads.
4. Logs should support root-cause analysis.
5. Recovery should be designed before failure happens.
6. Source data should be preserved where replay is required.
7. Security should be enforced at multiple layers.
8. Deployment should be controlled and reversible.
9. Monitoring should cover both technical failures and data quality failures.
10. Federation should be monitored for latency and source-system impact.

---

## Confidentiality Note

This document is a sanitized reference architecture artifact and does not describe any proprietary client implementation.