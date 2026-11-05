# ADR-0011: Defense-in-Depth Security for Sensitive HR Data

- **Status:** Accepted
- **Scope:** Both
- **Diagram marker:** D11

## Context
HR data ranges from general workforce attributes to restricted (demographics) and highly restricted (compensation, national identifiers) data. It flows through two platforms, a federation seam, BI tools and downstream feeds. A single control layer is one misconfiguration away from exposure.

## Decision
Layer the controls so that no single failure exposes sensitive data.

**Platform 1 (warehouse)**
1. **Schema separation by sensitivity tier**, with explicit GRANT / DENY per role. → [Security tiers](../architecture/governance-and-operations.md#security-tiers)
2. **Row-level security** in the BI layer on top of the schema controls.
3. **Highly restricted facts are queried through DirectQuery**, so they stay in the governed source and are never copied into the BI layer. → [D6](0006-bi-tool-by-audience.md)

**Seam and Platform 2 (lakehouse)**

4. **Restricted data stays in Platform 1.** Federated views expose only approved, non-restricted columns. → [D4](0004-federation-over-replication.md)
5. **Catalog permissions** (e.g., Unity Catalog) on catalogs, schemas, views and connections, with column masking where needed.
6. **Raw layers are restricted:** Bronze, which may hold personal data as received, is accessible only to platform engineers. → [D12](0012-bronze-captures-transient-sources.md)

**Everywhere**

7. **Data minimization** for downstream feeds and curated views: each consumer gets only the fields its contract needs.
8. **Encryption** in transit and at rest; sensitive file feeds are encrypted and delivered through managed file transfer.
9. **Least-privilege, read-only service accounts**, with secrets kept in a vault.
10. **Audit logging** of access to restricted data.

## Alternatives Considered
| Option | Why Not Chosen |
|---|---|
| Row-level security only | A single layer; bypassed by direct database access. |
| Column masking only | Masked data still travels to every layer; one masking misconfiguration exposes it. |
| Separate physical databases per tier | Stronger isolation, but duplicates conformed dimensions and the pipelines that build them. |

## Consequences
**Positive**
- A failure in one layer doesn't expose data.
- Sensitive data lives in as few places as possible.
- A clear audit story: who can see what, and who did. → [NFR-05](../nfrs.md)

**Negative / Trade-offs**
- Role mappings must be kept in sync across the warehouse, catalog and BI layers.
- More security testing for every release.
- Some analyses need approved access to the restricted tier, which adds process.

## Guardrails
- **Grants are defined before data lands** on any new platform, object or feed. → [R-05](../risks.md)
- **New columns are classified** by sensitivity before they are exposed anywhere.
- **Federated view columns are reviewed** whenever a view changes. → [R-04](../risks.md)
- **Access reviews** on a regular schedule; access outside the standard process is flagged.
- The **same role model** is used on both platforms, so a user's access means the same thing everywhere.

## Revisit When
- Lakehouse governance (fine-grained access, masking, auditing) is mature enough to hold restricted data, which would enable a single governed platform. → [D1](0001-two-platforms-not-one.md)
- New regulations or data categories need a new sensitivity tier.

## Related
- [Design Principle 13](../design-principles.md#13-secure-by-design)
- [Governance and Operations](../architecture/governance-and-operations.md)
- [Federation Pattern](../architecture/federation-pattern.md)
- [Risks](../risks.md)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*