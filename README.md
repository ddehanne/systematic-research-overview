# Systematic Trading Infrastructure — Engineering Case Study

**Independent research-to-execution project · March–August 2026 · Interactive Brokers · Python**

I built and operated a systematic U.S. equities pipeline spanning market data, research, portfolio construction, broker integration, order execution, reconciliation, and operational monitoring. After several months of paper trading, it reached **limited real-capital execution in August 2026**.

The project did **not** establish a profitable trading strategy or a production-ready execution system. Its value as an engineering case study is the documented progression from research to live operations—and the failures that became visible along the way.

## At a glance

### Project
**March–August 2026** · Independently operated

Interactive Brokers API / IB Gateway

Paper trading before August 11, 2026

### Live execution
**August 11–18, 2026** · Real-capital fills

**12 recorded fills** · Five completed round trips

One position still open at the last documented halt

### Financial record
**Initial account value:** approximately **$1,281** (about **€1,116** at the recorded conversion)

**Realized closed-trade result:** **−$19.32 gross**; approximately **−$29 net**, with some commissions estimated

**Final account value / liquidation:** Not established from available records

### Research outcome
**No measurable predictive signal** at the tested horizon in the corrected historical replay.

*Live figures above are reconstructed from the project's internal fill ledger and dated operational records, not independently verified against final broker statements. They are not a claim of audited performance.*

## The system

**Offline — research**

Market data → Historical replay → Signal definition

**Live — execution**

1. Market data
2. Portfolio targets and pre-trade checks
3. IB Gateway — orders and fills
4. Local execution ledger
5. Reconciliation and monitoring
6. Operator decisions

The implementation included historical-data ingestion, backtesting, target-weight allocation, an execution loop, broker-account checks, order submission and fill tracking, an append-only ledger, reconciliation, alerts, and incident runbooks.

These components did **not** all reach the same assurance level. Some safeguards were tested against failure conditions; others were implemented but never exercised, and some proved ineffective in live operation. The distinction between *implemented*, *tested*, and *reliable* is central to this case study.

The implementation repository remains private. Signal definitions, feature engineering, portfolio parameters, account identifiers, and operational access details are intentionally excluded.

## What operation revealed

### 1. Automatic recovery prevented a safe stop — paper trading

Multiple restart paths had accumulated over time. During a paper-trading incident, stopping the trading service did not keep it stopped: other mechanisms restarted it. In the same incident, a liquidation script ran twice and created unintended short positions in the simulated account.

**Response:** remove automatic restart paths, move the single-writer lock from the launcher to the shared resource, and require a fresh broker-state reconciliation before any start. Removing the restart paths took more than one pass: further paths were found after the first cleanup. Afterwards, the reconciliation gate refused to start the loop on several occasions in paper trading; in the two cases documented in detail, local state had gone stale after an outage.

**Trade-off:** the system became dependent on deliberate operator intervention after outages. This was a safety choice, not an availability achievement.

### 2. A dead connection looked healthy — live trading

The trading loop kept a broker connection that had silently died with an earlier gateway session. Reads still returned cached values, so the loop appeared healthy, but every order failed. When a protective stop triggered, it could not be sent. The loop logged the orders as "submitted" because it counted intentions rather than broker acknowledgements, and the reconciler reported a match because nothing had moved.

The position remained without the intended protection for approximately **14 minutes**, until the loop was restarted with a fresh connection.

**Response:** a connection check was added before the broker reads used for reconciliation. **Limit:** in the last available code snapshot, the order path itself had no round-trip liveness check, and the order counter still counted intentions.

### 3. The stop control reported success without stopping anything — live trading

On the same day, the operator issued a halt through the chat interface. The interface confirmed the halt, but the trading loop never read that flag; the loop had to be terminated at the process level.

**Response:** the loop was changed to check the halt flag at the start of each cycle. **Limit:** the available records do not establish the negative test that would prove the revised halt blocks live order submission.

### 4. A broker fill was invisible to the local ledger — live trading

An order that most likely survived from an earlier session executed later. Because the ledger was written only on the current session's submission path, the fill was missed. Ledger and broker disagreed for more than **80 minutes**, during which the loop repeatedly tried to sell a position it no longer held; the cash-account restriction rejected those orders instead of opening an unintended short.

**Response:** cancel outstanding broker orders at startup (implemented; no negative test in the available records). A more fundamental redesign—recording broker-confirmed events independently of the submission path and propagating reconciliation failures to alerts—remained unfinished when the project stopped.

### The recurring failure mode

> **A failed read silently became a permission.**

Across multiple incidents, missing data or swallowed errors could weaken a constraint while leaving the process apparently healthy. The resulting engineering principle was to **fail closed** on uncertain broker state and to test whether a safeguard actually works—not merely whether its process, flag, or dashboard exists.

## Research outcome: a negative result

A corrected historical replay of the scoring logic as actually executed, over 2016–2026, produced a full-period information coefficient of **+0.0029** (*t* = **+0.20**) at a five-day horizon. This provided **no measurable evidence of predictive power** at that horizon.

The review also uncovered defects in earlier research claims, including discrepancies between documented and executed scoring logic and a survivorship-biased universe. The replay only scores securities already listed at each date, but it draws from the same survivor-only security list, so it is, if anything, an optimistic bound. Earlier favorable claims were withdrawn. At the live account size, minimum commissions were material—roughly **0.4%–1.0% of order notional per side** in the recorded sample—further undermining economic viability.

The appropriate conclusion is **not** that a strategy was proven universally incapable of working. It is that **this implementation and its corrected evaluation did not justify further deployment**.

## What this project demonstrates—and what it does not

**Demonstrated:** independent integration of a research-to-broker workflow; actual order execution with personal capital; operational diagnosis across broker, ledger, and controls; documentation of failures; and a decision to halt a system whose evidence did not support continuation.

**Not demonstrated:** sustainable alpha, positive net returns, dependable unattended operation, fully validated safety controls, or institutional production readiness. The live sample comprised only twelve fills over a short period.

AI coding assistants were used during development. I remained responsible for the project’s technical direction, operational decisions, and deployment.

## Corrections to earlier public material

Earlier versions of this repository overstated system maturity and included unsupported architectural or performance claims. In particular, descriptions of a production-grade system, a purported execution component that did not exist, and certain historical research results were inaccurate. Those claims are withdrawn.

An earlier version of the incident write-up on duplicate trading loops (PM-015) contained an illustrative timeline and described live trading. The incident occurred in a **paper account**; the version in this repository is limited to what the project records support. Some older material may remain accessible through Git history or external copies.

## Repository contents

- **[Architecture](ARCHITECTURE.md)** — Historical architecture and assurance levels reached by each component.
- **[PM-015](docs/PM-015-INCIDENT.md)** — Duplicate trading-loop instances (paper, June 2026).
- **[PM-016](docs/PM-016-ZOMBIE-CGROUP.md)** — Processes surviving a service stop (paper, July 2026).
- **[INFRA-001](docs/INFRA-001-SYSTEMD-LIFECYCLE.md)** — Service stop hardening (July 2026; later superseded).
- **[PM-018](docs/PM-018.md)** — Automatic restart paths and duplicate liquidation (paper, July 2026).

The live-trading incidents summarized above are described here only; their full post-mortems remain in the private repository.

## Evidence and scope

This is a **historical engineering case study**, not an open-source release of the trading system. Its narrative is based on dated project commits, incident reports, runbooks, and internal fill records. The original private implementation and account records are not published here.

The trading loop was halted by **August 20, 2026**. A position was still open in the last available operational snapshot; the final liquidation date and account balance remain unverified. No claim is made that the account was fully liquidated on August 20.

**Status: closed as a research/execution experiment; no validated trading edge.**
