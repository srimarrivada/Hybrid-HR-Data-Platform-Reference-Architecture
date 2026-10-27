# Platform 1: Batch Enterprise Warehouse

## Overview

Platform 1 represents the traditional enterprise data warehouse side of the hybrid architecture.

It is designed for governed, enterprise-wide HR analytics using batch ingestion, operational data stores, curated data stores, Kimball dimensional modeling, Power BI consumption, and strong security controls.

This platform acts as the enterprise source of truth for conformed dimensions and historical workforce analytics.

---

## Primary Responsibilities

Platform 1 is responsible for:

- Ingesting system-of-record HR data
- Processing nightly batch data
- Maintaining current, historical, effective-dated, and future-dated records
- Building conformed dimensions
- Maintaining enterprise fact tables
- Applying strong data quality gates
- Supporting sensitive and highly restricted HR analytics
- Feeding downstream operational and analytical systems
- Serving broad enterprise Power BI dashboards

---

## Source Patterns

Representative source patterns include:

- HCM system APIs
- Custom reports
- Payroll files
- Benefits files
- Time and hours files
- Restricted census files
- Static reference files
- Master data and directory lookups

Common ingestion methods:

- API extraction
- SFTP / MFT file transfer
- Batch orchestration
- Reference lookup enrichment
- Effective-dated extraction
- Future-dated extraction

---

## Operational Data Store Pattern

Platform 1 uses operational data stores to organize incoming data before warehouse processing.

Typical ODS patterns include:

- Current ODS
- Historical ODS
- Effective-dated ODS
- Future-dated ODS
- Full / recovery ODS
- Domain-specific ODS databases

The ODS layer supports:

- Raw and semi-curated persistence
- Reprocessing
- Recovery
- Temporal processing
- Reference enrichment
- Transformation staging

---

## Processing Pattern

The processing flow typically follows this pattern:

```
Source Systems
    ↓
API / File Extraction
    ↓
Operational Data Stores
    ↓
Stored Procedure Transformations
    ↓
CDC / Delta Detection
    ↓
Curated Data Store
    ↓
Dimensional Warehouse
```

---

## Key processing capabilities:

- Dependency-driven batch orchestration
- Watermark / state management
- CDC and delta detection
- SCD Type 1 and Type 2 processing
- Reference lookups
- Data quality gates
- Audit logging
- Restart / recovery support

---

## Dimensional Warehouse

The warehouse follows Kimball dimensional modeling principles.

Core components include:

- Conformed dimensions
- Role-playing dimensions
- SCD Type 2 dimensions
- Transaction facts
- Periodic snapshot facts
- Accumulating snapshot facts
- Factless facts
- Bridge tables
- Aggregation tables
- Utility and audit tables

Representative subject areas:

- Headcount
- Hiring
- Termination
- Movement
- Leave
- Hours
- Performance
- Compensation
- Planning
- Partner / retiree analytics
- Data quality scorecards

---

## Security Model

Platform 1 manages sensitive HR data using tiered security.

Typical tiers:

1. General analytics
2. Restricted data
3. Highly restricted data

Security controls include:

- Schema-level separation
- Explicit GRANT / DENY
- Row-level security in BI
- Restricted Power BI workspaces
- User security mapping
- Audit logging
- Least-privilege access

This platform is the correct location for highly sensitive HR attributes such as compensation, national identifiers, demographics, and legal-sensitive data.

---

## Data Quality Pattern

Platform 1 uses stronger DQ gating because it supports broad enterprise analytics.

Typical DQ capabilities:

- Referential integrity checks
- Completeness checks
- Duplicate checks
- Source-to-target reconciliation
- Volume thresholds
- SCD2 validation
- Score-based DQ gating
- Blocking publish if DQ falls below threshold

The warehouse is published only after the quality gate passes.

---

## BI Consumption

Platform 1 primarily serves Power BI dashboards.

Consumption patterns include:

- Composite Power BI models
- Import mode for dimensions and aggregations
- DirectQuery for large or sensitive facts
- Row-level security
- Aggregation-backed performance
- Executive and operational dashboards

---

## Strengths of Platform 1

Platform 1 is strong in:

- Strong governance
- Historical depth
- Enterprise conformed dimensions
- Sensitive data handling
- Broad BI consumption
- Mature batch operations
- Trusted system of record for workforce analytics

---

## Limitations

Platform 1 is less optimal for:

- Near-real-time operational monitoring
- Frequent event-driven use cases
- Lightweight integration-specific analytics
- Rapid experimentation on transient source feeds

These use cases are handled by Platform 2.

---

## Relationship to Platform 2

Platform 1 remains the source of truth for enterprise conformed dimensions and governed historical workforce analytics.

Platform 2 extends this architecture by supporting lakehouse-based integration analytics, event-driven processing, operational dashboards, and domain-specific Gold models.

The two platforms are connected through Lakehouse Federation, allowing Platform 2 to reuse Platform 1 conformed dimensions without persistent replication.

---

## Confidentiality Note

This document is a sanitized reference architecture artifact and does not describe any proprietary client implementation.