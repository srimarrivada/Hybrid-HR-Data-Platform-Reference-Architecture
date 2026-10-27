# Architecture Overview

## Purpose

This document provides a high-level overview of the Hybrid HR Data Platform Reference Architecture.

The architecture demonstrates how a traditional batch-oriented enterprise data warehouse and a modern cloud lakehouse can coexist within the same enterprise analytics ecosystem.

The design combines:

- Batch enterprise data warehouse patterns
- Operational data stores
- ETL orchestration
- SQL Server dimensional modeling
- Azure Databricks lakehouse processing
- Medallion Architecture
- Lakehouse Federation
- Power BI and Databricks SQL consumption
- Data quality, governance, monitoring, and recovery patterns

This is a sanitized portfolio artifact created to demonstrate enterprise data platform architecture thinking. It is not client documentation and does not include proprietary data, production code, credentials, internal system names, or real employee/customer data.

---

## Architecture Diagram

![Hybrid HR Data Platform Reference Architecture](../diagrams/hybrid-hr-data-platform-reference-architecture.png)

---

## Architecture Summary

The reference architecture contains two major platforms.

## Platform 1: Batch Enterprise Warehouse

Platform 1 represents a traditional enterprise warehouse architecture.

It supports:

- Nightly batch ingestion
- Operational data stores
- Curated enterprise data stores
- Kimball dimensional modeling
- Conformed dimensions
- Fact and aggregate tables
- Power BI enterprise dashboards
- Strong security and governance controls

This platform is optimized for:

- Historical reporting
- Enterprise KPIs
- Workforce analytics
- Broad BI consumption
- Sensitive and highly governed HR data

---

## Platform 2: Cloud Lakehouse

Platform 2 represents a cloud lakehouse extension.

It supports:

- Event-driven and micro-batch integration analytics
- Azure Databricks processing
- Delta Lake storage
- Bronze / Silver / Gold layers
- Integration-specific Gold models
- Data quality checks
- Databricks SQL operational dashboards
- Power BI dashboards over Gold views
- Lakehouse Federation for enterprise conformed dimensions

This platform is optimized for:

- Operational monitoring
- Integration analytics
- Near-real-time visibility
- Lightweight domain-specific models
- Reduced data replication
- Cloud-native extensibility

---

## Federation Seam

The two platforms are connected through Lakehouse Federation.

The warehouse owns enterprise conformed dimensions such as:

- Worker
- Date
- Job
- Company
- Location
- Department
- Line of Service

The lakehouse accesses those dimensions live through governed federated views instead of copying them into Delta Lake.

This pattern preserves a single source of truth while enabling Databricks-based analytics.

---

## High-Level Flow

```
Source Systems
    ↓
Platform 1 Batch Warehouse
    ↓
Conformed Enterprise Dimensions
    ↓
Lakehouse Federation
    ↓
Platform 2 Databricks Lakehouse
    ↓
Gold Reporting Views
    ↓
Power BI / Databricks SQL
    ↓
Business Users
```

---

## Key Architecture Characteristics

- Hybrid batch + micro-batch design
- SQL Server warehouse for enterprise conformed data
- Databricks Lakehouse for integration-specific analytics
- Federation instead of persistent replication for shared dimensions
- Power BI for broad enterprise dashboards
- Databricks SQL for small operational teams
- Data quality designed into both platforms
- Security tiers for sensitive HR data
- CI/CD and monitoring across both platforms

---

## Related Documents

- [Platform 1: Batch Enterprise Warehouse](platform-1-batch-warehouse.md)
- [Platform 2: Cloud Lakehouse](platform-2-lakehouse.md)
- [Federation Pattern](federation-pattern.md)
- [Governance and Operations](governance-and-operations.md)
- [Consumption Pattern](consumption-pattern.md)
- [Design Principles](design-principles.md)

---

## Confidentiality Note

This document contains only sanitized, generic architecture concepts created for portfolio purposes. It does not include proprietary client documentation, confidential data, production code, credentials, internal system names, or real employee/customer data.