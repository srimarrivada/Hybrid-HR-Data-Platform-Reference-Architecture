# Design Principles

The hybrid architecture rests on one idea:

> Use the warehouse for governed enterprise truth, use the lakehouse for integration-specific agility, and connect both through federation instead of copying data.

Each principle below links to the decision record (ADR) that applies it.

| # | Principle | Decision |
|---|---|---|
| [1](#1-keep-the-enterprise-warehouse-as-the-source-of-truth) | Keep the enterprise warehouse as the source of truth | [D1](adr/0001-two-platforms-not-one.md) |
| [2](#2-extend-dont-replace) | Extend, don't replace | [D1](adr/0001-two-platforms-not-one.md) |
| [3](#3-federation-before-replication) | Federation before replication | [D4](adr/0004-federation-over-replication.md) |
| [4](#4-facts-stay-local-to-where-the-data-originates) | Facts stay local to where the data originates | [D4](adr/0004-federation-over-replication.md) |
| [5](#5-model-only-where-analytical-value-exists) | Model only where analytical value exists | [D7](adr/0007-selective-medallion.md) |
| [6](#6-separate-raw-cleaned-modeled-and-consumed-layers) | Separate raw, cleaned, modeled and consumed layers | [D7](adr/0007-selective-medallion.md) |
| [7](#7-capture-what-the-source-wont-keep) | Capture what the source won't keep | [D12](adr/0012-bronze-captures-transient-sources.md) |
| [8](#8-handle-time-explicitly) | Handle time explicitly | [D2](adr/0002-temporal-ods-family.md), [D8](adr/0008-change-detection-strategy.md) |
| [9](#9-every-load-is-safe-to-re-run) | Every load is safe to re-run | [D9](adr/0009-idempotent-loads.md) |
| [10](#10-treat-data-quality-as-architecture) | Treat data quality as architecture | [D5](adr/0005-blocking-vs-alerting-dq.md) |
| [11](#11-reconciliation-fixes-timing-not-bad-data) | Reconciliation fixes timing, not bad data | [D5](adr/0005-blocking-vs-alerting-dq.md) |
| [12](#12-right-bi-tool-for-the-audience) | Right BI tool for the audience | [D6](adr/0006-bi-tool-by-audience.md) |
| [13](#13-secure-by-design) | Secure by design | [D11](adr/0011-defense-in-depth-security.md) |
| [14](#14-design-for-recovery) | Design for recovery | [Failure modes](failure-modes.md) |
| [15](#15-make-the-architecture-observable) | Make the architecture observable | [Governance & Operations](architecture/governance-and-operations.md) |

---

## 1. Keep the Enterprise Warehouse as the Source of Truth
The batch warehouse owns enterprise conformed dimensions and governed history, giving consistent worker and organization context, consistent KPIs, accurate historical reporting and strong security controls. → [D1](adr/0001-two-platforms-not-one.md)

## 2. Extend, Don't Replace
The lakehouse adds what the warehouse isn't built for: integration-specific analytics, operational monitoring, event-driven and micro-batch use cases, and domain-specific Gold models. It doesn't duplicate what the warehouse already masters. → [D1](adr/0001-two-platforms-not-one.md)

## 3. Federation Before Replication
When the lakehouse needs conformed dimensions, read them through federation first. Materialize only when performance, concurrency, freshness targets or query cost require it, and record that as an explicit decision, never an accidental one. → [D4](adr/0004-federation-over-replication.md)

## 4. Facts Stay Local to Where the Data Originates
Enterprise workforce facts stay in the warehouse; integration-specific facts stay in the lakehouse. This avoids cross-platform movement, duplicate processing, sync complexity and unclear ownership. → [D4](adr/0004-federation-over-replication.md)

## 5. Model Only Where Analytical Value Exists
Build a Gold model when there is an analytical audience, historical value, and questions no other source can answer. Skip it when the flow is operational only, monitoring is enough, or the data already exists in a conformed source. Skipped flows are still monitored. → [D7](adr/0007-selective-medallion.md)

## 6. Separate Raw, Cleaned, Modeled and Consumed Layers
`Bronze → Silver → Gold → Reporting Views`
- **Bronze:** raw audit trail and replay source
- **Silver:** cleaned, validated, standardized data
- **Gold:** analytics-ready dimensional model
- **Reporting views:** the stable contract for BI and consumers

→ [D7](adr/0007-selective-medallion.md)

## 7. Capture What the Source Won't Keep
If a source deletes, overwrites or forwards data without keeping history, capture it on arrival before any transformation. Otherwise trends and replay are impossible. → [D12](adr/0012-bronze-captures-transient-sources.md)

## 8. Handle Time Explicitly
HR data is effective-dated. Keep current, back-dated and future-dated changes apart; detect change by content, not timestamps; correct history instead of overwriting it. → [D2](adr/0002-temporal-ods-family.md), [D8](adr/0008-change-detection-strategy.md)

## 9. Every Load Is Safe to Re-Run
State (watermarks, versions, publishes) advances only after success. Use upserts instead of blind inserts, and guard additive merges with run logs. Recovery should never require a judgement call about duplicates. → [D9](adr/0009-idempotent-loads.md)

## 10. Treat Data Quality as Architecture
Design DQ into capture, transformation, modeling, reconciliation, consumption and monitoring. Block publishing where bad data is unacceptable; alert where operations must keep flowing. → [D5](adr/0005-blocking-vs-alerting-dq.md)

## 11. Reconciliation Fixes Timing, Not Bad Data
- A new hire's unresolved worker key may be expected temporarily; reconcile it later.
- An existing employee's unresolved key is a DQ issue.
- External applicants are never forced into worker reconciliation.

→ [D5](adr/0005-blocking-vs-alerting-dq.md)

## 12. Right BI Tool for the Audience
- **Governed BI (e.g., Power BI or equivalent):** broad audiences, row-level security, advanced visuals, daily refresh, workspace governance.
- **Live SQL dashboards (e.g., Databricks SQL or equivalent):** small technical audiences, live monitoring, simple KPIs and tables.

→ [D6](adr/0006-bi-tool-by-audience.md)

## 13. Secure by Design
Protect sensitive data in layers: schema separation, least privilege, role-based access, row-level security, column masking, restricted workspaces and audit logging. Highly restricted data stays in the most governed platform unless there is a justified need. → [D11](adr/0011-defense-in-depth-security.md)

## 14. Design for Recovery
Every critical flow has a recovery path, defined before failure happens: re-run from the last successful state, replay from Bronze, table time travel, keep the prior published warehouse version, reprocess from ODS history, and manual reprocessing for integrations. → [Failure modes](failure-modes.md)

## 15. Make the Architecture Observable
Monitor job outcomes, runtimes, data freshness, DQ status, BI refreshes, source volumes, federation performance and error patterns, for both day-to-day operations and root-cause analysis. → [Governance & Operations](architecture/governance-and-operations.md)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*