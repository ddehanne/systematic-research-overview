# QS1-XSMR: Systematic Quantitative Research & Execution Infrastructure

## Overview

QS1-XSMR is a modular quantitative research and IBKR paper-trading infrastructure independently architected and operated across 238 US equities screened for average daily volume above $5 million.

**Research Framework:** Temporal walk-forward validation (IS: 2020-2023 locked, OOS Backtest: April 2024 - May 2026)

---

## System Architecture

The platform is organized around three independent engines with explicit contracts:

```text
┌─────────────────────────────────────────────────────┐
│         SIGNAL RESEARCH LAYER                       │
│                                                     │
│  Cross-Sectional Mean Reversion (238 equities)      │
│  Status: Invalidated post-costing (published        │
│          as methodology reference)                  │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│    EXECUTION & ORCHESTRATION LAYER                  │
│                                                     │
│  Scheduled Deterministic Order Routing                   │
│                                                     │
│  • Single-writer execution control                  │
│  • Automated broker state reconciliation (PM-015)   │
│  • Prometheus/Grafana monitoring                    │
│  • Circuit breaker & exposure controls              │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│      RISK OVERLAY & MACRO PROTECTION                │
│                                                     │
│  Rule-Based Strategy & Constraint Enforcement       │
│  [Proprietary - Available under NDA]                │
└─────────────────────────────────────────────────────┘
```

---

## Research Methodology

**Walk-Forward Validation Protocol:**
- Hypotheses, evaluation criteria, and framework locked before held-out testing
- In-sample: 2020-2023 (development only)
- Out-of-sample: April 2024 - May 2026 (evaluation on held-out historical data)
- All research decisions version-controlled and auditable

**Signal Evaluation:**
- 10 competing hypotheses tested under strict OOS criteria
- Information Coefficient (IC) and Information Coefficient Information Ratio (ICIR) used as quality measures
- Transaction cost assumption: 5 bps per traded leg, equivalent to 10 bps round-trip
- All candidates rejected post-costing due to turnover economics

**Key Finding:** The evaluated signal candidates did not meet predefined expected net-performance criteria after the modeled transaction costs. The methodology is retained as a documented research and architectural reference. Published for architectural reference.

---

## Infrastructure

**Modular Workflow:**

Data Ingestion → Signal Generation → Portfolio Construction → Execution → Monitoring → Reconciliation

**Operational Components:**

1. **Data Layer**
   - IBKR historical OHLCV (10 years, 238 tickers)
   - Pre-market, regular, and after-hours data handling
   - Volume filters: ADV > $5M

2. **Signal Layer**
   - Cross-sectional z-score normalization
   - Mean-reversion hypothesis (quintile L/S construction)
   - Daily rebalance with position persistence rules

3. **Execution Layer**
   - IBKR API integration with order acknowledgment tracking
   - Deterministic position delta computation
   - Lot-size quantization and min-notional enforcement

4. **Monitoring & Reconciliation**
   - Prometheus metrics for portfolio state, execution activity, broker connectivity, and operational controls
   - Grafana dashboards: live positions, NAV curve, exposure
   - Automated broker state validation (PM-015)

---

## Operational Controls

**Safeguards Enforced:**
- Maximum gross notional limits
- Sector concentration caps
- Pre-trade exposure validation
- Broker connectivity monitoring
- Automated halt on constraint violation

**Incident: PM-015 (June 2026)**

Duplicate execution-process instances created concurrent access to the same IBKR paper-trading account, resulting in order-state divergence.

Resolution: Implemented a single-writer lock-file guard, broker-state reconciliation, append-only event logging, and additional exposure safeguards.

See: docs/PM-015-INCIDENT.md

---

## Current Status

**Out-of-Sample Backtest Results (April 2024 - May 2026):**
- Historical walk-forward validation on held-out data completed
- Signal methodology: Mathematically sound, economically constrained at $250K scale

**Live Deployment (Infrastructure Validation Phase):**
- Paper-trading deployment: June 25, 2026 - Present
- Uptime: 100%
- Orders submitted: 9 | Fills executed: 9
- Broker-state reconciliation: 100% alignment (ledger ↔ IBKR)
- API latency: <50ms average
- Monitoring: Prometheus metrics and Grafana dashboards operational

---

## Design Philosophy

1. **Determinism First:** Every execution path is testable and reproducible
2. **Failure Containment: Operational controls are designed to detect, isolate, and reconcile known failure modes.
3. **Auditability:** Every decision logged with timestamp and hash
4. **Separation of Concerns:** Signal, execution, and risk management are independent modules

---

## Documentation

- `ARCHITECTURE.md` — System design, engine contracts, data flow
- `docs/PM-015-INCIDENT.md` — Incident root-cause analysis and recovery
- `research/methodology.md` — Research protocol, hypothesis testing, OOS validation

---

## Proprietary Components

Signal calibration, strategy parameters, live execution tuning, and detailed factor decomposition remain proprietary and available under professional confidentiality agreement.

---

## Disclaimer

This platform is operated through an IBKR paper-trading account for infrastructure validation. Simulated execution is not presented as a substitute for real-money trading. All performance figures and operational metrics remain subject to reproducible verification. Model, execution, broker, liquidity, data, and operational risks remain material.
