# PM-016 — Processes Surviving a Service Stop (Paper Account)

**Environment:** IBKR paper account
**Date:** 2026-07-02 (detected and mitigated the same day)
**Later status:** the service-based setup was replaced in July 2026; see [PM-018](PM-018.md).

> **Revision note.** An earlier public version stated a specific prior service configuration, a restart count on a predecessor unit, and that the fix left zero broker sessions. The project records do not support those details, and the record contradicts the last one. They are withdrawn.

## Symptom

Repeated start attempts failed with IBKR **Error 326 ("clientId already in use")**. The service had restarted 42 times. One leftover process held a broker client ID through open sockets for almost 17 hours.

## Findings

The session record identified **five distinct root causes** behind the same error message, one of which was this incident:

- client IDs hard-coded across several files;
- processes surviving a service stop because of the service manager's kill settings;
- gateway sessions leaking across crashes;
- no connectivity probe before connecting;
- defects introduced by the session's own patches.

The error message itself was misleading: it named the same client ID whatever the actual collision.

The patches made during that session were rolled back.

## Remediation (INFRA-001)

The service was changed to terminate its whole control group on stop, with a bounded stop timeout and start-rate limiting (see [INFRA-001](INFRA-001-SYSTEMD-LIFECYCLE.md)).

**Recorded validation:**

- an isolated control-group test passed (three processes reaped);
- a real service stop left no orphan process, and the local sockets were released.

**Recorded residual:** the gateway still held the session on the broker side. The fix therefore covered local process cleanup, not broker-session cleanup.

## Lessons

- "Stopped" has to be verified at the resource level (processes, sockets, broker sessions), not from the service manager's status.
- One error message can hide several causes; each needs its own confirmation.
