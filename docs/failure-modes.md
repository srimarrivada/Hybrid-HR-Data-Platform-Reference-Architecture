# Failure Modes and Recovery

This document describes how the reference architecture behaves when things go wrong, and how operators recover. Each failure mode matches a card (F1–F5) in the **Failure & Recovery** panel of the [detailed diagram](diagrams/svg/hybrid-hr-platform-reference-architecture.drawio.svg).

**Design Goal:** A failure should never cause silent data loss, a partial publish, or double counting. Every failure is either retried automatically or leaves the platform in a known good state that an operator can recover from with documented steps.

| # | Failure | Platform | Worst outcome if not designed for | Designed outcome |
|---|---|---|---|---|
| [F1](#f1-nightly-run-fails-part-way) | Nightly run fails part-way | Platform 1 | Data gap or partial load | Same window re-extracted; resume from failed step |
| [F2](#f2-data-quality-gate-below-threshold) | DQ gate below threshold | Platform 1 | Bad numbers in executive dashboards | Previous version stays live |
| [F3](#f3-integration-record-or-api-call-fails) | Integration record or API call fails | Platform 2 | Lost or duplicated downstream records | Record marked failed; retry or replay |
| [F4](#f4-federation-slow-or-enterprise-warehouse-unavailable) | Federation slow or enterprise warehouse unavailable | Seam | Lakehouse pipelines stall | Last good Gold served; keys reconciled later |
| [F5](#f5-late-or-back-dated-source-changes) | Late or back-dated source changes | Platform 1 | History silently wrong | Corrections captured and merged; history corrected |
---

## F1: Nightly Run Fails Part-way

**Trigger:** A job in the nightly dependency graph fails (source API error, timeout, procedure error, infrastructure issue).

**Detection**
- The scheduler marks the job failed and alerts the operations channel.
- The batch audit log records the failed step and the error.

**Automatic Behavior**
- Dependent jobs **do not start**; milestone gates hold back everything downstream. → [D3](adr/0003-dependency-tiered-batch.md)
- The extraction watermark is **not advanced**, because the post-load step runs only after full success. → [D9](adr/0009-idempotent-loads.md)
- Consumers continue to see the previous night's published data.

**Recovery Steps**
1. Classify the failure using the run log: source issue, pipeline issue or infrastructure issue.
2. Fix the cause (or wait for the source to recover).
3. Restart **from the failed step** using the restart flags; completed steps are not repeated.
4. If the window has already passed, the next scheduled run re-extracts the same window automatically. No manual backfill is needed.
5. Confirm row counts and the DQ gate before publishing (see F2).

**Prevention**
- Retries with backoff on transient API errors; concurrency limits that respect source connection limits.
- Error thresholds that stop a session early instead of loading partial data.
- Idempotency test in CI: every load is run twice and the results compared.

---

## F2: Data Quality Gate Below Threshold

**Trigger:** The quality score before the warehouse publish falls below the threshold, or a critical rule fails (e.g., referential integrity on a core dimension).

**Detection**
- The DQ gate records the score and the failing rules in the DQ history.
- A critical alert raises a ticket; warnings go to the operations channel.

**Automatic Behavior**
- Publishing **stops**. The previous warehouse version stays live. → [D5](adr/0005-blocking-vs-alerting-dq.md)
- BI refreshes, which are triggered by the publish, do not run, so dashboards keep the last good import.
- Dashboards show their data-freshness label, so users can see the data is from the previous cycle.

**Recovery Steps**
1. Review the failing rules in the DQ history and identify the affected entities.
2. Decide: **source issue** (raise with the source owner, wait for a corrected extract) or **pipeline issue** (fix and re-run the affected step).
3. Re-run only the affected steps and the gate; earlier steps don't need to repeat.
4. When the gate passes, publish. The BI refresh follows automatically.
5. If publishing is delayed past business start, notify consumers through the agreed channel.

**Prevention**
- Rules and thresholds held as configuration and reviewed when they fire too often or not at all.
- Trend monitoring of DQ scores to catch slow degradation before it trips the gate.
- Critical rules that block regardless of the overall score.

---

## F3: Integration Record or API Call Fails

**Trigger:** A single record fails to process, or a call to a downstream or upstream API fails (authentication, rate limit, validation error, outage).

**Detection**
- The record's status moves to **FAILED** in the integration tracking table, with the error captured.
- The DQ log and the live operations dashboard show failed and overdue records.

**Automatic Behavior**
- The failed record is isolated; **the rest of the batch continues**.
- Token refresh and retries with backoff handle transient errors.
- The raw payload is already preserved in Bronze, so nothing is lost. → [D12](adr/0012-bronze-captures-transient-sources.md)

**Recovery Steps**
1. Check the error category on the failed records (data error vs system error).
2. **System error:** Wait for the next scheduled retry, or trigger a manual reprocess for the affected records.
3. **Data error:** Correct the data at the source, then reprocess.
4. For wider outages, replay the affected window from Bronze. Processing is idempotent, so records that already succeeded are not sent twice. → [D9](adr/0009-idempotent-loads.md)
5. Confirm on the operations dashboard that the failed and overdue counts are back to normal.

**Prevention**
- Status tracking for every record (pending → success / failed) instead of fire-and-forget.
- Idempotency keys on outbound calls so retries can't create duplicates.
- Alerts on rising failure rates, not only individual failures.

---

## F4: Federation Slow or Enterprise Warehouse Unavailable

**Trigger:** Federated queries from the lakehouse to the warehouse time out, run slowly, or fail because the warehouse is unavailable or under heavy load.

**Detection**
- Federation query latency and failure monitoring.
- Gold pipeline runs fail or take longer than usual.
- The DQ log shows a rise in unresolved worker keys.

**Automatic Behavior**
- The lakehouse keeps serving its **last good Gold tables**. Delta table versions are not overwritten by a failed run.
- Records whose worker can't be resolved get the **unknown-member key** and a DQ entry, rather than being dropped. → [D4](adr/0004-federation-over-replication.md)
- Operational integrations keep flowing; only enrichment with worker context is delayed.

**Recovery Steps**
1. Check warehouse health and whether the nightly publish has finished. Federation reads before publish see the previous day's data.
2. Once the warehouse is available, re-run the failed Gold cycle.
3. Run the scheduled **reconciliation** to resolve unknown-member keys after the warehouse publish.
4. Keys that stay unresolved beyond the expected window are investigated as source issues, not reconciled away → [D5](adr/0005-blocking-vs-alerting-dq.md)

**Prevention**
- Resolve keys at ETL time and store them in facts, so dashboards don't trigger federation queries.
- Lakehouse schedules aligned to after the warehouse publish.
- **Materialize** the most-used federated views if latency or source load stays high. This is an explicit decision, recorded in an ADR.

---

## F5: Late or Back-dated Source Changes

**Trigger:** A change arrives with an effective date in the past (correction, retroactive change), or a future-dated change is entered ahead of its effective date.

**Detection**
- The effective-dated ODS loads changes within its look-back window.
- The future-dated ODS loads changes within its forward window. → [D2](adr/0002-temporal-ods-family.md)

**Automatic Behavior**
- Back-dated corrections are **merged into the current ODS** (insert-only), then flow through change detection like any other change. → [D8](adr/0008-change-detection-strategy.md)
- SCD2 history is **corrected, not overwritten**: affected versions are adjusted and the change stays auditable.
- Future-dated changes are held separately and become current on their effective date; they support pre-hire provisioning without affecting today's reporting.

**Recovery Steps** (when a change falls outside the look-back window)
1. Identify the affected entities and date range.
2. Run a targeted **full extract** for that range into the full ODS.
3. Re-run change detection and the affected dimension and fact loads.
4. Validate the corrected history with point-in-time queries before publishing.

**Prevention**
- Look-back and forward window sizes held as configuration and reviewed against how late changes actually arrive.
- Monitoring of corrections arriving near the edge of the look-back window.
- Merge-back logic that is insert-only and idempotent.

---

## Other Failure Modes (Summary)

| Failure | Detection | Automatic Behavior | Recovery |
|---|---|---|---|
| Expected source file missing | File-arrival check per schedule | Dependent steps wait; alert raised | Request the file; process on arrival; idempotent if it arrives twice |
| Same file delivered twice | File run log | Duplicate rejected | None needed |
| Schema drift in a source | Schema validation in Bronze/Silver | Load stopped for that source; others continue | Update mapping; replay from Bronze |
| BI refresh fails | Refresh monitoring | Dashboards keep the last good import | Re-trigger refresh after the cause is fixed |
| Access misconfiguration | Access reviews, audit logs, user reports | n/a | Correct role mapping; review audit log for exposure → [D11](adr/0011-defense-in-depth-security.md) |

## Operating Principles
1. **Fail closed for enterprise data, fail open for operations:** Block bad warehouse publishes; keep integrations flowing and reconcile later.
2. **Never advance state on failure:** Watermarks, versions and publishes move forward only after success.
3. **Every load is safe to re-run:** Recovery should never need a judgement call about duplicates.
4. **Keep the raw data:** Replay from preserved raw data instead of re-requesting it from sources.
5. **Make staleness visible:** Users should always know how current their data is.

## Related
- [Governance and Operations](architecture/governance-and-operations.md) 
- [Risks](risks.md) 
- [ADR Index](adr/README.md)

---

*This document describes a generalized reference architecture and does not describe any specific organization's implementation.*