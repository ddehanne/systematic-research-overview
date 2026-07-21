# Architecture Design Document

## System Overview

QS1-XSMR is a Python research-to-execution pipeline organized as three
loosely coupled layers:

- Signal research operates independently from execution
- Execution is gated by integrity controls at boot and per cycle
- Failures are contained by a fail-closed doctrine: on any state
  inconsistency the system refuses to trade

Two public companion projects (Rust, C++20) carry the same correctness
contract into independently verifiable artifacts — shared invariants,
**no code integration** with this pipeline. See "Verification
Companions" below.

---

## Data Flow Pipeline

```text
┌─────────────────────────────────────────────────────────────┐
│                    EXTERNAL DATA SOURCES                    │
│   IBKR historical OHLCV │ dated holdings (universe) │ FRED  │
└──────────────────────────────┬──────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────┐
│              DATA INGESTION & UNIVERSE (Python)             │
│  • Point-in-time universe construction from dated holdings  │
│    (membership is a function of time — no silent fallback:  │
│    missing coverage raises a loud error; see Data Audit     │
│    section)                                                 │
│  • Feature computation on the admissible universe           │
└──────────────────────────────┬──────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────┐
│         LAYER 1: SIGNAL RESEARCH (Python)                   │
│  Cross-sectional scoring → candidate ranking                │
│  Pre-registered evaluation; audited window 2021+ only       │
│  Classification: regime-conditional (see README)            │
└──────────────────────────────┬──────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────┐
│    LAYER 2: EXECUTION & ORCHESTRATION (Python)              │
│                                                             │
│  BOOT CHAIN (the only start path — deliberate operator      │
│  action):                                                   │
│    1. Kernel single-writer lock (flock on the ledger        │
│       resource) — second instance exits immediately         │
│    2. Broker reconciliation gate — fresh broker read        │
│       compared against ALL local state sources; any         │
│       divergence or unreachable broker → exit non-zero,     │
│       no trading (fail-closed)                              │
│    3. Trading loop starts                                   │
│                                                             │
│  PER CYCLE:                                                 │
│    • Load signal → allocation → target portfolio            │
│    • Diff: orders = target − current (idempotent: same      │
│      target twice ⇒ zero orders)                            │
│    • Order construction: lot quantization, min-notional     │
│    • Submission & fill tracking via ib_insync               │
│    • Append fills to ledger.jsonl (append-only)             │
└──────────────────────────────┬──────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────┐
│    LAYER 3: RISK & CONSTRAINT ENFORCEMENT                   │
│  Pre-trade constraints (notional, concentration,            │
│  tradability filters incl. earnings blacklist — hot-read    │
│  from disk each cycle)                                      │
│  [Detailed parameters proprietary]                          │
└──────────────────────────────┬──────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────┐
│      MONITORING & OBSERVABILITY (read-only)                 │
│  • Prometheus metrics, Grafana dashboards                   │
│  • Telegram alerting — alerts NEVER start or restart        │
│    anything (doctrine: see PM-018/PM-019)                   │
│  • Daily operator health check (read-only script)           │
└─────────────────────────────────────────────────────────────┘
```

---

## State Model

### Sources of local state

The pipeline maintains multiple local state artifacts; the **fill
ledger is the reconstruction authority**, and all sources are checked
against the broker at boot:

| Artifact | Role | Written by |
|---|---|---|
| `ledger.jsonl` | Append-only fill history — authoritative for position reconstruction | executor |
| executor position cache | Fast-path current positions | executor |
| entry price cache | Cost-basis tracking | executor |
| broker-view snapshot | Last known broker positions | cycle refresh |

**Invariant (learned from incident history):** any state file not
checked against the broker at boot will eventually diverge silently.
The reconciliation gate therefore compares **all** sources, not just
the ledger.

### Target-to-orders reconciliation (per cycle)

```text
current positions (rebuilt from ledger)
        +
target portfolio (from allocation)
        ↓
orders = target − current
        ↓
zero diff ⇒ zero orders (idempotence)
```

Two reconciliations exist at two levels and must not be confused:
- **Boot reconciliation** (integrity): local state vs broker; any
  unresolved divergence forbids trading.
- **Cycle reconciliation** (production): target vs current; produces
  the order list.

---

## Operational Doctrine — Fail-Closed

The system's restart doctrine evolved through documented incidents
(PM-015 → PM-020) and is deliberately **human-in-the-loop**:

- Detection is automatic; **correction is not.** On inconsistency the
  system refuses to act (gate exits non-zero, zero orders).
- Recovery is a deliberate operator action through the locked boot
  chain.
- Alerting components alert; they never start anything. Eight distinct
  autonomous restart mechanisms were identified and removed (cron
  watchdogs, systemd restart policies, supervisor units, control-API
  endpoints, bot commands, alerter auto-restart, scheduled restarts);
  the full process tree is audited for start paths.
- systemd hardening: `KillMode=control-group`, bounded stop timeout,
  start-rate limiting, no automatic restart (see INFRA-001, PM-016).

The earlier "self-healing / zero manual intervention" direction
documented in PM-015 was falsified in production and superseded — the
evolution is documented, not erased (see PM-015 addendum).

---

## Data Audit (methodological control)

A self-initiated audit found that the point-in-time universe module
silently substituted the current universe for dates before holdings
coverage began — projecting survivors backwards (look-ahead +
survivorship). The pre-2021 window of the IC series was invalidated;
the fallback was replaced with a loud error; **all published figures
come from the audited 2021+ window.** This audit is the reason the
strategy is classified regime-conditional rather than unconditional.

---

## Failure Modes & Recovery

| Failure | Detection | Response |
|---|---|---|
| Second instance starts | kernel flock | new instance exits immediately (exit 3) |
| Local/broker divergence at boot | reconciliation gate | refuse to trade (exit 4); operator investigates; state refreshed from broker read-only session only after root-cause |
| Broker unreachable at boot | reconciliation gate | fail-closed — no trading on unverifiable state |
| Broker API timeout in session | bounded retry, backoff | bounded attempts; failure logged; no unbounded resubmission |
| Gateway restart (weekly, external) | daily operator check incl. listener probe | operator-driven recovery; known limitation: in-session reconnection is a planned hardening item |
| Process kill mid-write | append-only ledger + reconstruction | positions rebuilt from ledger at next boot; verified against broker by the gate |

Real interceptions: the boot gate has refused to start on divergent
state on three occasions in paper operation — each documented with
root cause and remediation.

---

## Testing Strategy

- **Fault-injection harness** (public: `fault-injection-harness`) —
  five deterministic fault classes: process termination, broker
  disconnect, concurrent writers, phantom positions, clock skew. One
  class (concurrent writers) was later reproduced in the paper
  environment through three distinct vectors — root-caused and
  mitigated with resource-level locking.
- **Idempotence:** same target twice ⇒ zero orders (tested).
- **Boot-gate behavior:** divergence fixtures ⇒ non-zero exit, no
  orders (tested).
- Test evidence for the formally verified invariants lives in the
  companion repositories (see below).

---

## Verification Companions (public, no code integration)

| Project | Language | What it proves |
|---|---|---|
| [verified-ledger](https://github.com/ddehanne/verified-ledger) | Rust | Ledger correctness model: hash-chained journal, torn-tail recovery, replay determinism — property tests, Miri, 17M+-run fuzzing campaign, Kani bounded model checking |
| [execution-journal](https://github.com/ddehanne/execution-journal) | C++20 | Execution-integrity engine: durable journal with fsync dispatch-eligibility barrier, boot reconciliation vs a scriptable lying broker, real-process fault injection (SIGKILL), IBKR TWS adapter exercised against a live paper gateway, cross-validated position-by-position against this pipeline's ledger |

The Python pipeline applies the same invariants operationally
(append-only ledgering, single-writer, reconciliation, fail-closed);
the companions prove them formally in isolation. Claims about
hash-chained integrity and bounded proofs belong to the companions,
not to `ledger.jsonl`.

---

## Technology Stack

| Component | Language | Key libs | Purpose |
|---|---|---|---|
| Research & signal | Python | NumPy/Pandas/Parquet | Universe construction, features, evaluation |
| Execution & orchestration | Python | ib_insync | Order construction, submission, fills, state |
| Integrity gates | Python | flock (kernel) | Single-writer, boot reconciliation |
| Monitoring | Prometheus/Grafana | — | Metrics, dashboards, alert-only Telegram |
| Verification companions | Rust / C++20 | Kani, proptest, libFuzzer / GoogleTest, sanitizers | Formal and adversarial verification of the shared contract |

---

## Future Evolution

1. **In-session broker reconnection** — survive external Gateway
   restarts without operator action (known limitation, planned).
2. **Exchange calendar integration** — holidays and early closes in
   scheduler and alert thresholds ("stale at market close is not a
   fault" — PM-019 lesson).
3. **Configuration consolidation** — single authoritative config
   source (a dual-config ambiguity is a known open item).
4. **Cost-model refinement** — per-trade fixed costs and tail-of-
   liquidity (p10 ADV) participation in the capacity model.
5. **Staged live-capital transition** — gated on a hands-off
   validation period with denominator-reported metrics.

---

## References

- `README.md` — methodology, findings, current status
- `docs/PM-015-INCIDENT.md` (+ doctrine addendum), `docs/PM-016-ZOMBIE-CGROUP.md`, `docs/PM-018.md` — incident lineage
- `docs/INFRA-001-SYSTEMD-LIFECYCLE.md` — process lifecycle controls
- Companion repositories: verified-ledger, execution-journal, fault-injection-harness
