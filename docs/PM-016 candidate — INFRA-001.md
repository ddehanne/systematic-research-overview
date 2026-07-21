# PM-016 — Zombie process regeneration through systemd cgroup semantics

**Date detected:** 2026-07-02
**Date resolved:** 2026-07-02
**Environment:** IBKR paper account

---

## Symptom

Service restarts left Python child processes detached from the systemd
cgroup, retaining broker connections (clientId, TCP sockets to IB
Gateway). Each subsequent restart attempt failed with **"Error 326:
clientId already in use"**. Observed: 42 NRestarts on the execution
service, 839+ on a legacy predecessor unit.

## Root cause

The unit used `KillMode=mixed` with `TimeoutStopSec=60`. Under
`mixed`, SIGTERM is sent to the **main process only**; child processes
receive SIGKILL only at timeout expiry — and children that had
double-forked or detached from the main process group survived the
stop entirely, keeping their broker sessions alive. The next start
then collided with the zombie's clientId.

The failure is a **semantics gap, not a bug**: `systemctl stop`
returned success while broker-connected processes kept running.
"Stopped" and "no longer holds external resources" are different
claims; the unit configuration only guaranteed the first.

## Remediation (INFRA-001)

- `KillMode=control-group` — stop terminates the **entire cgroup**,
  no survivors regardless of forking behavior
- `KillSignal=SIGTERM`, `TimeoutStopSec=10` — bounded graceful
  shutdown, then guaranteed kill
- systemd start-rate limiting retained as a backstop against
  uncontrolled restart loops

Validated on the live paper-deployment service: post-fix, stop leaves
zero surviving processes and zero broker sessions (verified via
process table and Gateway connection list).

## Lessons

1. **"Stopped" must be verified at the resource level, not the
   service level.** A green `systemctl stop` proves nothing about
   sockets and sessions held by descendants.
2. **Process-lifecycle semantics are part of the trading system's
   correctness surface** — a cgroup detail caused broker-session
   exhaustion, which is an execution-layer failure.
3. This incident preceded and informed the later resurrection-
   mechanisms investigation (PM-018): both belong to the same class —
   *the system's actual process state diverging from the operator's
   intended state.*
