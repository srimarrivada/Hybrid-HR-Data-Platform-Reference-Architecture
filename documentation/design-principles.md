# Design Principles

## Overview

This document summarizes the main design principles behind the Hybrid HR Data Platform Reference Architecture.

The architecture is based on a hybrid model:

- Platform 1 provides enterprise-grade batch warehouse capabilities.
- Platform 2 provides cloud lakehouse and integration analytics capabilities.
- Lakehouse Federation connects both platforms without unnecessary replication.

---

## Principle 1: Keep the Enterprise Warehouse as the Source of Truth

The batch warehouse owns enterprise conformed dimensions and highly governed historical analytics.

This ensures:

- Consistent worker context
- Consistent organization context
- Consistent KPIs
- Historical reporting accuracy
- Strong security controls

---

## Principle 2: Extend, Do Not Replace

The lakehouse extends the enterprise warehouse.

It should not duplicate everything already mastered in the warehouse.

Use the lakehouse for:

- Integration-specific analytics
- Operational monitoring
- Event-driven use cases
- Micro-batch use cases
- Domain-specific Gold models

---

## Principle 3: Use Federation Before Replication

If the lakehouse needs conformed dimensions, query them through federation first.

Replicate only if:

- Performance requires it
- Concurrency requires it
- SLA requires it
- Federation query cost becomes too high

Materialization should be explicit, not accidental.

---

## Principle 4: Facts Stay Local to Where Data Originates

Existing enterprise workforce facts remain in the warehouse.

Integration-specific facts remain in the lakehouse.

This reduces:

- Cross-platform movement
- Duplicate processing
- Sync complexity
- Ownership ambiguity

---

## Principle 5: Model Only Where Analytical Value Exists

Not every integration needs a dimensional model.

Build Gold models when:

- There is a clear analytical audience
- There is historical value
- There are business questions to answer
- The model enables new insight

Skip dimensional modeling when:

- The flow is operational only
- Monitoring is sufficient
- No broad analytical audience exists
- Data already exists in another conformed source

---

## Principle 6: Separate Raw, Cleaned, Modeled, and Consumed Layers

Use Medallion Architecture in the lakehouse:

```text
Bronze → Silver → Gold → Reporting Views
```

Each layer has a purpose:

- **Bronze:** raw audit trail
- **Silver:** cleaned and validated data
- **Gold:** analytics-ready dimensional model
- **Reporting Views:** semantic consumption layer

---

## Principle 7: Treat Data Quality as Architecture

DQ is not an afterthought.

DQ should be designed into:

- Source capture
- Transformation
- Modeling
- Reconciliation
- BI consumption
- Monitoring

Use blocking DQ where publishing bad data is unacceptable.

Use alerting DQ where operational visibility should continue.

---

## Principle 8: Reconciliation Should Not Mask Bad Source Data

Reconciliation should fix timing gaps.

It should not hide source quality problems.

Examples:

- New hire unresolved worker key may be expected temporarily.
- Existing employee unresolved key should be treated as a DQ issue.
- External applicants should not be forced into worker dimension reconciliation.

---

## Principle 9: Use the Right BI Tool for the Audience

Use Power BI when:

- The audience is broad.
- Business users need governed dashboards.
- RLS and advanced visuals are needed.
- Daily refresh is acceptable.
- Enterprise sharing and workspace governance are important.

Use Databricks SQL when:

- The audience is small and technical.
- Live lakehouse monitoring is needed.
- Users already work in Databricks.
- Operational freshness matters.
- Simple KPIs and tables are sufficient.

---

## Principle 10: Secure by Design

Sensitive HR data should be protected through:

- Schema separation
- Least privilege
- Role-based access
- Row-level security
- Column masking where needed
- Restricted workspaces
- Audit logging

Highly restricted data should remain in the most governed layer unless there is a justified need.

---

## Principle 11: Design for Recovery

Every critical flow should have a recovery path.

Examples:

- Batch rerun from last successful state
- Re-run from Bronze
- Delta time travel
- Prior published warehouse state
- Reprocess from ODS/history
- Manual reprocess path for operational integrations

Recovery should be designed before failure happens.

---

## Principle 12: Make Architecture Observable

Architectures should be monitored.

Monitor:

- Job success / failure
- Runtime duration
- Data freshness
- DQ status
- BI refresh
- Source volume
- Federation performance
- Error patterns

Observability should support both operations and root-cause analysis.

---

## Final Summary

The hybrid architecture is designed around one central idea:

> Use the warehouse for governed enterprise truth, use the lakehouse for integration-specific agility, and connect both through federation to avoid unnecessary replication.

---

## Confidentiality Note

This document is a sanitized reference architecture artifact and does not describe any proprietary client implementation.