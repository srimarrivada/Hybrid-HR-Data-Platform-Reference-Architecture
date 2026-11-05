# Architecture Overview

## Purpose

This document gives a high-level overview of the Hybrid HR Data Platform Reference Architecture and explains how a batch-oriented enterprise data warehouse and a cloud lakehouse coexist in one analytics ecosystem, each doing what it does best, connected without copying data.

The design combines:

- Batch enterprise warehouse patterns such as operational data stores, change detection, curated stores
- Dimensional modeling (Kimball) on a relational warehouse (e.g., SQL Server)
- Lakehouse processing (e.g., Azure Databricks, Delta Lake) with Medallion Architecture
- Zero-copy federation between the platforms (e.g., Lakehouse Federation)
- Consumption by audience such as governed BI (e.g., Power BI) and live SQL dashboards (e.g., Databricks SQL)
- Data quality, governance, monitoring and recovery patterns across both platforms

---

## Architecture Diagram

![Hybrid HR Data Platform Reference Architecture](../diagrams/svg/hybrid-hr-platform-reference-architecture.drawio.svg)

For the interactive version (layers, decision markers, tooltips), see the link in the [README](../../README.md#detailed-diagram).

---

## Architecture Summary

The architecture contains two platforms with distinct roles. → [D1](../adr/0001-two-platforms-not-one.md)

### Platform 1: Batch Enterprise Warehouse

The **producer** of enterprise conformed dimensions and governed history.

It supports:
- Dependency-tiered nightly batch ingestion → [D3](../adr/0003-dependency-tiered-batch.md)
- Temporal operational data stores (current, effective-dated, future-dated, history, full) → [D2](../adr/0002-temporal-ods-family.md)
- Content-based change detection into a curated store → [D8](../adr/0008-change-detection-strategy.md)
- Kimball dimensional model with conformed dimensions, facts and aggregates → [D10](../adr/0010-kimball-dimensional-model.md), [D13](../adr/0013-fact-design-patterns.md)
- A **blocking** data quality gate before publish → [D5](../adr/0005-blocking-vs-alerting-dq.md)
- Tiered security for sensitive HR data → [D11](../adr/0011-defense-in-depth-security.md)
- Governed enterprise dashboards, curated views and file feeds

Optimized for:
 - Historical reporting
 - Enterprise KPIs 
 - Workforce analytics
 - Broad BI consumption
 - Sensitive, highly governed data

### Platform 2: Cloud Lakehouse

The **consumer and integrator** for integration-specific use cases. It extends the warehouse and does not replace it.

It supports:
- Event-driven and micro-batch integrations with status tracking and replay → [D9](../adr/0009-idempotent-loads.md)
- Bronze / Silver / Gold layers, with Bronze capturing transient sources → [D12](../adr/0012-bronze-captures-transient-sources.md)
- Gold models built only where they add analytical value → [D7](../adr/0007-selective-medallion.md)
- **Alerting** data quality checks plus reconciliation → [D5](../adr/0005-blocking-vs-alerting-dq.md)
- Live operational dashboards and imported business dashboards over Gold views → [D6](../adr/0006-bi-tool-by-audience.md)

Optimized for:
- Operational monitoring
- Integration analytics
- Near-real-time visibility 
- Lightweight domain models

---

## Federation Seam

The warehouse owns enterprise conformed dimensions such as:

- Worker
- Date
- Job
- Company
- Location
- Department
- Business Unit

The lakehouse reads these **in place** through a small number of governed federated views instead of copying them into Delta Lake. Keys are resolved at ETL time and stored in Gold facts, so dashboards don't trigger federation queries. Restricted data is never exposed through federation. → [D4](../adr/0004-federation-over-replication.md)

--- 

## High-Level Flow

```text
 HCM system of record + providers            Events / APIs / files
          │ nightly batch                       │ events + micro-batches
          ▼                                     ▼
 ┌─────────────────────────────┐        ┌─────────────────────────────┐
 │ Platform 1 - Batch warehouse│        │ Platform 2 - Lakehouse      │
 │ ODS → change detection →    │──────▶ │ Bronze → Silver → Gold →    │
 │ curated store → Kimball DW  │federa- │ reporting views             │
 │ + blocking DQ gate          │ tion   │ + alerting DQ               │
 └─────────────────────────────┘(read   └─────────────────────────────┘
          │                      in place)       │
          ▼                                      ▼
 Enterprise BI                            Live ops dashboard
 Curated views                            Lakehouse BI 
 File feeds to downstream systems         Integrations
```

Each platform ingests its own sources. Federation carries only conformed context (worker, organization, date) from Platform 1 to Platform 2.

---

## Key Architecture Characteristics

- Hybrid batch + event / micro-batch design
- Enterprise warehouse is the source of truth for conformed data and sensitive data
- Lakehouse for integration-specific analytics and operational monitoring
- Federation instead of persistent replication for shared dimensions
- Blocking DQ for enterprise publish and alerting DQ for operations
- BI tool and refresh mode chosen by audience
- Every load safe to re-run; documented recovery for every critical flow → [Failure Modes](../failure-modes.md)
- CI/CD and monitoring across both platforms → [Governance and Operations](governance-and-operations.md)

---

## Related Documents

- [Platform 1: Batch Enterprise Warehouse](platform-1-batch-warehouse.md)
- [Platform 2: Cloud Lakehouse](platform-2-lakehouse.md)
- [Federation Pattern](federation-pattern.md)
- [Consumption Pattern](consumption-pattern.md)
- [Governance and Operations](governance-and-operations.md)
- [Design Principles](../design-principles.md)
- [ADR Index](../adr/README.md)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*