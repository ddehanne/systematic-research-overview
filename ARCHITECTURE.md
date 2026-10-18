# Historical Architecture

This document describes the system **as it stood when the project stopped (August 20, 2026)**. It is a historical record, not a design for a running system. The implementation repository is private; signal definitions, portfolio parameters, universe composition, account identifiers and host details are intentionally omitted.

Each component is labelled with the highest assurance level the project records support:

| Level | Meaning |
|---|---|
| **Implemented** | Code present; no documented test of the failure it guards against |
| **Tested** | A documented negative or unit test exercised the failure condition |
| **Operated** | Used in paper and/or live operation |
| **Failed in operation** | Observed not to provide its intended protection |

"Operated" does not imply "tested", and neither implies "reliable".

---

## Data flow

```text
OFFLINE
  Historical market data ──► Research & historical replay ──► signal definition

LIVE (one cycle, launched manually)
  Halt check ──► Market data ──► Targets ──► Order diff ──► Pre-trade checks
                                                                 │
                                                                 ▼
                                    IB Gateway ◄────────── Order submission
                                         │
                                         ▼
                                 Fills ──► Append-only local ledger
                                                   │
                                                   ▼
                                    Per-cycle reconciliation, alerts
                                                   │
                                                   ▼
                                           Operator decisions
```

---

## Boot chain

Starting the trading loop was a deliberate operator action. In the final design, three checks ran in sequence, and any failure stopped the start before an order could be sent.

| Step | What it does | Assurance |
|---|---|---|
| 1. Single-writer lock | Kernel `flock` on a dedicated lock file guarding shared state; a second instance exits immediately | Implemented; operated (paper, live). Introduced after PM-018 |
| 2. Account identity guard | Refuses to start if the connected broker account is not the expected one | **Tested** (refused a paper account), live from August 2026 |
| 3. Reconciliation gate | Fresh read-only broker read compared with every local state file; exits on unreadable files, unreachable broker, divergence or stale snapshots | **Operated** in paper: refused starts several times; the two cases documented in detail were stale local snapshots after outages |

**Trade-off:** no automatic restart path exists. After any outage, trading resumed only when the operator restarted it. In one documented case, a weekend gateway outage followed by a blocked restart kept the paper system down for more than a day.

---

## Per-cycle execution

| Component | Assurance | Notes |
|---|---|---|
| Operator halt check (top of cycle) | Implemented | Added after the live incident in which the halt flag was never read by the loop. Freezes the cycle; no negative test proving it blocks order submission is recorded |
| Target → order diff with drift threshold | Operated | Unchanged targets produce no orders; a unit test for this exists |
| Whole-share orders, regular-hours only | Implemented; operated | Fractional shares were unavailable to the account |
| Per-order notional cap (entries) | Implemented | Never triggered |
| Hard stop | Operated; **failed in operation** | Once could not be sent over a dead connection (~14 min unprotected); once was partly undone when the rebalance step re-entered the same name in the same cycle |
| Stopped-symbol block (after the above) | Implemented | No recorded test |
| Order counter | **Failed in operation** | Counted order intentions, not broker acknowledgements; unchanged at shutdown |
| Ledger write | Operated; **failed in operation** | Written only on the current session's submission path; a fill from an order that most likely survived an earlier session was missed |
| Cancel outstanding orders at boot | Implemented | Added after that incident; no recorded negative test |
| Read-failure handling ("failed read ≠ empty result") | Tested at some sites | Applied at four sites: two passed negative tests on the real path, one unit-verified, coverage of the remainder partial |
| Per-cycle reconciliation | Operated; **failed in operation** | Reported "OK" on a cycle where orders could not be sent; a divergence status was not propagated to alerts |
| Per-cycle automatic position correction | Implemented; suspended by the halt flag | Descends from the June remediation (PM-015). It can place orders to correct a ledger/broker difference, which contradicts the fail-closed boot doctrine. It was still present at shutdown |

---

## Monitoring

| Component | Assurance | Notes |
|---|---|---|
| Chat bot (alerts and commands) | Operated | The halt command wrote a flag the loop did not read; part of the bot's control logic still referred to a service that no longer existed; emergency commands were never tested. The documented emergency stop was terminating the process |
| Scheduled health probe (every 10 min) | Operated; **failed in operation** | Checked process presence, not function, and reported "up" while the loop held a dead connection. A heartbeat-based check was specified, but it was not in the last code snapshot |
| Daily operator check | Operated | Manual, read-only |
| Metrics exporter and dashboards | Operated in paper (April–July) | The exporter was stopped on July 9 and its absence went unnoticed for 34 days; the end-of-day report read the missing metrics as zero |
| Live performance reporting | **Failed in operation** | Initially computed live returns against the paper capital base; fixed for the main reports, not for every consumer |

---

## Restart doctrine

Between June and July, eight independent mechanisms were found that could start or restart the trading process:

- a cron watchdog;
- service restart policies;
- a supervisor unit;
- a control API;
- a second service unit;
- bot commands;
- an alerter that restarted the loop on a stale cycle;
- a scheduled daily restart.

Two of them were found only after the first cleanup ([PM-018](docs/PM-018.md), then PM-019).

All eight were removed or disabled. The resulting doctrine:

- detection may be automatic, starting is not;
- an alerting component alerts and never restarts anything;
- on uncertain state, the boot chain refuses to start.

The per-cycle automatic correction listed above is the known exception that was not resolved.

---

## Testing

- **Unit and component tests:** the private repository contains about 180 test files. Their pass/fail state at shutdown was not re-established for this document. Several depend on host-specific paths.
- **Fault-injection harness:** [fault-injection-harness](https://github.com/ddehanne/fault-injection-harness), a public, standalone, paper-environment harness with five deterministic fault classes: process kill during ledger write, broker disconnect, concurrent instances, phantom position and clock skew. The concurrent-instance class was later reproduced in paper operation (PM-018).
- **Negative tests on the real path:** documented for the account identity guard, account scoping on a multi-account gateway, and two read-failure sites.
- **Not tested:** the revised halt, order cancellation at boot, the notional cap, and the end-to-end live close path.

---

## Related public projects

[verified-ledger](https://github.com/ddehanne/verified-ledger) (Rust) and [execution-journal](https://github.com/ddehanne/execution-journal) (C++20) are separate projects that model ledger and execution-journal invariants in isolation. They share no code with this pipeline. Their claims and evidence are documented in their own repositories and are not part of this case study.

---

## State at shutdown (August 20, 2026)

The loop was halted by the operator. Open technical items included:

- no round-trip broker liveness check on the order path;
- ledger writes tied to the submission path;
- reconciliation status not alerting;
- the halt never negative-tested;
- the per-cycle automatic correction still present.

None of these were resolved; the project stopped because the research result did not justify further work (see [README](README.md)).

## Incident documents in this repository

- [docs/PM-015-INCIDENT.md](docs/PM-015-INCIDENT.md): duplicate loop instances (paper, June)
- [docs/PM-016-ZOMBIE-CGROUP.md](docs/PM-016-ZOMBIE-CGROUP.md): processes surviving a service stop (paper, July)
- [docs/INFRA-001-SYSTEMD-LIFECYCLE.md](docs/INFRA-001-SYSTEMD-LIFECYCLE.md): service stop hardening (July; later superseded)
- [docs/PM-018.md](docs/PM-018.md): automatic restart paths and duplicate liquidation (paper, July)
