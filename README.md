# QS1-XSMR: Systematic Quantitative Research & Execution Infrastructure

## Overview

QS1-XSMR is a production-grade quantitative research and execution framework independently architected and operated across 238 US equities screened for average daily volume above $5 million.

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
│  Live Deterministic Order Routing                   │
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
│  Autonomous Strategy & Constraint Enforcement       │
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
- Transaction cost model: 5 bps per leg, round-trip
- All candidates rejected post-costing due to turnover economics

**Key Finding:** The signal methodology is mathematically sound but economically unviable at $250K capital scale. Published for architectural reference.

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
   - Prometheus metrics: order latency, fill rate, cost accumulation
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

**Critical Incident: PM-015 (June 2026)**

Race condition induced by duplicate systemd service instances: systemd restart spawned concurrent `live_loop` processes writing to same IBKR account, creating silent order state divergence.

**Resolution:** Single-writer lock-file guard + automated state reconciliation with exponential backoff retry. System now detects and corrects divergence in <60 seconds without manual intervention.

See: `docs/PM-015-INCIDENT.md`

---

## Current Status

**Out-of-Sample Backtest Results (April 2024 - May 2026):**
- Historical walk-forward validation on held-out data completed
- Signal methodology: Mathematically sound, economically constrained at $250K scale

**Live Incubation (Infrastructure Validation):**
- Paper-trading deployment: June 25, 2026 - Present
- Uptime: 100%
- Orders routed: 9 | Execution fills: 9 | Rejection rate: 0%
- State reconciliation: 100% alignment (Local state ↔ IBKR ledger)
- Broker API latency: <50ms avg

---

## Design Philosophy

1. **Determinism First:** Every execution path is testable and reproducible
2. **Fail-Safe Architecture:** System detects and corrects its own failures
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

This platform is operated through IBKR paper-trading account for infrastructure validation. Simulated execution is not presented as a substitute for real-money trading. All performance figures are subject to revalidation for exact reproducibility. Model, execution, broker, liquidity, data, and operational risks remain unmitigated.
