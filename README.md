# QS1-XSMR: Systematic Quantitative Research & Execution Infrastructure

## Overview

QS1-XSMR is a quantitative research and execution framework independently architected and operated over a point-in-time U.S. equity universe (dated holdings; universe membership is a function of time, not a fixed list). The system began operating with real capital through Interactive Brokers on August 11, 2026, following paper-environment validation.

**Research discipline:** pre-registered hypotheses, held-out evaluation, and a self-initiated data audit that invalidated part of my own backtest — documented below.

---

## System Architecture

Three layers with explicit contracts:

```text
┌─────────────────────────────────────────────────────┐
│         SIGNAL RESEARCH LAYER                       │
│  Cross-sectional mean reversion,                    │
│  point-in-time universe construction                │
│  Status: reclassified regime-conditional            │
│          after data audit (see below)               │
└────────────────────┬────────────────────────────────┘
                     ▼
┌─────────────────────────────────────────────────────┐
│    EXECUTION & ORCHESTRATION LAYER                  │
│  Deterministic order routing                        │
│  (IBKR; paper → real capital)                       │
│  • Kernel-level single-writer lock (flock)          │
│  • Broker reconciliation gate at boot (fail-closed) │
│  • Append-only fill ledger                          │
│  • Prometheus/Grafana monitoring                    │
└────────────────────┬────────────────────────────────┘
                     ▼
┌─────────────────────────────────────────────────────┐
│      RISK OVERLAY & CONSTRAINT ENFORCEMENT          │
│  [Proprietary — available under NDA]                │
└─────────────────────────────────────────────────────┘
```

---

## Research Methodology

**Evaluation protocol:**
- Hypotheses and evaluation criteria locked before held-out testing
- 10 competing cross-sectional mean-reversion candidates evaluated
  (IC / ICIR, turnover constraints, transaction-cost model)
- Candidates failing predefined robustness or net-performance
  criteria were rejected
- All research decisions version-controlled and auditable

**Data audit (self-initiated):**
The point-in-time universe module contained a silent fallback that
substituted the *current* universe for dates before holdings coverage
began — projecting survivors backwards (look-ahead + survivorship).
This invalidated the pre-2021 portion of the IC series. The fallback
was removed (the module now fails loudly); **all quoted figures come
from the audited 2021+ window only.**

**Key finding (clean window):**
Unconditional edge is **not established**. A statistically significant
conditional edge exists under rate-stress regimes (bootstrap p=0.001,
robust to multiple-testing correction). The strategy is therefore
classified as **regime-conditional**, not all-weather — conditional
abstention is a condition of existence, not an enhancement.

---

## Operational Doctrine

**Fail-closed, human-in-the-loop.** After a documented incident lineage
(PM-015 → PM-020), every autonomous restart mechanism was deliberately
removed — eight distinct resurrection vectors identified and
eliminated (cron watchdogs, systemd restart policies, bot commands,
alerter auto-restarts, scheduled cargo-cult restarts).

Current start path — the only one:

```text
deliberate operator action
  → kernel single-writer lock (exit on conflict)
  → broker reconciliation gate: fresh broker read compared against
    ALL local state sources; any divergence or unreachable broker
    → refuse to trade (fail-closed)
  → trading loop
```

Alerting components may alert; they never start or restart anything.
During paper-environment qualification, the reconciliation gate intercepted broker/local-state divergences on three occasions, refusing to boot rather than proceeding on inconsistent state. The same fail-closed reconciliation contract is retained for real-capital operation.

Incident documentation: root cause, remediation, and regression
coverage for each event (available under NDA; one public post-mortem
in the companion repositories).

---

## Current Status

* Real-capital operation began through IBKR on August 11, 2026, following completion of the paper-environment qualification phase.
* The first real order completed the live execution path through broker fill, append-only ledger recording, and broker-state reconciliation.
* Current validation focuses on realized execution costs, reconciliation, operational stability, failure handling, and recovery under real broker conditions.
* Not all operational paths have yet been exercised with real capital; exit and stop-loss paths remain subject to live validation.
* Operational metrics will continue to be reported with denominators: N sessions, X reconciliation cycles, Y controlled restarts, unresolved divergences — never headline claims without them.
* No live-performance or persistent-alpha claim is made from the current operating history.

---

## Public Verification Companions

Two public projects reflect the same correctness principles into
independently verifiable artifacts (shared invariants — **no code
integration** with this pipeline):

- **[verified-ledger](https://github.com/ddehanne/verified-ledger)** (Rust)
  — order/execution ledger core: bounded model checking (Kani),
  property-based testing, Miri, coverage-guided fuzzing.
- **[execution-journal](https://github.com/ddehanne/execution-journal)** (C++20)
  — execution-integrity engine: durable hash-chained journaling,
  real-process fault injection, broker reconciliation, IBKR TWS
  adapter exercised against a live paper gateway (read-only),
  cross-validated position-by-position against an independent
  ledger implementation.

---

## Design Philosophy

1. **Determinism first** — every execution path testable and reproducible
2. **Fail-closed** — on any inconsistency, the system refuses to act;
   recovery is a deliberate human decision through a gated boot chain
3. **Auditability** — append-only ledgering; decisions logged and
   reconstructible by replay
4. **Evidence over intention** — external facts (broker state, fills)
   are recorded and reconciled, never assumed

---

## Proprietary Components

Signal construction, calibration, regime definitions, and live
configuration remain proprietary — available for discussion under
professional confidentiality agreement.

---

## Disclaimer

QS1-XSMR was qualified in an IBKR paper environment before transitioning to limited real-capital operation on August 11, 2026. The live operating history remains short and does not establish persistent alpha, investment performance, or complete operational validation. Not all execution and recovery paths have yet been exercised under real-capital conditions. All research figures are quoted from the audited data window and remain subject to revalidation. Model, execution, broker, liquidity, data, and operational risks remain.
