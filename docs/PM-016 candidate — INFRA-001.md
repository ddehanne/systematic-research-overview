### PM-016 — Zombie process regeneration through systemd cgroup semantics

**Date detected:** 2026-07-02
**Date resolved:** 2026-07-02
**Environment:** IBKR paper-deployment (DU6082389)

**Symptom:**
Service restarts left python child processes detached from the systemd 
cgroup, retaining broker connections (clientId, TCP sockets to IB Gateway). 
Each subsequent restart attempt failed with "Error 326: clientId already 
in use". Observed: 42 NRestarts on trading-loop.service, 839+ on the 
legacy trading-loop-live.service.

**Root cause:**
Unit configuration used `KillMode=mixed` with `TimeoutStopSec=60`. On 
`systemctl stop`, SIGTERM was sent to the main process (the shell 
wrapper) but not propagated to its Python child. The wrapper exited 
before the child could complete `ib_insync` async cleanup, leaving 
the child reparented to PID 1, outside the service cgroup, retaining 
its IBKR socket connections.

**Fix:**
Systemd drop-in `/etc/systemd/system/trading-loop.service.d/hardening.conf`:

    [Unit]
    StartLimitIntervalSec=300
    StartLimitBurst=3
    
    [Service]
    KillMode=control-group
    KillSignal=SIGTERM
    TimeoutStopSec=10
    Restart=on-failure
    RestartSec=5

`KillMode=control-group` propagates SIGTERM to every process in the 
service cgroup. `StartLimitBurst=3` prevents silent restart loops 
by requiring manual `systemctl reset-failed` after 3 failures in 5 
minutes.

**Validation:**
- Isolated test (`cgroup-test.service` with two `sleep` children): 
  cgroup destroyed on stop, all children reaped.
- Real service halt: cgroup destroyed, `ss -tnp | grep 4002` empty, 
  `ps -ef | grep live_loop` empty within 3 seconds of `systemctl stop`.

**Invariant established:**
∀ t : `systemctl stop trading-loop.service` at time t implies 
∀ p ∈ cgroup(trading-loop.service) : p.dead(t + TimeoutStopSec).

**Residual scope not covered:**
- IB Gateway session state broker-side (unrelated, INFRA-003 scope)
- clientId preflight before adapter.connect() (unrelated, separate ticket)
- Alerting on inactive service or NRestarts threshold breach (separate)

