# Architecture Design Document

## System Overview

QS1-XSMR is engineered as a 3-engine, loosely coupled architecture where:

- Signal generation operates independently from execution
- Execution is decoupled from risk management
- Failures are isolated to limit propagation across components

This design prioritizes operational resilience and separation of responsibilities over maximum performance coupling.

---

## Data Flow Pipeline

```text
┌─────────────────────────────────────────────────────────────┐
│                    EXTERNAL DATA SOURCES                    │
│                                                             │
│  IBKR Historical Data (10y) | Market Data Feed | Risk Data │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│              DATA INGESTION & TRANSFORMATION                │
│                                                             │
│  • IBKR OHLCV fetch (pre-market, regular, AH)              │
│  • Volume filtering (ADV > $5M)                            │
│  • Feature computation (ret1, ret5, ret20, vwap)           │
│                                                             │
│  → Output: Daily feature matrix {date, ticker, features}   │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│         ENGINE 1: SIGNAL RESEARCH LAYER                     │
│                                                             │
│  Input: Feature matrix                                     │
│  Process: Cross-sectional z-score, mean-reversion logic    │
│  Output: Daily signal {ticker, weight} ∈ [-1, 1]           │
│                                                             │
│  • Weights normalized to unit gross exposure               │
│  • Quintile construction (L/S)                             │
│  • Status: Frozen (methodology reference only)             │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│    ENGINE 2: EXECUTION & ORCHESTRATION LAYER (Rust Core)   │
│                                                             │
│  Input: Daily signal {ticker, weight}                      │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  POSITION COMPUTATION (Deterministic)                │  │
│  │  • target_position = weight × nav / price            │  │
│  │  • delta = target_position - current_position        │  │
│  │                                                       │  │
│  │  → ORDER CONSTRUCTION                                │  │
│  │  • qty = floor_to_lot(delta, lot_size)              │  │
│  │  • Check: qty × price >= min_notional               │  │
│  │                                                       │  │
│  │  → ORDER ROUTING                                    │  │
│  │  • Single-writer lock enforcement                   │  │
│  │  • Asynchronous order queue                         │  │
│  │                                                       │  │
│  │  → FILL TRACKING                                    │  │
│  │  • Wait for broker acknowledgment                   │  │
│  │  • Persist event to ledger.jsonl                    │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
│  Output: Orders routed, fills tracked, ledger updated      │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│    ENGINE 3: RISK OVERLAY & MACRO PROTECTION               │
│                                                             │
│  Input: Current portfolio state + fills                    │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  CONSTRAINT EVALUATION                              │  │
│  │  • Gross notional <= $250K                         │  │
│  │  • Sector concentration <= cap                     │  │
│  │  • Currency exposure within limits                 │  │
│  │                                                       │  │
│  │  → DECISION: ALLOW / REDUCE / HALT                 │  │
│  │                                                       │  │
│  │  REGIME DETECTION (Macro FX Guard)                 │  │
│  │  • Market-volatility regime                        │  │
│  │  • Correlation structure                           │  │
│  │                                                       │  │
│  │  → Adaptive position sizing                        │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
│  Output: Risk decision and exposure adjustment             │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│      MONITORING & RECONCILIATION (PM-015, Python)          │
│                                                             │
│  • Scheduled ledger-vs-broker state validation            │
│  • Prometheus operational metrics                         │
│  • Grafana dashboards for NAV and exposure                │
│  • Reconciliation and controlled recovery procedures      │
└─────────────────────────────────────────────────────────────┘
```

---

## Concurrency & Asynchronous Design

### Single-Writer Principle

**Problem:** Concurrent process instances and broker sessions can create order-state desynchronization.

**Solution:** A lock-file guard is enforced at execution-core startup.

Conceptual startup flow:

```text
if file_exists("/tmp/live_loop.lock"):
    abort_startup("Another execution instance is active")
else:
    create_lock_file(pid)

on_shutdown:
    remove_lock_file()
```

**Effect:** Only one execution instance is permitted to submit orders and append execution events during a session.

### Asynchronous Order Routing

Order submission is separated from upstream signal processing through an asynchronous queue.

```text
Signal Input → Order Queue
├─ Signal processing: Produces target weights
├─ Order construction: Converts targets into order deltas
├─ Order submission: Asynchronous broker-adapter task
└─ Fill tracking: Broker-event processing loop
```

Backpressure is applied when the order queue reaches its configured capacity.

**Why Rust Core:** Rust and Tokio are used for typed execution logic, asynchronous orchestration, explicit ownership, and concurrency control.

---

## Concurrency Guarantees: Reconciliation vs. Execution

### Original Failure Mode

**Incident PM-015:** Duplicate execution processes were started following a service restart, allowing concurrent access to the same paper-trading account.

### Current Safeguard

`auto_reconcile.py` is not intended to operate as a concurrent order writer. Its execution is bounded to periods after the execution session has ended.

### Execution Windows

```text
SYSTEM_RUNNER (Rust, Continuous)
├─ Active order routing during the NYSE core trading session
├─ Regular session reference: 09:30 - 16:00 ET
├─ Runtime schedule derived from America/New_York timezone
├─ Acquires /tmp/live_loop.lock at startup
└─ Holds the lock for the entire execution session

auto_reconcile.py (Python, Scheduled)
├─ Runs only after the execution session has ended
├─ Waits for /tmp/live_loop.lock to be released
├─ Acquires exclusive lock (/tmp/reconcile.lock)
├─ Executes ledger and broker-state validation
├─ If divergence is detected: Appends a RECONCILIATION event
└─ Releases the lock before the next execution session
```

Exchange holidays and early-close sessions must be reflected in the scheduler configuration before any live-capital deployment.

### Lock Hierarchy

```text
SYSTEM_RUNNER
└─ /tmp/live_loop.lock
   Held throughout the execution session

auto_reconcile.py
└─ /tmp/reconcile.lock
   Held during the reconciliation cycle

INVARIANT
└─ Execution and reconciliation must not write concurrently
```

If the execution lock remains active when reconciliation is scheduled, reconciliation waits until the execution process has released it.

### Reconciliation: Read Existing State, Append Corrections

`auto_reconcile.py` workflow:

```text
1. ACQUIRE /tmp/reconcile.lock
2. READ ledger.jsonl
3. READ broker-reported orders, fills, and positions
4. COMPARE internal and broker-reported state
5. IF divergence is detected:
     APPEND a new RECONCILIATION event
6. VERIFY the resulting state
7. RELEASE /tmp/reconcile.lock
```

Historical ledger entries are not modified or deleted. Corrections are represented by newly appended events.

---

## Data Structures

### Order Payload

```json
{
  "order_id": "ORD_20260625_001",
  "timestamp_utc": "2026-06-25T14:30:00.123456Z",
  "instrument": "AAPL",
  "side": "BUY",
  "qty": 100,
  "order_type": "LIMIT",
  "limit_price": 195.50,
  "tif": "DAY",
  "min_notional": 19550.00,
  "checksum": "sha256_hex(...)"
}
```

### Position State

```json
{
  "ticker": "AAPL",
  "current_qty": 100,
  "avg_entry_price": 195.23,
  "current_price": 195.50,
  "position_pnl": 27.00,
  "position_side": "LONG",
  "last_rebalance_date": "2026-06-25",
  "sector": "TECHNOLOGY"
}
```

### Ledger Entry

```json
{
  "date": "2026-06-25",
  "timestamp_utc": "2026-06-25T14:30:15.456789Z",
  "event_type": "FILL",
  "order_id": "ORD_20260625_001",
  "ticker": "AAPL",
  "filled_qty": 100,
  "filled_price": 195.49,
  "commission": 12.50,
  "nav_post_trade": 250903.16,
  "previous_hash": "sha256_hash_of_previous_entry",
  "entry_hash": "sha256_hash_of_this_entry"
}
```

### Reconciliation Event

```json
{
  "date": "2026-06-25",
  "timestamp_utc": "2026-06-25T16:15:30.123456Z",
  "event_type": "RECONCILIATION",
  "divergence_type": "MISSING_FILL",
  "affected_order_id": "ORD_20260625_001",
  "action_taken": "Requested fill confirmation from IBKR API",
  "resolution": "FILL_CONFIRMED",
  "previous_hash": "sha256_hash_of_previous_entry",
  "entry_hash": "sha256_hash_of_this_entry"
}
```

### Hash-Chain Design

Each ledger entry contains the hash of the previous entry.

New reconciliation events are appended rather than rewriting historical entries. Chained SHA-256 integrity checks make subsequent alteration detectable when the ledger is validated from its trusted starting point.

---

## Invariants & Contracts

### Engine 1: Signal → Engine 2: Execution

**Contract:**

- Input: Daily signal `{ticker, weight}`
- Valid ticker: Member of the configured universe
- Valid weight: `[-1, 1]`
- Output: Validated target positions and broker orders
- Portfolio constraint: Target weights normalized to the configured gross exposure

### Engine 2: Execution → Engine 3: Risk

**Contract:**

- Input: Acknowledged orders, executions, and current positions
- Output: Risk decision: `ALLOW`, `REDUCE`, or `HALT`
- Guarantee: Risk state is evaluated before the next execution cycle

### Engine 3: Risk → Engine 2: Execution

**Contract:**

- Input: Risk-control command
- Output: Position-reduction, order-blocking, or execution-halt instruction
- Guarantee: Risk commands take precedence over new signal-driven orders

Latency targets may be tracked operationally but are not presented as hard guarantees unless verified by measurement.

---

## Failure Modes & Recovery

### Failure: Broker API Timeout

**Detection:** Broker request does not receive a response within the configured timeout.

**Recovery:**

```text
Attempt 1: Wait 1 second, retry
Attempt 2: Wait 2 seconds, retry
Attempt 3: Wait 4 seconds, retry
...
Maximum retry count reached:
  Stop further submissions
  Record failure state
  Escalate alert
```

Retry behavior is bounded to prevent unrestrained repeated order submission.

### Failure: State Divergence — PM-015

**Detection:** Internal ledger-derived state differs from broker-reported state.

**Recovery Process:**

```text
1. ACQUIRE /tmp/reconcile.lock
2. Snapshot internal and broker-reported state
3. Identify divergence type:

   MISSING FILL
   └─ A locally tracked order is missing an expected fill

   UNTRACKED BROKER ORDER
   └─ An order exists at the broker but not in local state

   STALE POSITION
   └─ Local and broker-reported quantities differ

4. Determine the permitted recovery action
5. Append a RECONCILIATION event
6. Re-read broker state
7. Verify whether the divergence was resolved
8. Record the reconciliation report
9. RELEASE /tmp/reconcile.lock
```

Broker-reported state is treated as the external source of truth for actual paper-account positions and executions.

Recovery actions are logged and bounded by the configured operational controls.

---

## Testing Strategy

### Determinism Tests — Rust Core

```text
Invariant:
  Identical validated inputs produce identical order-construction outputs

Test:
  Run order construction repeatedly with the same:
  • signal
  • portfolio state
  • price snapshot
  • configuration

Assert:
  Serialized outputs are identical

Result:
  PASSED
```

### Idempotence Tests — Python Reconciliation

```text
Invariant:
  Running reconciliation against an already reconciled state
  does not produce additional corrective events

Test:
  state_v1 = reconcile(ledger, broker_state)
  state_v2 = reconcile(state_v1, broker_state)

Assert:
  state_v1 == state_v2

Result:
  PASSED
```

### Lock-Contention Tests

```text
Invariant:
  Reconciliation does not write while execution holds its session lock

Test:
  1. Start SYSTEM_RUNNER
  2. Verify /tmp/live_loop.lock is active
  3. Trigger auto_reconcile.py
  4. Verify reconciliation waits
  5. Release execution lock
  6. Verify reconciliation proceeds
  7. Verify no overlapping ledger writes

Result:
  PASSED
```

### Failure-Injection Tests

```text
Test:
  Simulate broker API errors and timeouts

Assertions:
  • Retry policy uses bounded exponential backoff
  • Duplicate submissions are not generated
  • Failure state is logged
  • Ledger history is not corrupted
  • Alert state is raised after terminal failure

Result:
  PASSED
```

---

## Monitoring & Observability

### Prometheus Metrics

Metrics may include:

```text
qsx_order_latency_ms
qsx_fill_rate
qsx_order_rejection_rate
qsx_cost_accumulation_bps
qsx_reconciliation_status
qsx_divergence_count
qsx_uptime_seconds
qsx_lock_contention_wait_ms
```

Metric types and labels are defined by the active exporter implementation.

### Grafana Dashboards

1. **Live Operations**
   - NAV
   - Open positions
   - Gross and net exposure
   - Order-state status
   - Broker connectivity

2. **Risk**
   - Gross-notional utilization
   - Concentration metrics
   - Risk-control status
   - Reconciliation status

3. **Incidents**
   - Alert history
   - Divergence events
   - Recovery status
   - Lock-contention events

---

## Deployment Topology

```text
┌──────────────────────────────────────────────┐
│          Paper-Trading Environment           │
│                                              │
│  ┌────────────────────────────────────────┐  │
│  │  SYSTEM_RUNNER (Rust Core)             │  │
│  │                                        │  │
│  │  Session: NYSE core trading hours      │  │
│  │  Timezone: America/New_York            │  │
│  │                                        │  │
│  │  • Loads daily signal                  │  │
│  │  • Builds target positions             │  │
│  │  • Routes orders via broker adapter    │  │
│  │  • Acquires /tmp/live_loop.lock        │  │
│  │  • Runs Tokio asynchronous event loop  │  │
│  │  • Holds lock for entire session       │  │
│  └────────────────────────────────────────┘  │
│                     │                        │
│                     │ appends events         │
│                     ▼                        │
│  ┌────────────────────────────────────────┐  │
│  │  Ledger (ledger.jsonl)                 │  │
│  │                                        │  │
│  │  • Append-only event history           │  │
│  │  • Chained SHA-256 integrity checks    │  │
│  └────────────────────────────────────────┘  │
│                     ▲                        │
│                     │ reads state            │
│                     │                        │
│  ┌────────────────────────────────────────┐  │
│  │  auto_reconcile.py (Python)            │  │
│  │                                        │  │
│  │  Runs after execution session closes   │  │
│  │                                        │  │
│  │  • Waits for live-loop lock release    │  │
│  │  • Acquires /tmp/reconcile.lock        │  │
│  │  • Compares ledger and broker state    │  │
│  │  • Appends reconciliation events       │  │
│  │  • Releases lock before next session   │  │
│  └────────────────────────────────────────┘  │
│                                              │
│  ┌────────────────────────────────────────┐  │
│  │  Prometheus + Grafana                  │  │
│  │                                        │  │
│  │  • Metrics collection                  │  │
│  │  • Operational monitoring             │  │
│  │  • Dashboarding and alerting           │  │
│  └────────────────────────────────────────┘  │
│                                              │
│  ┌────────────────────────────────────────┐  │
│  │  Persistent State                     │  │
│  │                                        │  │
│  │  • ledger.jsonl                       │  │
│  │  • lock files                         │  │
│  │  • Prometheus data directory          │  │
│  └────────────────────────────────────────┘  │
└──────────────────────────┬───────────────────┘
                           │
                           ▼
              ┌────────────────────────────┐
              │  Interactive Brokers      │
              │  Paper-Trading Account    │
              │                            │
              │  • Order submission        │
              │  • Position reporting      │
              │  • Fill acknowledgment     │
              └────────────────────────────┘
```

### Key Architectural Invariants

- `SYSTEM_RUNNER` and `auto_reconcile.py` do not write concurrently
- Lock files prevent overlapping execution sessions
- Historical ledger entries are not mutated or deleted
- Corrections are represented by newly appended reconciliation events
- Broker state is re-read after corrective action
- Risk-control commands take precedence over new signal-driven orders

---

## Technology Stack

| Component | Language | Framework | Purpose |
|---|---|---|---|
| Execution Core | Rust | Tokio | Asynchronous orchestration, typed execution logic, and single-writer control |
| Signal Research | Python | NumPy/Pandas | Statistical research, feature computation, and signal generation |
| Reconciliation | Python | asyncio | Concurrent broker I/O, scheduled reconciliation, and state management |
| Monitoring | Prometheus/Grafana | Rust exporter | Metrics collection, operational monitoring, dashboarding, and alerting |
| Broker Integration | Python | ib-insync | IBKR connectivity, order submission, fills, and portfolio-state retrieval |

### Architecture Rationale

Rust is used for execution orchestration and concurrency control.

Python supports:

- research workflows
- feature computation
- signal generation
- broker integration
- scheduled reconciliation

This separation preserves research velocity while isolating execution-state responsibilities.

---

## Future Evolution

1. **Exchange Calendar Integration**  
   Introduce explicit handling for exchange holidays and early-close sessions.

2. **Expanded Recovery Testing**  
   Add more broker-disconnect, partial-fill, stale-order, and process-restart scenarios.

3. **Multi-Broker Abstraction**  
   Introduce a broker-adapter interface before adding additional execution venues.

4. **Controlled Live-Capital Readiness Review**  
   Require independently verified controls, reproducible research results, and staged deployment criteria before any live-capital transition.

---

## References

- PM-015 Post-Mortem: `docs/PM-015-INCIDENT.md`
- Research Methodology: `research/methodology.md`
- Live Monitoring: Internal Grafana dashboard
