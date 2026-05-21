# Note 5 — In-process scheduler tick

**Status:** Accepted
**Context chapter:** [8. Scheduled decks](../08-scheduled-decks.md)

## Context

Scheduled decks need a recurring "fire at the right time" mechanism.
The common choices are:

1. **External queue + worker fleet.** Producer enqueues a job at the
   target time (or uses a delay queue); workers consume.
2. **External cron service** hits an HTTP endpoint at scheduled
   times.
3. **In-process scheduler** (APScheduler) running inside the
   existing application.

## Decision

Use an in-process APScheduler tick that wakes every minute, scans the
`SCHEDULE` table for due rows, and spawns runs as background tasks.

```
   APScheduler tick (every minute)
                |
                v
   SELECT due schedules
                |
                v
   Spawn N background runs
                |
                v
            Next tick
```

## Alternatives considered

| Option | Pros | Cons | Rejected because |
|--------|------|------|------------------|
| Celery + Redis / RQ | Battle-tested; retries, dead letters, large fleets | Adds a broker + worker tier to operate | Operational cost outweighs benefit at this volume |
| Managed scheduler hitting an HTTP endpoint | No new process | Adds external dependency for a feature that lives entirely in our app; auth on the endpoint becomes a concern | More moving parts than necessary |
| System cron + script | Trivial | Cron environment is not the app environment; requires CLI entry point | Operationally clumsy |
| **In-process APScheduler (chosen)** | Zero extra infra; uses same code paths as live generation | Tightly coupled to single application origin | Right size for the workload |

## Why minute-level granularity is fine

User-perceived schedules are "Monday morning at 9," not "9:00:00." A
minute of jitter is below the perception threshold. No need for
sub-minute precision.

## Consequences

- Runs share the process with HTTP and Socket.IO traffic. The
  concurrency model (chapter 9) absorbs this without contention
  because LLM calls are I/O-bound.
- If the process restarts mid-tick, in-flight runs are lost. The run
  row records `started_at` without `finished_at`; an operator query
  surfaces stalled runs for re-trigger.
- A single process means the maximum concurrent runs is bounded by
  the process's LLM-call concurrency.

## When this would fail

- **Volume.** If scheduled run volume grew 100×, an external worker
  tier becomes attractive.
- **Strict deadlines.** If users started expecting "fired at exactly
  9:00:00," APScheduler is not the right tool.

## Revisit when

- Run volume per day enters the tens-of-thousands range.
- Deploy-time outage of in-flight runs becomes a customer-visible
  problem.
