# Federation Pattern

## Overview

This document explains the federation pattern between the enterprise warehouse (Platform 1) and the cloud lakehouse (Platform 2). Products named (e.g., Databricks Lakehouse Federation, Unity Catalog) are examples; the pattern applies to equivalent technologies.

> Keep enterprise conformed dimensions in the warehouse and let the lakehouse read them in place through governed federation. → [D4](../adr/0004-federation-over-replication.md)

This avoids unnecessary replication while keeping business context consistent across both platforms.

---

## Why Federation Is Used

Without federation, the lakehouse would need to replicate enterprise dimensions such as:

- Worker
- Date
- Job
- Company
- Location
- Department
- Business Unit

Replication creates risk:

- Duplicate dimension copies
- Synchronization lag
- Reconciliation overhead
- Inconsistent worker or organization context
- Additional data movement
- More governance complexity

Federation avoids these issues through zero-copy access to approved enterprise objects.

---

## Federation Role in the Architecture

```text
Enterprise warehouse (Platform 1)
    ↓
Conformed dimensions
    ↓
Governed federated views
    ↓
Lakehouse Gold models (Platform 2)
```

The lakehouse uses federation for:

- Worker identity
- Worker organization context
- Surrogate key lookups
- BI-ready Gold views
- Cross-domain analytics

Federation carries only conformed context. Each platform still ingests its own sources.

---

## Common Federated Views

Keep the number of federated views small and their purpose clear. Typical views:

1. [Worker Identity View](#worker-identity-view): Lightweight, used for most lookups
2. [Worker Organization Context View](#worker-organization-context-view): Heavier, used only where organization context is needed

### Worker Identity View

A reusable worker identity layer for the lakehouse.

**Provides**
- Worker key
- Worker ID
- Worker name
- Work email
- Hire date
- Employee status
- Active worker flag

**Used by**
- New hire analytics
- Recruiting analytics
- Training analytics
- Worker key resolution
- Recruiter lookup
- Learner / instructor lookup
- BI-ready Gold views

**Purpose:** Centralizes worker identity and avoids repeated direct access to the full worker dimension. Because it exposes only the current-worker attributes that are needed, it reduces the risk of inconsistent filtering or SCD misuse.

### Worker Organization Context View

Worker identity plus current organization context.

**Provides**
- Worker identity
- Business unit
- Management level
- Company
- Location
- Department
- Job family

**Used by**
- Training coverage analytics
- Organization-based slicing
- Cross-domain workforce analytics
- Dashboards that need worker organization context

**Supports questions such as**
- What is training coverage by business unit?
- What is training coverage by level or location?
- Which worker populations are underrepresented in a program?
- How do integration events align to current worker organization context?

This view joins several conformed dimensions, so it is heavier than the Worker Identity View. Use it only where organization context is actually needed.

---

## Local vs Federated Ownership

The architecture separates enterprise-owned objects from lakehouse-owned objects.

### Enterprise-Owned Objects (Platform 1)

These remain in the warehouse as the source of truth:

- Worker
- Date
- Job
- Company
- Location
- Department
- Business Unit
- Action Reason

### Lakehouse-Owned Objects (Platform 2)

These remain local to the lakehouse because they are specific to the integration domain:

- Event type
- Notification status
- Candidate stage
- Requisition
- Session
- Enrollment status
- Integration-specific facts

They do not need to be mastered in the enterprise warehouse.

---

## Federation Usage Patterns

Federation is used in two ways. **ETL-time lookup is preferred**; reporting-time joins are the exception.

### 1. ETL-Time Lookup (Preferred)

Used when a Gold fact needs to resolve a key.

```text
Query federated view
    ↓
Resolve surrogate key
    ↓
Store key in Gold fact
    ↓
BI reads the Gold fact / view
```

**Best for:**
- Surrogate key resolution
- Low to moderate lookup volume
- Controlled pipeline execution
- Avoiding repeated federation queries from dashboards

**Example:** A Gold fact may resolve worker, job, company, location, department and action reason keys. Once the keys are stored in the fact, dashboards never query the source dimensions.

**Unresolved keys:** If a worker can't be resolved yet (e.g., a new hire who isn't in the warehouse until its next publish), the record gets the **unknown-member key** and a DQ entry rather than being dropped. A scheduled reconciliation resolves it after the warehouse publish. Keys that stay unresolved are treated as DQ issues, not reconciled away. → [D5](../adr/0005-blocking-vs-alerting-dq.md)

### 2. Reporting-Time View (Limited)

Used when a report needs live organization context.

```text
Gold fact
    ↓
Join federated view
    ↓
Gold reporting view
    ↓
BI (e.g., Power BI) or live SQL dashboard (e.g., Databricks SQL)
```

**Best for:**
- Current organization context
- Business-friendly reporting views
- Scheduled dashboard refresh
- Moderate-volume report queries
- Use cases where live conformed context is required

**Consideration:** Heavy federation joins at user-click time can hurt performance and load the source system. Prefer scheduled refreshes, which read federated context once per refresh rather than on every click. → [D6](../adr/0006-bi-tool-by-audience.md)

---

## When to Materialize

Federation is not always the answer. Consider materializing when:

- Query concurrency is high
- Federation queries are too slow
- Dashboard freshness or response targets are strict
- Large unfiltered scans repeat
- Source system load becomes a concern
- BI refresh windows become too long
- Interactive dashboard queries repeatedly trigger heavy federation joins

Materialization should be an explicit, recorded design decision (a new ADR), not the default.

---

## When Federation Fails

If federated queries are slow or the warehouse is unavailable:

- The lakehouse keeps serving its **last good Gold tables**.
- Unresolved workers get the unknown-member key and are reconciled later.
- Operational integrations keep flowing; only enrichment with worker context is delayed.

→ [F4: Federation slow or enterprise warehouse unavailable](../failure-modes.md#f4-federation-slow-or-enterprise-warehouse-unavailable)

---

## Federation Guardrails

- Use approved views, not unrestricted schemas
- Limit exposed columns to what consumers need
- Centralize current-row filters in the views
- Avoid heavy joins at user-click time
- Monitor query latency and source system load
- Never expose restricted or highly restricted data through federation → [D11](../adr/0011-defense-in-depth-security.md)
- Use least-privilege, read-only service accounts
- Don't let federation become uncontrolled cross-system querying

---

## Security Controls

Federation should use:

- Catalog permissions (e.g., Unity Catalog)
- Read-only service accounts
- Secret management
- Network controls
- Audit logging
- Least-privilege access
- Column and row restrictions where required
- Controlled access to catalogs, schemas, views, and connections

Security must be enforced on **both** sides: **the enterprise warehouse source system** and **the lakehouse consuming platform**. → [D11](../adr/0011-defense-in-depth-security.md)

---

## Observability

Monitor federated access for:

- Query latency
- Failed federation queries
- Source system load
- BI refresh duration
- Dashboard query time
- Lakehouse SQL compute performance
- Access errors
- DQ check status (especially unresolved keys)
- Impact on freshness targets

If federation introduces performance or operational risk, review the architecture and consider materialization. → [R-01](../risks.md)

---

## Federation Rules

These apply [Design Principles](../design-principles.md) 3 and 4 to the federation seam:

1. Enterprise conformed dimensions remain in the warehouse.
2. The lakehouse does not replicate conformed dimensions by default.
3. Federation is used for governed access only.
4. Integration-specific dimensions remain local to the lakehouse.
5. Facts stay local to where the data originates.
6. Materialization is explicit and justified.
7. Federation should reduce duplication, not create hidden complexity.
8. Reporting-time federation is limited to appropriate use cases.
9. Surrogate keys are resolved and stored in facts where practical.
10. Security and performance guardrails are designed in from the start.

---

## Related Documents

- [Architecture Overview](architecture-overview.md)
- [Platform 1: Batch Enterprise Warehouse](platform-1-batch-warehouse.md)
- [Platform 2: Cloud Lakehouse](platform-2-lakehouse.md)
- [Consumption Pattern](consumption-pattern.md)
- [D4: Federation Over Replication](../adr/0004-federation-over-replication.md)
- [Risks](../risks.md)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*