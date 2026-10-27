# Platform 2: Cloud Lakehouse

## Overview

Platform 2 represents the Azure Databricks Lakehouse side of the hybrid architecture.

It extends the enterprise warehouse by providing integration-specific analytics, operational monitoring, Medallion Architecture, Delta Lake processing, Gold-layer models, and Databricks SQL / Power BI consumption patterns.

This platform does not replace the enterprise warehouse. It extends it.

---

## Primary Responsibilities

Platform 2 is responsible for:

- Processing event-driven and micro-batch integrations
- Capturing raw data in Bronze
- Standardizing and validating data in Silver
- Building integration-specific Gold models
- Writing DQ check results
- Supporting operational dashboards
- Supporting Power BI dashboards over Gold views
- Reusing enterprise conformed dimensions through federation

---

## Source Patterns

Representative source types include:

- Message queue events
- Operational NRT databases
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

---

## Medallion Architecture

The lakehouse follows Medallion Architecture.

```
Sources
    ↓
Bronze
    ↓
Silver
    ↓
Gold
    ↓
Reporting Views
    ↓
BI Consumption
```

---

## Bronze Layer

The Bronze layer preserves source data as received.

### Purpose

The Bronze layer supports:

- Raw capture
- Audit trail
- Replay support
- Source reconstruction
- File metadata preservation

### Typical Patterns

- Append-only raw tables
- Full overwrite for small reference data
- Load timestamp
- Source identifier
- File name where applicable
- Durable retention

Bronze is the recovery and replay foundation of the lakehouse.

---

## Silver Layer

The Silver layer cleans and validates data.

### Purpose

The Silver layer supports:

- Type casting
- Deduplication
- Standardization
- Status derivation
- Business classification
- Data quality boundary
- Enrichment from secondary sources

Silver prepares data for Gold modeling.

---

## Gold Layer

The Gold layer provides analytics-ready dimensional models.

Gold includes:

- Local facts
- Local dimensions
- Federated dimensions
- Shared DQ log
- Reporting views

Representative Gold models include:

- New Hire operational analytics
- Recruiting pipeline analytics
- Training enrollment analytics

Gold is the primary analytical layer consumed by Databricks SQL and Power BI.

---

## Local vs Federated Objects

Platform 2 separates local integration-specific objects from federated enterprise conformed dimensions.

---

### Local Objects

Integration-specific objects live in Delta Lake.

Examples:

- Event type
- Notification status
- Candidate stage
- Requisition
- Session
- Enrollment status
- Integration-specific facts

These objects are maintained by lakehouse pipelines because they are specific to the integration domain.

---

### Federated Objects

Enterprise conformed dimensions remain in Platform 1 and are accessed through Lakehouse Federation.

Examples:

- Worker
- Date
- Job
- Company
- Location
- Department
- Line of Service

Federation allows Platform 2 to reuse enterprise context without copying conformed dimensions into Delta Lake.

---

## Data Quality Pattern

Platform 2 uses a shared Gold-layer DQ log.

Typical DQ checks include:

- Unresolved worker keys
- Unresolved recruiter keys
- Unresolved session keys
- Missing candidate stages
- Volume anomalies
- SLA breaches
- Withdrawal rate spikes

DQ is alert-oriented and non-blocking.

This means DQ issues are logged and alerted, but the pipeline is not automatically blocked for every warning or failure.

---

## Reconciliation Pattern

Reconciliation is used only where unresolved keys are expected due to timing.

Examples:

- **New Hire:** worker may not exist in the enterprise worker dimension yet; reconcile later.
- **Recruiting:** candidate is external; no worker reconciliation is required.
- **Training:** learner/instructor should already exist; unresolved key becomes a DQ alert.

The reconciliation strategy is based on business meaning, not just technical lookup failure.

---

## BI Consumption

Platform 2 supports two consumption styles:

1. Databricks SQL
2. Power BI

---

### Databricks SQL

Best for:

- Operational monitoring
- Small technical audiences
- Live SQL execution
- Near-real-time dashboards
- Simple KPI and table-based operational views

Typical use case:

- New Hire operational monitoring
- SLA tracking
- Overdue notification monitoring
- Data freshness / operational status checks

---

### Power BI

Best for:

- Business-facing analytics
- Larger audiences
- Scheduled refresh
- Row-level security
- Advanced visuals
- Familiar enterprise reporting experience

Typical use cases:

- Recruiting pipeline analytics
- Training enrollment analytics
- Leadership reporting
- Self-service business analysis

---

## Strengths of Platform 2

Platform 2 is strong in:

- Fast integration-specific analytics
- Reduced replication
- Flexible lakehouse modeling
- Operational monitoring
- Event and micro-batch processing
- Domain-specific Gold models
- Federation with enterprise conformed context
- Rapid extension of analytics use cases

---

## Limitations

Platform 2 should not become a duplicate enterprise warehouse.

It should avoid:

- Replicating all conformed dimensions unnecessarily
- Rebuilding enterprise history that already exists in Platform 1
- Storing sensitive data unless explicitly governed
- Modeling operational-only use cases with no analytical audience
- Creating duplicate KPI definitions that conflict with the enterprise warehouse

---

## Relationship to Platform 1

Platform 2 depends on Platform 1 for enterprise conformed context.

Platform 1 remains the source of truth for:

- Worker identity
- Worker organization context
- Date
- Job
- Company
- Location
- Department
- Line of Service

Platform 2 extends Platform 1 by adding:

- Integration-specific lakehouse analytics
- Operational monitoring
- Domain-specific Gold models
- Micro-batch/event-driven processing
- Databricks SQL dashboards

---

## Design Principles

1. Use Bronze to preserve raw source data.
2. Use Silver to clean, validate, and standardize data.
3. Use Gold to build analytics-ready dimensional models.
4. Keep integration-specific dimensions local to Delta Lake.
5. Reuse enterprise conformed dimensions through federation.
6. Avoid persistent replication unless performance or SLA requires it.
7. Treat DQ and reconciliation as first-class architecture components.
8. Use Databricks SQL for operational monitoring.
9. Use Power BI for broader business analytics.
10. Apply dimensional modeling only where it creates analytical value.

---

## Confidentiality Note

This document is a sanitized reference architecture artifact and does not describe any proprietary client implementation.