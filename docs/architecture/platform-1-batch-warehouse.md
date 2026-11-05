# Platform 1: Batch Enterprise Warehouse

## Overview

Platform 1 is the traditional enterprise data warehouse side of the hybrid architecture: the **producer** of enterprise conformed dimensions and governed workforce history. → [D1](../adr/0001-two-platforms-not-one.md)

It provides governed, enterprise-wide HR analytics using nightly batch ingestion, temporal operational data stores, content-based change detection, a curated data store, a Kimball dimensional warehouse, governed BI and strong security controls. Products named (e.g., SQL Server, Informatica, Power BI) are examples; the patterns apply to equivalent technologies.

---

## Primary Responsibilities

Platform 1 is responsible for:

- Ingesting system-of-record HR data
- Processing nightly batch data
- Maintaining current, historical, effective-dated and future-dated records
- Building conformed dimensions
- Maintaining enterprise fact tables
- Applying a blocking data quality gate before publish
- Supporting sensitive and highly restricted HR analytics
- Feeding downstream operational and analytical systems
- Serving broad, governed enterprise dashboards

---

## Source Patterns

Representative Sources:

- HCM system APIs
- Custom reports
- Payroll files
- Benefits files
- Time and hours files
- Restricted population files
- Static reference files
- Master data and directory lookups

Common Ingestion Methods:

- API extraction
- SFTP / MFT file transfer
- Batch orchestration
- Reference lookup enrichment
- Effective-dated extraction
- Future-dated extraction

Each source is assessed before design. → [Source assessment](../source-assessment.md)

---

## Operational Data Store Pattern

Platform 1 uses a **family of operational data stores**. They share one schema and one code base, and each loads a different time window. → [D2](../adr/0002-temporal-ods-family.md)

| Store | Role |
|---|---|
| Current ODS (primary) | Latest changes; runs all transformations and feeds change detection |
| Effective-dated ODS | Catches back-dated corrections; merged back into Current (insert-only) |
| Future-dated ODS | Holds changes ahead of their effective date (e.g., pre-hire provisioning) |
| History ODS | Append-only snapshots for point-in-time reconstruction |
| Full / recovery ODS | Complete extract on demand for recovery and history reloads |
| Domain ODS databases | Payroll, benefits, time and other provider data, plus restricted population data |

The ODS layer supports:

- Raw and semi-curated persistence
- Reprocessing and recovery
- Temporal processing
- Reference enrichment
- Transformation staging

Complex business logic (hierarchy traversal, classification rules, effective-date handling) lives in set-based SQL, not visual ETL mappings. The ETL tool handles data movement and orchestration.

---

## Processing Pattern

```text
Source systems
    ↓
API / file extraction
    ↓
Temporal operational data stores
    ↓
Set-based SQL transformations
    ↓
Content-based change detection
    ↓
Curated data store (history kept, SCD2)
    ↓
Dimensional warehouse
    ↓
DQ gate → publish
```

### Key Processing Capabilities

- **Dependency-driven orchestration:** Jobs start when their predecessors finish, with milestone gates and tiered warehouse loads. → [D3](../adr/0003-dependency-tiered-batch.md)
- **Content-based change detection:** Set-based compare or row hash, excluding volatile columns. → [D8](../adr/0008-change-detection-strategy.md)
- **Watermark / state management:** The extraction window advances only after full success. → [D9](../adr/0009-idempotent-loads.md)
- **SCD processing:** Type 1 for corrections, Type 2 where history matters; history is corrected, not overwritten. → [D10](../adr/0010-kimball-dimensional-model.md)
- **Reference enrichment:** Lookups to reference and master data are applied during transformation, before change detection.
- **Data quality gate:** The warehouse is published only when quality checks pass. → [D5](../adr/0005-blocking-vs-alerting-dq.md)
- **Audit logging:** Every batch, error and DQ result is logged for root-cause analysis.
- **Restart and recovery:** A failed run resumes from the failed step; completed steps are not repeated. → [F1](../failure-modes.md#f1-nightly-run-fails-part-way)
---

## Dimensional Warehouse

The warehouse follows Kimball dimensional modeling principles. → [D10](../adr/0010-kimball-dimensional-model.md)

Core Components:

- Conformed dimensions
- Role-playing dimensions
- SCD Type 2 dimensions
- Transaction facts
- Periodic snapshot facts
- Accumulating snapshot facts
- Factless facts
- Bridge tables (weighted, to prevent double counting)
- Aggregation tables
- Utility and audit tables

Design patterns such as consolidated facts, junk dimensions and weighted bridges are described in [D13](../adr/0013-fact-design-patterns.md).

Representative Subject Areas:

- Headcount
- Hiring
- Termination
- Movement
- Leave
- Hours
- Performance
- Compensation
- Planning
- Restricted population analytics
- Data quality scorecards

Back-dated changes correct history rather than overwrite it. → [F5](../failure-modes.md#f5-late-or-back-dated-source-changes)

---

## Security Model

Platform 1 manages sensitive HR data using tiered security. → [D11](../adr/0011-defense-in-depth-security.md)

Typical Tiers:

1. General analytics
2. Restricted data
3. Highly restricted data

Security Controls:

- Schema-level separation
- Explicit GRANT / DENY
- Row-level security in BI
- Restricted BI workspaces
- User security mapping
- Audit logging
- Least-privilege access

This platform is the right place for highly sensitive HR attributes such as compensation, national identifiers, demographics and legal-sensitive data. That data is never exposed through federation. → [Security tiers](governance-and-operations.md#security-tiers)

---

## Data Quality Pattern

Platform 1 uses a **blocking** DQ gate because it serves broad enterprise analytics. → [D5](../adr/0005-blocking-vs-alerting-dq.md)

Typical DQ Capabilities:

- Referential integrity checks
- Completeness checks
- Duplicate checks
- Source-to-target reconciliation
- Volume thresholds
- SCD2 validation
- Score-based gating, with critical rules that always block

The enterprise warehouse is published only after the gate passes. If it fails, the previous version stays live. → [F2](../failure-modes.md#f2-data-quality-gate-below-threshold)

---

## BI Consumption

Platform 1 primarily serves governed enterprise dashboards (e.g., Power BI). → [D6](../adr/0006-bi-tool-by-audience.md)

- Composite models: import mode for dimensions and aggregations, DirectQuery for large or highly restricted facts
- Row-level security
- Aggregation-backed performance for trend visuals
- Refresh triggered by the warehouse publish, not a clock
- Executive and operational dashboards

Platform 1 also serves curated views and file-based feeds to downstream systems. → [Consumption pattern](consumption-pattern.md)

---

## Strengths and Limitations

| Strong At | Less Suited To |
|---|---|
| Governance and sensitive data handling | Near-real-time operational monitoring |
| Historical depth | Frequent event-driven use cases |
| Enterprise conformed dimensions | Lightweight integration-specific analytics |
| Broad BI consumption | Rapid experimentation on transient source feeds |
| Mature batch operations; trusted system of record | |

The use cases in the right-hand column are handled by Platform 2.

---

## Relationship to Platform 2

Platform 1 remains the source of truth for enterprise conformed dimensions and governed historical workforce analytics.

Platform 2 extends the architecture with lakehouse-based integration analytics, event-driven processing, operational dashboards and domain-specific Gold models.

The two platforms are connected through zero-copy federation (e.g., Lakehouse Federation). Platform 2 reads Platform 1's conformed dimensions in place, without persistent replication. → [D4](../adr/0004-federation-over-replication.md), [Federation pattern](federation-pattern.md)

---

## Related Documents

- [Architecture Overview](architecture-overview.md)
- [Platform 2: Cloud Lakehouse](platform-2-lakehouse.md)
- [Federation Pattern](federation-pattern.md)
- [Governance and Operations](governance-and-operations.md)
- [Failure Modes](../failure-modes.md)
- [ADR Index](../adr/README.md)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*