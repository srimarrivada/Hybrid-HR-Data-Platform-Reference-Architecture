# Platform 2: Cloud Lakehouse

## Overview

Platform 2 is the cloud lakehouse side of the hybrid architecture: the **consumer and integrator** for integration-specific use cases. It **extends** the enterprise warehouse and does not replace it. → [D1](../adr/0001-two-platforms-not-one.md)

It provides integration-specific analytics, operational monitoring and domain-specific Gold models using Medallion Architecture on Delta tables, with live SQL and governed BI consumption. Products named (e.g., Azure Databricks, Delta Lake, Unity Catalog, Databricks SQL, Power BI) are examples; the patterns apply to equivalent technologies.

---

## Primary Responsibilities

Platform 2 is responsible for:

- Processing event-driven and micro-batch integrations
- Capturing raw data in Bronze on arrival
- Standardizing and validating data in Silver
- Building integration-specific Gold models where they add analytical value
- Tracking the status of every integration record
- Writing DQ check results and reconciling timing gaps
- Supporting live operational dashboards and business dashboards over Gold views
- Reusing enterprise conformed dimensions through federation

---

## Source Patterns

Representative source types:

- Message queue events
- Operational near-real-time databases
- REST / SOAP APIs
- CSV files
- Managed file transfer feeds
- File arrival triggers
- Scheduled micro-batches

Common ingestion patterns:

- JDBC incremental reads
- File capture
- API extraction
- Event processing
- Watermark-based extraction
- File-level deduplication

Each source is assessed for history, capture urgency, identifiers and sensitivity before design. → [Source assessment](../source-assessment.md)

---

## Medallion Architecture

```text
Sources
    ↓
Bronze      raw, as received
    ↓
Silver      cleaned and validated
    ↓
Gold        analytics-ready model
    ↓
Reporting views   stable BI contract
    ↓
BI consumption
```

### Bronze Layer

Preserves source data **as received**, before any transformation, encryption or forwarding. Some sources keep no history (transient exports, overwritten records), so capturing on arrival is the only way to keep their history and support replay. → [D12](../adr/0012-bronze-captures-transient-sources.md)

Supports:
- Raw capture and audit trail
- Replay and source reconstruction
- File metadata preservation

Typical patterns:
- Append-only raw tables
- Full overwrite for small reference data
- Load timestamp, source identifier and file name
- Defined retention per source
- Access restricted to platform engineers

Bronze is the recovery and replay foundation of the lakehouse.

### Silver Layer

Cleans and validates data, and is the data quality boundary.

- Type casting
- Deduplication
- Standardization
- Status derivation
- Business classification
- Enrichment from secondary sources
- Schema validation (a schema change stops that source only)

### Gold Layer

Analytics-ready dimensional models.

Gold includes:
- Local facts and local dimensions
- Federated dimensions (read from Platform 1)
- Shared DQ log
- Reporting views

Representative Gold models:
- New hire operational analytics
- Recruiting pipeline analytics
- Training enrollment analytics

Modeling patterns are chosen for the questions: for example, an ordered **step dimension** for funnel conversion and time-in-stage, and **dual-key lookups** where source systems identify the same entity differently.

---

## Medallion Only Where It Adds Value

Not every integration gets a Gold layer. An integration is modeled only if **all** of these hold: → [D7](../adr/0007-selective-medallion.md)

1. It builds history that isn't kept anywhere else.
2. It has a real analytical audience.
3. It answers questions no existing source can.
4. The value justifies the build and run effort.

A simple system-to-system sync that keeps only current state, whose data already exists in the warehouse, is a deliberate **skip**. Skipped integrations are still monitored, logged and reprocessable.

---

## Integration Reliability

- Every record is tracked: **pending → success / failed**, with the error captured.
- A failed record is isolated, and the rest of the batch continues.
- Token refresh and retries with backoff handle transient errors.
- Processing is idempotent, so replaying from Bronze never sends a record twice. → [D9](../adr/0009-idempotent-loads.md)

→ [F3: Integration record or API call fails](../failure-modes.md#f3-integration-record-or-api-call-fails)

---

## Local vs Federated Objects

### Local Objects

Integration-specific objects live in the lakehouse (Delta tables), maintained by lakehouse pipelines:

- Event type
- Notification status
- Candidate stage
- Requisition
- Session
- Enrollment status
- Integration-specific facts

### Federated Objects

Enterprise conformed dimensions stay in Platform 1 and are read in place through governed federated views (e.g., Lakehouse Federation): → [D4](../adr/0004-federation-over-replication.md)

- Worker
- Date
- Job
- Company
- Location
- Department
- Business Unit

Keys are resolved at ETL time and stored in Gold facts, so dashboards don't trigger federation queries. → [Federation pattern](federation-pattern.md)

---

## Data Quality Pattern

Platform 2 uses **alert-oriented, non-blocking** DQ with a shared Gold-layer DQ log. Issues are logged and alerted, but they don't block the pipeline, so operations keep flowing. → [D5](../adr/0005-blocking-vs-alerting-dq.md)

Typical checks:

- Unresolved worker keys
- Unresolved recruiter keys
- Unresolved session keys
- Missing candidate stages
- Volume anomalies
- Freshness / SLA breaches
- Business-rule spikes (e.g., unusual withdrawal rates)

---

## Reconciliation Pattern

Reconciliation is used only where unresolved keys are expected because of timing. Records that can't be resolved yet get the **unknown-member key** and a DQ entry, rather than being dropped.

How it applies:

- **New hire:** the worker may not exist in the enterprise worker dimension yet; reconcile after the next warehouse publish.
- **Recruiting:** the candidate is external; no worker reconciliation is needed.
- **Training:** the learner or instructor should already exist; an unresolved key is a DQ alert.

The strategy follows business meaning, not just technical lookup failure. Keys that stay unresolved beyond the expected window are investigated as source issues. → [Design Principle 11](../design-principles.md#11-reconciliation-fixes-timing-not-bad-data)

If federation is slow or the warehouse is unavailable, the lakehouse keeps serving its last good Gold tables. → [F4](../failure-modes.md#f4-federation-slow-or-enterprise-warehouse-unavailable)

---

## BI Consumption

Platform 2 supports two consumption styles. → [D6](../adr/0006-bi-tool-by-audience.md)

| Style | Best for | Typical use cases |
|---|---|---|
| **Live SQL dashboards** (e.g., Databricks SQL) | Small technical audiences, live queries, simple KPIs and tables | New hire monitoring, SLA tracking, overdue notifications, freshness and status checks |
| **Governed BI** (e.g., Power BI), import mode | Larger business audiences, scheduled refresh, row-level security, advanced visuals | Recruiting pipeline, training enrollment, leadership reporting, self-service analysis |

Details: [Consumption Pattern](consumption-pattern.md)

---

## Strengths and Limitations

| Strong At | Should Avoid |
|---|---|
| Fast integration-specific analytics | Replicating conformed dimensions unnecessarily |
| Operational monitoring and replay | Rebuilding enterprise history that already exists in Platform 1 |
| Event and micro-batch processing | Holding restricted or highly restricted data (it stays in Platform 1) |
| Flexible, domain-specific Gold models | Modeling operational-only flows with no analytical audience |
| Reuse of enterprise context through federation | Duplicate KPI definitions that conflict with the warehouse |

Platform 2 should not become a second enterprise warehouse. → [D11](../adr/0011-defense-in-depth-security.md)

---

## Relationship to Platform 1

Platform 2 depends on Platform 1 for enterprise conformed context. Platform 1 remains the source of truth for:

- Worker identity
- Worker organization context
- Date
- Job
- Company
- Location
- Department
- Business Unit

Platform 2 extends Platform 1 by adding:

- Integration-specific lakehouse analytics
- Operational monitoring
- Domain-specific Gold models
- Event-driven and micro-batch processing
- Live operational dashboards

---

## Lakehouse Rules

These apply the [Design Principles](../design-principles.md) to Platform 2:

1. Use Bronze to preserve raw source data, captured on arrival.
2. Use Silver to clean, validate and standardize data.
3. Use Gold for analytics-ready dimensional models, only where they create analytical value.
4. Keep integration-specific dimensions local to the lakehouse.
5. Reuse enterprise conformed dimensions through federation.
6. Avoid persistent replication unless performance or freshness targets require it, and record that decision.
7. Track every integration record and make every load safe to replay.
8. Treat DQ and reconciliation as first-class architecture components.
9. Use live SQL dashboards for operational monitoring.
10. Use governed BI for broader business analytics.

---

## Related Documents

- [Architecture Overview](architecture-overview.md)
- [Platform 1: Batch Enterprise Warehouse](platform-1-batch-warehouse.md)
- [Federation Pattern](federation-pattern.md)
- [Consumption Pattern](consumption-pattern.md)
- [Governance and Operations](governance-and-operations.md)
- [Failure Modes](../failure-modes.md)
- [ADR Index](../adr/README.md)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*