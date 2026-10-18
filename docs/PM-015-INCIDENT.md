# PM-015 — Duplicate Trading-Loop Instances (Paper Account)

**Environment:** IBKR paper account (simulated capital). No real capital was involved.
**Detected:** 2026-06-22, 21:00 UTC
**Closed:** 2026-06-25
**Later status:** the remediation approach was superseded in July 2026 (see [PM-018](PM-018.md) and the [architecture](../ARCHITECTURE.md)).

> **Revision note.** An earlier public version of this document contained an illustrative timeline (dates, process IDs, position counts), described the system as trading live, and listed tests and uptime that the project records do not support. That content is withdrawn. This version is limited to the incident record written at the time; where the record is uncertain, it says so.

---

## Summary

Two instances of the trading loop ran at the same time against the same paper account. The broker's view and the local ledger diverged by more than an order of magnitude before the divergence was noticed.

## Timeline (UTC)

| Date | Event | Basis |
|---|---|---|
| 2026-06-18 | One loop instance had been running since this date | Incident record |
| Unknown | A second instance started | Start time not recorded |
| 2026-06-22 21:00 | Divergence detected: broker gross position value ≈ $647K vs ≈ $33K in the local ledger; negative broker cash (margin used in the paper account); per-symbol differences of up to two orders of magnitude | Incident record |
| 2026-06-22 | Operator halt set; the orphaned instance was killed | Incident record |
| 2026-06-24 | Excess positions closed by a newly written automatic reconciliation script (eight orders) | Incident record; commit of the same day |
| 2026-06-25 | Paper account reset at the broker; pipeline restarted | Incident record |

## Root cause

The initial entry recorded the cause as "to be investigated, probably a service restart without a clean stop of the previous instance". The closing entry records it as confirmed: the service manager's restart policy started a new instance without terminating the previous one, and the paper gateway accepted both connections to the same account without rejecting either. The evidence behind that confirmation is not preserved beyond the incident record itself.

## Remediation at the time (June 2026)

1. A launcher-level lock file: the start script refused to start if a recorded process was still alive.
2. Automatic reconciliation at start: compare broker positions with the local ledger and **automatically close** excess positions.
3. A shared broker-position snapshot written each cycle, so that reconciliation did not need a second broker connection.

No test evidence for these June changes is preserved in the records reviewed.

## What happened next

The June remediation did not hold:

- **July 2 ([PM-016](PM-016-ZOMBIE-CGROUP.md)):** stopping the service could leave child processes alive and connected to the broker.
- **July 7–10 ([PM-018](PM-018.md)):** a liquidation script launched twice bypassed the launcher-level lock, and several independent mechanisms kept restarting a service the operator had stopped.

The design then changed direction:

- The single-writer lock moved from the launcher to the shared resource (a kernel `flock`).
- Automatic restarts were removed.
- A reconciliation gate at boot refuses to start on divergent or unverifiable state instead of correcting it automatically.

One inconsistency remained: a per-cycle automatic correction routine descended from this June remediation was still present in the executor when the project stopped (see the architecture document).

## Lessons

- A lock on a launcher protects one entry path; a lock on the resource protects all of them.
- An automatic correction of broker state is itself an order-generating component and needs the same scrutiny as the strategy.
- Two loop instances on the same account were not rejected by the paper gateway; process uniqueness has to be enforced locally.
