# QS1-XSMR

Systematic US equity research and paper-trading infrastructure.
Independently developed and operated across a universe of 238 US equities screened for average daily volume above $5 million, covering data ingestion, cross-sectional signal generation, portfolio construction, broker execution, monitoring, and reconciliation.

The platform is operated through an IBKR paper-trading account to evaluate broker integration, order-state handling, reconciliation, monitoring, failure containment, and controlled recovery. Simulated execution is not presented as a substitute for a real-money track record.

---

## Research Framework

The research process uses temporal walk-forward validation with an in-sample period from 2020 to 2023 and a held-out evaluation period from April 2024 to June 2026.

Research hypotheses, evaluation criteria, and the validation framework were documented and version-controlled before held-out evaluation.
Model development and parameter selection were conducted using the designated research data before evaluation on the held-out period.

Information Coefficient and Information Coefficient Information Ratio are used as signal-quality measures.

Tested configurations, rejected hypotheses, research decisions, and associated artifacts are retained internally to support reproducibility and auditability.

Performance figures are not currently presented publicly while the underlying research artifacts, calculation conventions, and source outputs are being revalidated for exact reproducibility.

---

## Infrastructure

The platform follows a modular workflow:

```text
data ingestion → signal generation → portfolio construction → execution → monitoring
```

The signal methodology, calibration process, and portfolio rules remain proprietary and are not publicly disclosed.

Infrastructure includes scheduled research and trading workflows, Prometheus and Grafana monitoring, pre-trade exposure controls, deterministic internal research artifacts, integrity checks, and documented incident reviews.

Automated controls monitor execution state, broker connectivity, portfolio exposure, signal health, system failures, and recovery procedures.

---

## Operational Controls

The platform can reduce exposure, block new orders, or halt execution when predefined portfolio-risk, broker-state, or operational thresholds are breached.

An order-state synchronization failure involving concurrent broker sessions was identified through continuous operational monitoring.
Following root-cause analysis, the platform was updated with single-writer execution controls, automated reconciliation of internal target positions, acknowledged orders, executions, and broker-reported positions, together with additional exposure safeguards.

The presence of automated controls does not eliminate model, execution, broker, liquidity, data, or operational risk.

---

## Review and Documentation

The proprietary signal methodology, calibration process, portfolio rules, and detailed incident documentation are not publicly disclosed.
