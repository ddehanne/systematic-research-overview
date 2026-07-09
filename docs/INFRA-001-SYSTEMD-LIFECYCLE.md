# INFRA-001 — systemd Process Lifecycle Hardening

## Status

DONE — validated on the real IBKR paper-deployment service.

## Objective

Ensure that stopping the execution service terminates the complete service cgroup and limits uncontrolled restart loops through systemd start-rate controls.

## Implemented Controls

- `KillMode=control-group`
- `KillSignal=SIGTERM`
- `TimeoutStopSec=10`
- `Restart=on-failure`
- `RestartSec=5`
- `StartLimitIntervalSec=300`
- `StartLimitBurst=3`

## Validation Evidence

- isolated cgroup test: passed;
- real service stop: passed;
- service cgroup destroyed after stop;
- no residual executor process observed;
- executor-side broker-related sockets were no longer observed after service stop;
- service state confirmed as `inactive/dead`;
- automatic boot start disabled.

## Scope

This control validates service process-lifecycle behavior only.

It does not establish:

- broker-state reconciliation;
- order idempotence;
- single-writer enforcement;
- complete session cleanup inside IBKR;
- production readiness.

## Next Dependencies

- INFRA-002 — local single-writer control;
- INFRA-003 — broker reconciliation gate.

