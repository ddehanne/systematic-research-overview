# QS1-XSMR: Systematic Quantitative Research & Execution Infrastructure

## Overview

QS1-XSMR is a quantitative research and execution system independently
architected and operated around U.S. equities and Interactive Brokers. It
progressed from historical research to paper-environment validation and a
short real-capital deployment from August 11 to August 20, 2026.

**Final result: the signal does not work.**

A corrected ten-year replay using a point-in-time reconstructed universe
measured the Information Coefficient of score(T) against return(T→T+5) at
**+0.0029, t = +0.20** (n = 129). No measurable predictive effect was found.

The strategy was closed on that evidence on August 20, 2026.

What remains is the research and engineering record: the methodology used to
reach the verdict, execution infrastructure, broker reconciliation, operational
controls, twelve post-mortems, and the audit process that ultimately overturned
earlier conclusions.

---

## What was measured

The final replay rebuilt the eligible universe point-in-time at each evaluation
date using trailing information only. This corrected an earlier research dataset
whose fixed 238-name universe was subsequently found to contain survivorship
bias.

Forward returns were aligned explicitly against a wide price matrix rather than
through group operations, a class of misalignment that had previously inverted
the sign of a live metric.

Information Coefficient by period:

| Period | IC | std | n | t | names scored |
|---|---:|---:|---:|---:|---:|
| **FULL 2016–2026** | **+0.0029** | 0.166 | 129 | **+0.20** | 188 |
| 2020 COVID | −0.1335 | 0.217 | 19 | −2.68 | 167 |
| 2022 bear | +0.0110 | 0.205 | 51 | +0.38 | 218 |
| 2024 bull | −0.0109 | 0.141 | 51 | −0.55 | 232 |
| 2025 crash | −0.0199 | 0.152 | 17 | −0.54 | 233 |

The FULL line is the primary result: a ten-year window, not selected after
observing the outcome.

The COVID line is the only individually notable subperiod and does not survive
scrutiny — nineteen observations, found among five periods examined, four of
which were selected because they represented stressed market conditions. It is
therefore treated as exploratory evidence rather than a validated
regime-conditional effect.

The replay harness that produced these figures iterates date by date through
the production scoring path, applies commission and slippage as declared
parameters, prints the number of names scored at each rebalance, and emits the
hash of the commit that generated the output.

At the observed sampling variability, the estimate is too small relative to
its uncertainty to support a practically meaningful predictive effect.

---

## Correction to earlier versions of this document

Prior revisions stated a statistically significant regime-conditional edge
under rate stress (bootstrap p = 0.001) and described the historical 238-name
research universe as point-in-time.

**Both statements are withdrawn.**

### On the claimed edge

Those figures came from a computation module referenced throughout the codebase
as the source of the benchmark results. That module does not exist in the
repository and no commit ever added it.

Its outputs are therefore not reproducible, cannot be corrected, and cannot be
re-run on a clean universe.

A content hash proves file integrity; it does not prove computational
reproducibility. That distinction was not made at the time, and an audit trail
was claimed on that basis.

### On the historical universe

The earlier historical dataset holds 238 tickers over eleven years with
**zero delistings**, and 56 names introduced after March 2020.

For a universe described as high-volatility U.S. equities, zero attrition over
eleven years is not credible evidence of a point-in-time universe. The list had
been composed among surviving names.

Backtested crisis-period figures derived from that dataset were therefore
subject to survivorship bias of unquantified magnitude.

By comparison, a separate 38-name universe used elsewhere in this work carried
six measured delistings, and the engine displayed them.

The final IC results reported above were produced only after replacing the
survivorship-biased historical universe with a point-in-time reconstruction.

### On the signal path

A static audit of the executing code found three further discrepancies with its
own documentation:

- Sector neutralisation never runs: the guarding condition tests for a column
  the feature builder does not produce, so the block is silently skipped on
  every cycle.
- Beta neutralisation is not imported by the runner.
- One term of the final score multiplies a zero vector, so the documented
  two-component weighting has only one effective component.

These findings were treated as implementation defects, not retroactively
reinterpreted as research results.

---

## Research method

### Evaluation protocol

Hypotheses and acceptance criteria were written and committed before each test
was run.

Four strategies were closed against pre-declared criteria:

| Strategy | Method | Verdict |
|---|---|---|
| QS1-XSMR | Cross-sectional mean reversion, attention/residual pipeline | IC +0.003 over ten years, t = +0.20 |
| QS2-VBMR | Linear regression channel, StochRSI, volume, rebound quality | Entry signal shows no predictive power; the positive tail came from the exit rule, not the entry |
| QS3-XLMR | Same signal, low-volatility large-cap universe | −10.90%; gross edge +0.47% against 11.00% friction over 27 months |
| QS4-DTP | QS2 entry, net take-profit exit | Hit rate rises 48.1% → 58.5% while return falls; six −19% trades erase twenty-nine +5.9% ones |

### Exploration budget

Eighteen configurations were evaluated across three universes and three
periods.

No parameter set survived a change of either universe or period — the best
variant of one period was the worst or mediocre in another.

A take-profit family that appeared stable across two periods was then tested on
a third period that had not previously influenced the search, with the criterion
declared in advance. It failed.

Beyond roughly a dozen cells, continued adaptive search was judged more likely
to manufacture artifacts than produce independent evidence.

The exploration budget was therefore declared spent and the search stopped
rather than continued until something passed.

---

## System architecture

```text
┌─────────────────────────────────────────────────────┐
│         SIGNAL RESEARCH LAYER                       │
│                                                     │
│  Cross-sectional mean reversion                     │
│  Status: CLOSED — IC +0.003, t = +0.20 over 10y    │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│    EXECUTION & ORCHESTRATION LAYER                  │
│                                                     │
│  Deterministic order routing (IBKR)                 │
│                                                     │
│  • Kernel-level single-writer lock (flock)          │
│  • Broker reconciliation gate at boot (fail-closed) │
│  • Account identity guard                           │
│  • Operator kill switch                             │
│  • Per-order notional cap                           │
│  • Stopped-symbol blacklist: no same-session        │
│    re-entry after a hard stop                       │
│  • Append-only fill ledger                          │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│      OBSERVABILITY                                  │
│                                                     │
│  Strict "absence is never zero" semantics           │
│                                                     │
│  • Every value carries its age                      │
│  • Health checks test function, not process presence│
│  • Denominators withheld until samples support them │
└─────────────────────────────────────────────────────┘
```

The execution and observability layers are independent of the rejected signal
and remain useful as engineering artifacts.

---

## Operational doctrine

### Fail-closed, human-in-the-loop

After a documented incident lineage, autonomous restart mechanisms were
deliberately removed.

Eight distinct resurrection vectors were identified and eliminated, including
cron watchdogs, systemd restart policies, bot commands, alerter auto-restarts,
and scheduled restarts.

The only permitted start path became:

```text
deliberate operator action
        │
        ▼
kernel single-writer lock
(exit on conflict)
        │
        ▼
account identity guard
(exit on wrong broker account)
        │
        ▼
broker reconciliation gate
        │
        ├── broker unreachable ──────────► REFUSE TO TRADE
        │
        ├── local/broker divergence ─────► REFUSE TO TRADE
        │
        ▼
trading loop
```

The reconciliation gate performs a fresh broker read and compares it against
the relevant local state sources before execution is allowed to begin.

Alerting components may alert; they never start or restart the trading system.

During qualification, the reconciliation gate intercepted broker/local
divergences on three occasions, refusing to boot rather than proceeding on
inconsistent state.

---

## Post-mortem corpus

Twelve written post-mortems were produced.

Each entry follows the same structure:

```text
trigger
→ evidence
→ hidden assumption
→ failure mode
→ missed signal
→ remediation
→ negative test
→ proof level
→ residual risk
```

Proof levels distinguish between controls that are merely implemented,
negatively tested, exercised end-to-end, and operationally observed.

Representative entries:

### PM-043 — Kill switch present but non-functional

The operator kill switch was never read by the trading loop.

It reported success while doing nothing.

The failure was demonstrated before the fix.

### PM-044 — Stop/rebalance race

Hard stop and rebalance were uncoordinated.

After a stop closed a position, the allocator observed zero holdings against a
non-zero target and re-entered one second later.

This was observed during real-capital operation as a sell followed immediately
by a buy on the same symbol.

### PM-046 — Read failure becoming permission

One structural primitive appeared behind four separate defects on the capital
path:

> **A read failure produces a permission, never a refusal.**

The sector-neutralisation defect described above represents the same primitive
on the computation path and was identified four days later.

### PM-050 — Invisible broker fill

A broker fill executed and remained invisible to the trading loop for eighty
minutes, generating an orphan order retried every cycle.

A cash account prevented the condition from becoming an unintended short
position.

### Recurring failure pattern

A recurring theme across the incident corpus was:

> **Controls verified presence, not function.**

Process alive is not process healthy.

Config file present is not config file consumed.

Control present is not control enforced.

---

## Live operating record

QS1-XSMR ran with real capital from August 11 to August 20, 2026 on a small
account deliberately sized for validating the execution path rather than
producing investment returns.

Every closed trade was net negative.

Total realised loss was of the order of tens of dollars.

Execution costs measured on real fills were approximately **37–42 basis points
per side** at that account size, against a gross predictive edge subsequently
measured at effectively zero.

The live deployment also exposed operational failures that had not been
identified during paper-environment qualification and became inputs to the
post-mortem and hardening process.

The account was liquidated and the QS1 strategy infrastructure decommissioned
on August 20, 2026 after the research verdict was established.

---

## Public verification companions

Two public projects carry selected correctness principles into independently
verifiable artifacts. They share engineering invariants with QS1-XSMR but do
not integrate its proprietary research implementation.

### [verified-ledger](https://github.com/ddehanne/verified-ledger)

Rust order/execution ledger core with:

- bounded model checking using Kani
- property-based testing
- Miri
- coverage-guided fuzzing

### [execution-journal](https://github.com/ddehanne/execution-journal)

C++20 execution-integrity engine with:

- durable hash-chained journaling
- real-process fault injection
- broker reconciliation
- IBKR TWS adapter exercised against a live paper gateway in read-only mode
- position-by-position cross-validation against an independent ledger
  implementation

---

## Design philosophy

### 1. Determinism first

Execution paths should be reproducible enough to test, replay, and reason about.

### 2. Fail closed

On state inconsistency or unavailable required evidence, the system refuses to
act.

Recovery is a deliberate human decision through a gated boot chain.

### 3. Auditability

Execution events are recorded through append-only ledgering so decisions can be
reconstructed by replay.

A hash proves integrity, not reproducibility.

### 4. Evidence over intention

A control is not considered effective because it exists in code or
configuration.

Its behavior must be tested or observed.

The same rule applies to research claims.

### 5. Negative results are results

A hypothesis that fails its declared evaluation criteria is rejected rather
than reframed.

The objective is not to make the strategy survive the experiment.

The objective is to make the conclusion survive scrutiny.

---

## Scope of public disclosure

This repository documents the research process, system architecture,
operational failures, validation methodology, and final empirical conclusions.

Exact signal definitions, feature transformations, calibration details,
portfolio-construction parameters, sizing logic, and selected implementation
details are intentionally not published.

The purpose of the public artifact is to make the reasoning and engineering
auditable without publishing a complete strategy specification.

---

## Source code

The primary implementation repository is private.

The replay harness, post-mortem corpus, pre-registrations, and closure documents
can be made available for technical discussion where appropriate.

---

## Status

**CLOSED — August 20, 2026**

The signal was measured and does not work.

The measurement is the result, and it is published as such.
