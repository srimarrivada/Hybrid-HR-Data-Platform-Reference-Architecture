# Federation Pattern

## Overview

This document explains the federation pattern used between the enterprise warehouse and the cloud lakehouse.

The key principle is:

> Keep enterprise conformed dimensions in the warehouse and allow the lakehouse to query them live through governed federation.

This avoids unnecessary replication while preserving consistent business context.

---

## Why Federation Is Used

Without federation, the lakehouse would need to replicate enterprise dimensions such as:

- Worker
- Date
- Job
- Company
- Location
- Department
- Line of Service

This creates risk:

- Duplicate dimension copies
- Synchronization lag
- Reconciliation overhead
- Inconsistent worker or organization context
- Additional data movement
- More governance complexity

Federation avoids these issues by enabling zero-copy access to approved enterprise objects.

---

## Federation Role in the Architecture

Federation connects the enterprise warehouse and the cloud lakehouse.

```text
Enterprise Warehouse
    ↓
Conformed Dimensions
    ↓
Federated Views
    ↓
Databricks Gold Models
```

The lakehouse uses federation for:

- Worker identity
- Worker organization context
- Surrogate key lookups
- BI-ready Gold views
- Cross-domain analytics

---

## Common Federated Views

Typical federated views include:

1. Worker Identity View
2. Worker Organization Context View

---

## Worker Identity View

The Worker Identity View provides a reusable worker identity layer for the lakehouse.

### Provides

- Worker key
- Worker ID
- Worker name
- Work email
- Hire date
- Employee status
- Active worker flag

### Used By

- New Hire analytics
- Recruiting analytics
- Training analytics
- Worker key resolution
- Recruiter lookup
- Learner / instructor lookup
- BI-ready Gold views

### Purpose

This view centralizes worker identity and avoids repeated direct access to the full worker dimension.

It also reduces the risk of inconsistent filtering or SCD misuse by exposing only the required current-worker attributes.

---

## Worker Organization Context View

The Worker Organization Context View provides worker identity plus current organization context.

### Provides

- Worker identity
- Line of Service
- Management level
- Company
- Location
- Department
- Job family

### Used By

- Training coverage analytics
- Organization-based slicing
- Cross-domain workforce analytics
- Dashboards that need worker organization context

### Purpose

This view supports business questions such as:

- What is training coverage by Line of Service?
- What is training coverage by level or location?
- Which worker populations are underrepresented in a program?
- How do integration events align to current worker organization context?

This view may be heavier than the basic Worker Identity View because it typically joins multiple conformed dimensions.

---

## Local vs Federated Ownership

The architecture separates enterprise-owned objects from lakehouse-owned objects.

---

## Enterprise-Owned Objects

These remain in the warehouse:

- Worker
- Date
- Job
- Company
- Location
- Department
- Line of Service
- Action Reason

These are enterprise conformed dimensions and should remain the source of truth.

---

## Lakehouse-Owned Objects

These remain local to Databricks:

- Event type
- Notification status
- Candidate stage
- Requisition
- Session
- Enrollment status
- Integration-specific facts

These objects are specific to the lakehouse integration domain and do not need to be mastered in the enterprise warehouse.

---

## Federation Usage Patterns

Federation is used in two main patterns:

1. ETL-time lookup
2. Reporting-time view

---

## ETL-Time Lookup

ETL-time lookup is used when a Gold fact needs to resolve a key.

### Pattern

```text
Query federated dimension
    ↓
Resolve surrogate key
    ↓
Store key in Gold fact
    ↓
BI uses Gold fact/view
```

### Best For

- Surrogate key resolution
- Low to moderate lookup volume
- Controlled pipeline execution
- Avoiding repeated dashboard-time federation queries

### Example

A Gold fact may need to resolve:

- Worker key
- Job key
- Company key
- Location key
- Department key
- Action reason key

Once the surrogate keys are resolved and stored in the fact, dashboards do not need to repeatedly query those source dimensions.

---

## Reporting-Time View

Reporting-time federation is used when a report needs live organizational context.

### Pattern

```
Gold fact
    ↓
Join federated view
    ↓
Gold reporting view
    ↓
Power BI or Databricks SQL
```

### Best For

- Current organization context
- Business-friendly reporting views
- Scheduled dashboard refresh
- Moderate-volume report queries
- Use cases where live conformed context is required

### Consideration

Reporting-time federation should be used carefully. Heavy federation joins at user-click time can impact performance and source system load.

---

## When to Materialize

Federation is not always the answer.

Materialization may be considered when:

- Query concurrency is high
- Federation query is too slow
- Dashboard SLA is strict
- Large unfiltered scans are repeated
- Source system load becomes a concern
- BI refresh windows become too long
- Interactive dashboard queries repeatedly trigger heavy federation joins

Materialization should be an explicit design decision, not the default.

---

## Federation Guardrails

Federation should follow these guardrails:

- Use approved views, not unrestricted schemas
- Limit exposed columns
- Centralize current-row filters
- Avoid heavy joins at user-click time
- Monitor query latency
- Monitor source system load
- Avoid access to restricted or highly restricted data unless authorized
- Use least-privilege service accounts
- Avoid turning federation into uncontrolled cross-system querying

---

## Security Controls

Federation should use:

- Unity Catalog permissions
- Read-only service accounts
- Secret management
- Network controls
- Audit logging
- Least-privilege access
- Column and row restrictions where required
- Controlled access to catalogs, schemas, views, and connections

Security must be enforced on both sides:

1. The warehouse source system
2. The lakehouse consuming platform

---

## Observability

Federated access should be monitored.

Recommended monitoring areas:

- Query latency
- Failed federation queries
- Source system load
- BI refresh duration
- Dashboard query time
- SQL Warehouse performance
- Access errors
- DQ check status
- SLA impact

If federation introduces performance or operational risk, the architecture should be reviewed and materialization should be considered.

---

## Design Principles

1. Enterprise conformed dimensions remain in the warehouse.
2. Databricks does not replicate conformed dimensions by default.
3. Federation is used for governed access.
4. Local dimensions remain in the lakehouse when integration-specific.
5. Facts stay local to where data originates.
6. Materialization is explicit and justified.
7. Federation should reduce duplication, not create hidden complexity.
8. Reporting-time federation should be limited to appropriate use cases.
9. Surrogate keys should be resolved and stored in facts where practical.
10. Security and performance guardrails must be designed into federation patterns.

---

## Confidentiality Note

This document is a sanitized reference architecture artifact and does not describe any proprietary client implementation.