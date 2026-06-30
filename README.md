# QS1-XSMR: Systematic Quantitative Research & Execution Infrastructure

## Overview

QS1-XSMR is a modular quantitative research and IBKR paper-trading
infrastructure independently architected and operated across a universe
of 238 US equities screened for average daily volume above $5 million.

**Research Framework:** Fixed chronological development (2020–2023) and
held-out evaluation (April 2024–May 2026)

---

## System Architecture

The platform is organized into two primary logical layers with explicit module boundaries:

```text
┌─────────────────────────────────────────────────────┐
│         SIGNAL RESEARCH LAYER                       │
│                                                     │
│  Cross-Sectional Research Candidates                │
│  Status: Evaluated and rejected                     │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│    EXECUTION & OPERATIONAL CONTROLS LAYER           │
│                                                     │
│  Scheduled Order Routing & State Management         │
│                                                     │
│  • Single-writer execution control                  │
│  • Periodic broker-state verification               │
│  • Prometheus/Grafana monitoring                    │
│  • Constraint enforcement & exposure controls       │
└─────────────────────────────────────────────────────┘
```

---

## Research Methodology

**Temporal Held-Out Validation Protocol:**
- Development period: 2020–2023 (in-sample, hypothesis exploration)
- Evaluation period: April 2024–May 2026 (held-out, no retraining)
- Single chronological split, not rolling or expanding windows
- Hypotheses and evaluation criteria were documented as part of
  the research protocol

**Signal Evaluation:**
- A finite set of documented cross-sectional research candidates evaluated
- Information Coefficient (IC) calculated using Spearman rank correlation
- IC Information Ratio (ICIR), calculated from the mean and variability of
  the daily IC series, subject to the implementation's annualization convention
- Deflated Sharpe Ratio (DSR) applied as an additional diagnostic,
  subject to simplifying distributional assumptions
- Transaction cost assumption: 5 bps per traded leg (10 bps round-trip)
  applied to turnover calculations

**Evaluation Outcome:**

All evaluated candidates were rejected under the documented evaluation
framework after considering held-out behavior, turnover, modeled transaction
costs, and portfolio-level robustness diagnostics.

**Research Observation:**

One candidate exhibited favorable performance in selected historical
high-stress windows, including a 2020 COVID-related period and an April 2025
window, while underperforming in normal or bullish regimes. This
regime-dependent behavior is treated as exploratory research evidence only,
not as validated alpha.

---

## Infrastructure

**Modular Workflow:**

Data Ingestion → Signal Generation → Portfolio Construction → Execution →
Monitoring → Verification

**Operational Components:**

1. **Data Layer**
   - IBKR historical daily OHLCV covering the available research period
   - Universe: 238 US equities after liquidity and data-availability screening

2. **Research Signal Layer**
   - Cross-sectional ranking and normalization
   - Candidate-specific mean-reversion transformations
   - Quintile-based portfolio experiments for selected candidates
   - Position-persistence rules evaluated to reduce turnover

3. **Execution Layer**
   - IBKR API integration with order-state observation
   - Deterministic position-delta generation with portfolio and
     order-size validation
   - Scheduled order submission during configured execution windows

4. **Monitoring & Verification**
   - Prometheus metrics for portfolio state, execution activity,
     broker connectivity, and operational controls
   - Grafana dashboards for positions, NAV, exposure, and verification status
   - Periodic broker-state comparison workflow for divergence detection

---

## Execution Timing

**Signal Computation**
- Portfolio targets are generated on a scheduled daily research cycle
- Outputs are prepared before the next execution session

**Order Routing**
- Orders are submitted through the IBKR paper-trading environment during
  configured execution windows
- The Rust execution component consumes JSON-serialized outputs produced
  by the Python research workflow

**Transaction-Cost Assumption**
- Research evaluation applies a linear assumption of 5 bps per traded leg,
  equivalent to 10 bps round-trip
- The assumption captures first-order turnover drag but does not model
  nonlinear market impact, intraday bid-ask spreads, or order-level slippage

---

## Operational Controls

**Safeguards Enforced:**
- Maximum gross notional limits
- Sector concentration limits
- Pre-trade exposure validation
- Broker connectivity monitoring
- Constraint checks capable of blocking new execution when predefined
  limits are violated
- Position-delta generation with explicit validation
- Append-only event logging with hash-chain integrity checks designed
  to detect later modification

**Incident: PM-015 (June 2026)**

Duplicate execution-process instances created concurrent access to the same
IBKR paper-trading account, resulting in order-state divergence.

Resolution: Implemented a local single-writer lock-file guard, stronger
broker-state verification, an explicit reconciliation workflow, append-only
event logging, and additional exposure safeguards.

See: `docs/PM-015-INCIDENT.md`

---

## Known Limitations

**Data & Universe:**
- Survivorship bias: Historical universe of 238 tickers with ADV > $5M;
  delistings and universe changes not explicitly modeled
- Corporate actions: No independent adjustment pipeline was implemented.
  The exact treatment of splits, dividends, mergers, symbol changes, and
  delistings in the source data requires further verification

**Research Protocol:**
- Temporal validation: One primary chronological holdout was used rather
  than repeated rolling or expanding evaluation windows; conclusions may
  therefore be sensitive to the selected period and regime composition
- Multiple testing: The candidate set and evaluation framework were
  documented; DSR was used as an additional diagnostic, while formal
  Bonferroni, Holm, or FDR procedures were not applied

**Implementation:**
- Cross-language interface: JSON serialization chosen for readability and
  debuggability at the current scale; explicit schema versioning remains
  an area for further hardening
- Verification workflow: Detects divergences; automatic correction capability
  remains limited to defined scenarios
- Concurrency controls: Single-writer enforcement via lock-file; not a
  distributed-system solution

---

## Current Status

**Held-Out Historical Evaluation (April 2024–May 2026):**
- Historical evaluation on a held-out chronological period completed
- All evaluated candidates rejected under the documented evaluation framework

**Paper-Trading Infrastructure Validation:**
- Verification workflow operational
- Orders and fills observed during recent operational windows
- Known divergence: One position-weight discrepancy was documented and
  remained under investigation at the latest operational review

---

## Design Philosophy

**1. Determinism First:** Core execution paths designed for reproducible testing

**2. Failure Containment:** Operational controls detect and isolate known
failure modes (PM-015 reference)

**3. Auditability:** Execution events recorded with timestamps and
hash-chain integrity checks

**4. Separation of Concerns:** Signal, execution, and risk-management
separated through explicit module boundaries

**5. Honest Rejection:** Candidates that failed the documented economic and
robustness criteria were rejected rather than reframed as successful results

---

## Documentation

- `README.md` — This file; high-level overview
- `ARCHITECTURE.md` — System design, module contracts, data flow,
  concurrency controls, and cross-language interfaces
- `docs/PM-015-INCIDENT.md` — Incident analysis, recovery mechanism,
  and verification testing

**Planned documentation:**
- Research protocol, candidate framework, and validation assumptions

---

## Proprietary Components

- Signal calibration, candidate definitions, and selected implementation
  details are intentionally omitted from the public repository
- Public documentation focuses on methodology, system boundaries,
  operational controls, and known limitations

---

## Disclaimer

This platform is operated through an IBKR paper-trading environment for
research and infrastructure validation only. Simulated execution is not
presented as evidence of real-money execution quality or strategy viability.

Research conclusions depend on data quality, universe construction,
transaction-cost assumptions, validation design, and implementation choices.

The evaluated candidates were rejected under the documented research framework.
Selected subperiod behavior is treated as exploratory evidence only.

Material risks remain: model risk, data-quality risk, execution risk,
broker API ambiguity, liquidity risk, and operational risk.
