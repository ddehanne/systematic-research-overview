# INFRA-001 — Service Stop Hardening

## Status

Completed 2026-07-02 on the paper deployment. **Superseded in July 2026:** after [PM-018](PM-018.md), automatic restart was disabled entirely, the service units were masked, and the trading loop was later launched manually as a standalone process. The settings below describe the July 2 state only.

## Objective

Ensure that stopping the execution service terminates every process it started, and limit uncontrolled restart loops.

## Controls applied (July 2 state)

- `KillMode=control-group`
- `KillSignal=SIGTERM`, `TimeoutStopSec=10`
- `Restart=on-failure`, `RestartSec=5`. This setting was later identified as one of the automatic restart paths and removed (PM-018).
- `StartLimitIntervalSec=300`, `StartLimitBurst=3`

## Recorded validation

- Isolated control-group test: passed (three processes reaped).
- Real service stop: passed. The control group was destroyed, sockets were released, and no orphan process remained.
- Service state afterwards: inactive, not enabled at boot.

## Scope and limits

This control covered local process lifecycle only. It did not establish broker-side session cleanup (the gateway still held a session afterwards), broker-state reconciliation, order idempotence, single-writer enforcement, or production readiness.

## Follow-ups

- INFRA-002: single-writer lock on the shared resource (July 10).
- INFRA-003: broker reconciliation gate at boot (July 14).
