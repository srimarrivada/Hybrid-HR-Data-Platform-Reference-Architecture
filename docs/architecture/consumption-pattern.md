# Consumption Pattern

## Overview

This document explains the consumption layer of the Hybrid HR Data Platform Reference Architecture. Products named (Power BI, Databricks SQL) are examples; the patterns apply to equivalent tools.

> Use the right consumption pattern for the audience, latency, security, and analytical need. → [D6](../adr/0006-bi-tool-by-audience.md)

The architecture supports five consumption channels:

1. [Power BI over the enterprise warehouse](#1-power-bi-over-enterprise-warehouse)
2. [Power BI over lakehouse Gold views](#2-power-bi-over-lakehouse-gold-views)
3. [Databricks SQL dashboards](#3-databricks-sql-dashboards)
4. [Curated views](#4-curated-views)
5. [File-based feeds](#5-file-based-feeds)

---

## 1. Power BI over Enterprise Warehouse

Power BI over the enterprise warehouse is used for broad HR and enterprise analytics.

### Best Suited For

- Large user populations
- Governed enterprise dashboards
- Row-level security
- Executive reporting
- Historical workforce analytics
- Sensitive and restricted data reporting

### Typical Model

- Composite Power BI model
- Import dimensions
- Import aggregation tables
- DirectQuery for large or sensitive facts
- Heavy DAX
- RLS roles

### Common Use Cases

- Headcount analytics
- Attrition analytics
- Leave analytics
- Compensation analytics
- Executive dashboards
- Workforce planning
- Data quality scorecards

### Why This Pattern Works

This pattern works well when the data warehouse is the enterprise source of truth and business users need governed, consistent, security-controlled analytics.

The warehouse publish triggers the refresh, rather than a fixed clock time. Highly restricted facts stay in the source through DirectQuery. → [D6](../adr/0006-bi-tool-by-audience.md), [D11](../adr/0011-defense-in-depth-security.md)

---

## 2. Power BI over Lakehouse Gold Views

Power BI over lakehouse Gold views is used for business-facing lakehouse analytics.

### Best Suited For

- Recruiting analytics
- Training analytics
- Departmental analytics
- Gold reporting views
- Scheduled refresh
- Business-friendly datasets

### Typical Model

- Import mode
- Native Databricks connector
- Gold reporting views
- Minimal DAX
- Pre-modeled business logic in SQL views
- Optional RLS

### Common Use Cases

- Recruiting pipeline analytics
- Candidate conversion analysis
- Requisition health
- Training enrollment analytics
- Training coverage analytics
- Session utilization
- Instructor workload

### Why This Pattern Works

This pattern works well when the lakehouse produces Gold reporting views that are already business-friendly and dashboard-ready.

Power BI imports those views on a schedule, so business users get fast dashboards without repeated live queries to the lakehouse. Federated worker context is read once per refresh, not on every click. → [D4](../adr/0004-federation-over-replication.md), [D6](../adr/0006-bi-tool-by-audience.md)

---

## 3. Databricks SQL Dashboards

Databricks SQL dashboards are used for operational monitoring directly on the lakehouse.

### Best Suited For

- Small technical teams
- Near-real-time monitoring
- Live SQL execution
- Operational SLA tracking
- No imported BI dataset

### Typical Model

- Databricks SQL Warehouse
- Gold views
- Simple KPIs
- Tables and basic charts
- Small user group
- Minimal or no RLS

### Common Use Cases

- New hire operations monitoring
- Event processing status
- Notification SLA monitoring
- Overdue operational items
- Integration health checks
- Near-real-time data quality visibility

### Why This Pattern Works

This pattern works well when:

- The audience is technical
- The user base is small
- The dashboard needs fresher data
- Users already work in Databricks
- Advanced BI visuals are not required

→ [D6](../adr/0006-bi-tool-by-audience.md)

---

## 4. Curated Views

Curated views are used for downstream consumers that need standardized relational access.

### Best Suited For

- Controlled downstream extracts
- Application integration
- Business-friendly view access
- Data minimization
- Consumer-specific shape

### Typical Model

- One view per major consumer or subject area
- Business-friendly column names
- Pre-joined required attributes
- No unnecessary sensitive fields
- Stable contract for downstream systems

### Common Use Cases

- Employee portal feeds
- Workforce change feeds
- Expense provisioning
- Onboarding applications
- Recognition systems
- Controlled application integration

### Why This Pattern Works

Curated views reduce downstream complexity and prevent consumers from accessing raw warehouse tables directly.

They provide a stable, governed, and minimized interface for applications and reporting users. → [D11](../adr/0011-defense-in-depth-security.md)

---

## 5. File-Based Feeds

File-based feeds are used for downstream systems that require files.

### Best Suited For

- Legacy integrations
- External systems
- MFT-based delivery
- Fixed format requirements
- Highly controlled field sets

### Typical Model

- Extract from curated store or semantic view
- Generate file in required format
- Encrypt if required
- Deliver through MFT / SFTP
- Track package execution history
- Archive delivered files

### Common Use Cases

- External benefits systems
- Payroll-related downstream systems
- Compliance systems
- Restricted population administration
- Legacy HR applications
- File-only vendor interfaces

### Why This Pattern Works

Many enterprise systems still require fixed-format file delivery.

A governed file feed pattern ensures data minimization, repeatability, auditability, and controlled delivery. → [D9](../adr/0009-idempotent-loads.md), [D11](../adr/0011-defense-in-depth-security.md)

---

## BI Tool Selection

| Factor | Power BI | Databricks SQL |
|---|---|---|
| Audience | Business users | Technical / operational users |
| Scale | Medium to large | Small to medium |
| Refresh | Scheduled | Live query |
| RLS | Strong support | Simpler use cases |
| Visual complexity | Advanced | Basic operational visuals |
| Best fit | Enterprise analytics | Operational monitoring |

---

## Refresh Strategy

A staggered refresh pattern helps avoid contention and aligns dashboards with upstream readiness.

Example:

```text
Warehouse publish
    ↓
Enterprise Power BI refresh
    ↓
Lakehouse reconciliation
    ↓
Lakehouse-powered Power BI refresh
    ↓
Operational dashboards remain live
```

### Benefits

- Avoids compute contention
- Aligns with upstream readiness
- Supports business-day availability
- Separates operational monitoring from enterprise reporting
- Avoids unnecessary live query load for broad business dashboards

If the DQ gate holds back the warehouse publish, the refreshes that depend on it don't run, so dashboards keep the last good data. → [F2](../failure-modes.md#f2-data-quality-gate-below-threshold)

---

## Freshness Patterns

Different consumption channels have different freshness needs.

| Channel | Typical Freshness |
|---|---|
| Enterprise Power BI dashboards | Daily / business-day refresh |
| Lakehouse-powered Power BI dashboards | Scheduled refresh after Gold data is ready |
| Databricks SQL operational dashboard | Near-real-time / live query |
| Curated views | Based on warehouse publish cycle |
| File-based feeds | Based on downstream contract |

Every dashboard should show users how current its data is.

---

## Consumption Principles

These expand [Design Principle 12](../design-principles.md#12-right-bi-tool-for-the-audience).

1. Use governed BI (e.g., Power BI) for broad business consumption.
2. Use live SQL dashboards (e.g., Databricks SQL) for operational lakehouse monitoring.
3. Do not expose raw tables directly to business users.
4. Use semantic views to simplify consumption.
5. Precompute common business logic where possible.
6. Keep sensitive data in the most governed platform.
7. Match refresh frequency to business need.
8. Use RLS where the same dataset serves multiple audiences.
9. Prefer data minimization for downstream feeds.
10. Keep dashboard freshness transparent.

---

## Related Documents

- [Architecture Overview](architecture-overview.md)
- [Platform 1: Batch Enterprise Warehouse](platform-1-batch-warehouse.md)
- [Platform 2: Cloud Lakehouse](platform-2-lakehouse.md)
- [Design Principles](../design-principles.md)
- [ADR Index](../adr/README.md)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*