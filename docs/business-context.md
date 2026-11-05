# Business Context

The architecture starts from who uses the data and what they need to know. Each design decision traces back to one or more of the personas and questions below.

## Personas

| # | Persona | Needs | Primary channel |
|---|---|---|---|
| P1 | HR leadership | Workforce trends, attrition, hiring vs plan | Enterprise dashboards |
| P2 | HR business partners | Team-level headcount, movement, leave | Enterprise dashboards (row-level security) |
| P3 | Talent acquisition | Funnel conversion, time-in-stage, requisition health | Lakehouse dashboards |
| P4 | Learning & development | Enrollment, session utilization, coverage gaps | Lakehouse dashboards |
| P5 | Integration operations | Event status, overdue notifications, failures | Live ops dashboard |
| P6 | Restricted-data users | Compensation, demographics (governed) | Restricted workspaces |

## Representative Business Questions

| # | Question | Personas | Answered by | Enabled by |
|---|---|---|---|---|
| BQ1 | How is headcount trending by business unit, location and level? | P1, P2 | Platform 1 | [D10](adr/0010-kimball-dimensional-model.md), [D13](adr/0013-fact-design-patterns.md) |
| BQ2 | Where is attrition concentrated, and is it voluntary or involuntary? | P1, P2 | Platform 1 | [D2](adr/0002-temporal-ods-family.md), [D10](adr/0010-kimball-dimensional-model.md) |
| BQ3 | Where do candidates drop out of the hiring funnel, and how long does each stage take? | P3 | Platform 2 | [D7](adr/0007-selective-medallion.md), [D12](adr/0012-bronze-captures-transient-sources.md) |
| BQ4 | Which populations are under-covered by required training? | P4 | Platform 2 + federation | [D4](adr/0004-federation-over-replication.md) |
| BQ5 | Are onboarding notifications reaching downstream systems on time? | P5 | Platform 2 (live) | [D5](adr/0005-blocking-vs-alerting-dq.md), [D6](adr/0006-bi-tool-by-audience.md) |
| BQ6 | How do pay and representation compare across comparable groups? | P6 | Platform 1 (restricted) | [D11](adr/0011-defense-in-depth-security.md) |

## What This Means for the Architecture

- **Different freshness needs:** Leadership and HR partners work from daily, reconciled data (Platform 1) while operations needs data that is minutes or hours old (Platform 2). → [D1](adr/0001-two-platforms-not-one.md)
- **History matters:** Trends and attrition need accurate point-in-time history, including back-dated corrections. → [D2](adr/0002-temporal-ods-family.md), [D8](adr/0008-change-detection-strategy.md)
- **Some sources don't keep history:** Funnel and training trends exist only if the platform captures them. → [D12](adr/0012-bronze-captures-transient-sources.md)
- **Shared worker context:** Lakehouse questions still need the same worker and organization view as the warehouse. → [D4](adr/0004-federation-over-replication.md)
- **Sensitive data:** Compensation and demographics have a small, governed audience, so they stay in the most controlled platform. → [D11](adr/0011-defense-in-depth-security.md)
- **Different audiences:** A broad governed audience, a small operations team and lakehouse business users each need a different tool and refresh mode. → [D6](adr/0006-bi-tool-by-audience.md)

## Related
- [Non-Functional Requirements](nfrs.md) 
- [Source Assessment](source-assessment.md) 
- [ADR Index](adr/README.md)