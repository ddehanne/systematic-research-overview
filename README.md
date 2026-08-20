# QS1-XSMR: Systematic Quantitative Research & Execution Infrastructure

## Overview

QS1-XSMR is a quantitative research and execution framework independently
architected and operated over a U.S. equity universe. It ran with real capital
through Interactive Brokers from August 11 to August 20, 2026, following
paper-environment validation.

**The signal has no measurable predictive power.** Information Coefficient of
score(T) against return(T→T+5), measured over 2016–2026 on a point-in-time
universe: **+0.0029, t = +0.20** (n = 129). The test is powered to detect an IC
of 0.03 (standard error 0.0146). No effect is present.

The strategy was closed on that measurement on August 20, 2026. This document
records the system, the method that produced the verdict, and the audit that
made it necessary.

---

## What was measured

Information Coefficient by period. Universe rebuilt point-in-time at each date
from trailing data only; forward returns aligned explicitly against a wide price
matrix rather than through group operations, a class of misalignment that had
previously inverted the sign of a live metric.

| Period | IC | std | n | t | names scored |
|---|---|---|---|---|---|
| **FULL 2016–2026** | **+0.0029** | 0.166 | 129 | **+0.20** | 188 |
| 2020 COVID | −0.1335 | 0.217 | 19 | −2.68 | 167 |
| 2022 bear | +0.0110 | 0.205 | 51 | +0.38 | 218 |
| 2024 bull | −0.0109 | 0.141 | 51 | −0.55 | 232 |
| 2025 crash | −0.0199 | 0.152 | 17 | −0.54 | 233 |

The FULL line is the result: a ten-year window, not selected after seeing
results. The COVID line is the only individually notable one and does not
survive scrutiny — nineteen observations, found among five periods examined,
four of which were chosen for being crisis events. It is also consistent with a
known behaviour of cross-sectional strategies in liquidity crises, where
dispersion collapses and dislocated names keep falling.

The replay harness that produced these figures iterates date by date through the
production scoring path, applies commission and slippage as declared parameters,
prints the number of names scored at each rebalance, and emits the hash of the
commit that generated the output.

---

## Correction to earlier versions of this document

Prior revisions stated a statistically significant regime-conditional edge under
rate stress (bootstrap p = 0.001) and described the universe as point-in-time.
**Both statements are withdrawn.**

**On the claimed edge.** Those figures came from a computation module referenced
throughout the codebase as the source of the benchmark results. That module does
not exist in the repository and no commit ever added it. Its outputs are not
reproducible, cannot be corrected, and cannot be re-run on a clean universe. A
content hash proves file integrity; it does not prove computational
reproducibility. The distinction was not made at the time, and an audit trail
was claimed on that basis.

**On the universe.** The historical dataset holds 238 tickers over eleven years
with **zero delistings**, and 56 names introduced after March 2020. For a
universe described as high-volatility U.S. equities, zero attrition over eleven
years is not possible; the list was composed among survivors. Backtested
crisis-period figures were therefore an optimistic ceiling of unquantified size.
By comparison, a separate 38-name universe used elsewhere in this work carried
six measured delistings, and the engine displayed them.

**On the signal path.** A static audit of the executing code found three further
discrepancies with its own documentation: sector neutralisation never runs — the
guarding condition tests for a column the feature builder does not produce, so
the block is skipped silently on every cycle; beta neutralisation is not
imported by the runner; and one term of the final score multiplies a zero
vector, so the documented two-component weighting has one component.

---

## Research method

**Evaluation protocol.** Hypotheses and acceptance criteria were written and
committed before each test was run. Four strategies were closed against
pre-declared criteria:

| Strategy | Method | Verdict |
|---|---|---|
| QS1-XSMR | cross-sectional mean reversion, attention/residual pipeline | IC +0.003 over ten years, t = +0.20 |
| QS2-VBMR | linear regression channel, StochRSI, volume, rebound quality | entry signal shows no predictive power; the positive tail came from the exit rule, not the entry |
| QS3-XLMR | same signal, low-volatility large-cap universe | −10.90%; gross edge +0.47% against 11.00% friction over 27 months |
| QS4-DTP | QS2 entry, net take-profit exit | hit rate rises 48.1% → 58.5% while return falls; six −19% trades erase twenty-nine +5.9% ones |

**Exploration budget.** Eighteen configurations across three universes and three
periods. No parameter set survived a change of either — the best variant of one
period was the worst or mediocre of another. A take-profit family that appeared
stable across two periods was then tested on a third, never previously touched,
with the criterion declared in advance. It failed.

Beyond roughly a dozen cells, additional search produces artifacts rather than
information. The budget was declared spent and the search stopped, rather than
continued until something passed.

---

## System architecture

```text
┌─────────────────────────────────────────────────────┐
│         SIGNAL RESEARCH LAYER                       │
│  Cross-sectional mean reversion                     │
│  Status: CLOSED — IC +0.003, t=+0.20 over 10y       │
└────────────────────┬────────────────────────────────┘
                     ▼
┌─────────────────────────────────────────────────────┐
│    EXECUTION & ORCHESTRATION LAYER                  │
│  Deterministic order routing (IBKR)                 │
│  • Kernel-level single-writer lock (flock)          │
│  • Broker reconciliation gate at boot (fail-closed) │
│  • Account identity guard                           │
│  • Operator kill switch, per-order notional cap     │
│  • Stopped-symbol blacklist: no same-session        │
│    re-entry after a hard stop                       │
│  • Append-only fill ledger                          │
└────────────────────┬────────────────────────────────┘
                     ▼
┌─────────────────────────────────────────────────────┐
│      OBSERVABILITY                                  │
│  Strict "absence is never zero" semantics: every    │
│  value carries its age, health checks test function │
│  rather than process presence, denominators are     │
│  withheld until sample sizes support them           │
└─────────────────────────────────────────────────────┘
```

The execution and observability layers are independent of the signal and remain
valid.

---

## Operational doctrine

**Fail-closed, human-in-the-loop.** After a documented incident lineage, every
autonomous restart mechanism was deliberately removed — eight distinct
resurrection vectors identified and eliminated: cron watchdogs, systemd restart
policies, bot commands, alerter auto-restarts, scheduled restarts.

The only start path:

```text
deliberate operator action
  → kernel single-writer lock (exit on conflict)
  → account identity guard (exit on wrong broker account)
  → broker reconciliation gate: fresh broker read compared against all
    local state sources; any divergence or unreachable broker
    → refuse to trade
  → trading loop
```

Alerting components may alert; they never start or restart anything. During
qualification the reconciliation gate intercepted broker/local divergences on
three occasions, refusing to boot rather than proceeding on inconsistent state.

---

## Post-mortem corpus

Twelve written post-mortems. Structure per entry: trigger, evidence, hidden
assumption, failure mode, missed signal, remediation, negative test with proof
level, residual risk. Proof levels run from implemented, to negatively tested,
to end-to-end exercised, to operationally observed.

Representative entries:

**PM-043** — the operator kill switch was never read by the trading loop. It
reported success while doing nothing. Demonstrated live, before the fix.

**PM-044** — hard stop and rebalance were uncoordinated: after a stop closed a
position, the allocator saw zero holdings against a non-zero target and
re-entered one second later. Observed in live trading as a sell followed
immediately by a buy on the same symbol.

**PM-046** — one structural primitive behind four separate defects on the
capital path: *a read failure produces a permission, never a refusal*. The
sector-neutralisation defect described above is the same primitive on the
computation path, found four days later.

**PM-050** — a broker fill executed and remained invisible to the trading loop
for eighty minutes, generating an orphan order retried every cycle. A cash
account prevented it from becoming an unintended short position.

Recurring theme: controls verified presence, not function. Process alive is not
process healthy; config file present is not config file consumed.

---

## Live operating record

Real capital ran from August 11 to August 20, 2026 on a small account sized for
validating the execution path rather than producing returns. Every closed trade
was net negative; total realised loss was of the order of tens of dollars.

Execution costs measured on real fills: approximately 37–42 basis points per
side at that account size, against a gross edge subsequently measured at
effectively zero.

The account was liquidated and the infrastructure decommissioned on
August 20, 2026.

---

## Public verification companions

Two projects reflecting the same correctness principles into independently
verifiable artifacts (shared invariants, no code integration with this pipeline):

- **[verified-ledger](https://github.com/ddehanne/verified-ledger)** (Rust) —
  order/execution ledger core: bounded model checking (Kani), property-based
  testing, Miri, coverage-guided fuzzing.
- **[execution-journal](https://github.com/ddehanne/execution-journal)** (C++20)
  — execution-integrity engine: durable hash-chained journaling, real-process
  fault injection, broker reconciliation, IBKR TWS adapter exercised against a
  live paper gateway (read-only), cross-validated position-by-position against
  an independent ledger implementation.

---

## Design philosophy

1. **Determinism first** — every execution path testable and reproducible.
2. **Fail-closed** — on any inconsistency the system refuses to act; recovery is
   a deliberate human decision through a gated boot chain.
3. **Auditability** — append-only ledgering; decisions logged and
   reconstructible by replay. A hash proves integrity, not reproducibility.
4. **Evidence over intention** — external facts are recorded and reconciled,
   never assumed. Including when the evidence closes the project.

---

## Source code

The implementation repository is private. The replay harness, post-mortem
corpus, pre-registrations, and closure documents can be made available for
technical discussion.

---

## Status

Closed. The signal was measured and does not work. The measurement is the
result, and it is published as such.
