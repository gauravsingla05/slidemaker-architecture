# 8. Scheduled Decks

A user can configure SlideMaker to generate a deck on a recurring
schedule: "every Monday at 9am, build a deck from this Google Sheet
of last week's metrics." This chapter describes the moving parts that
make that work without an external job queue.

## 8.1 What a schedule is

A schedule is a persistent specification with three parts:

1. **Cadence** — daily, weekly (with weekday), or monthly (with
   day-of-month), plus a timezone-anchored time-of-day.
2. **Source** — what data feeds the generation: CSV upload, Google
   Sheet (file-scoped), or web URLs (up to a small number).
3. **Prompt** — the user's natural-language description of the deck
   the AI should make from the source data.

Each schedule has a `next_run_at` the system uses to decide when to
fire.

```
       SCHEDULE                       RUN
   +----------------+              +-----------------+
   | id             |     1..N     | id              |
   | user_id        | -----------> | schedule_id     |
   | cadence        |              | started_at      |
   | source_kind    |              | finished_at     |
   | source_ref     |              | status          |
   | prompt         |              | deck_id         |
   | next_run_at    |              | error (if any)  |
   | paused         |              +-----------------+
   | auto_run_streak|
   +----------------+
```

## 8.2 The tick

One scheduler exists: an in-process APScheduler job that fires on a
fixed interval (every minute). Each tick scans for schedules whose
`next_run_at` is in the past and processes them.

```
   Tick (every ~1 min)
        |
        v
   SELECT schedules
   WHERE next_run_at <= now
     AND paused = false
        |
        v
   +-------------+
   | for each:   |
   |   - claim   |   <-- update next_run_at to next future occurrence
   |   - spawn   |       (so the next tick does not double-fire)
   |     run     |
   +-----+-------+
         |
         v
       wait
   for next tick
```

The tick is intentionally short and idempotent: claim, recompute
`next_run_at`, spawn the run asynchronously, return.

### Why in-process

External job queues (RQ, Celery, SQS) add an operational tier — a
broker to run, a worker fleet to scale, a dead-letter queue to
monitor. The scheduled-decks workload is small and not latency-
sensitive (a minute of jitter is invisible). APScheduler in-process
is sufficient and avoids the extra moving parts. Trade-offs in
[ADR-005](decisions/ADR-005-in-process-scheduler.md).

## 8.3 A run

A run is one execution of a schedule. Five stages, four reused from
the generation pipeline (chapter 2).

```
          +---------+
          | Pending |
          +----+----+
               | tick fires
               v
          +-----------+
   +----- | Fetching  |
   |      +-----+-----+
   |            | source OK
   |            v
   |      +-----------+
   |----- | Generating|
   |      +-----+-----+
   |            | OK
   |            v
   |      +-----------+
   |      |  Saving   |
   |      +-----+-----+
   |            | OK
   |            v
   |      +-----------+
   |      | Notifying |
   |      +-----+-----+
   |            |
   |            v
   |      +-----------+
   |      |  Success  |
   |      +-----------+
   |
   |  (any stage failure)
   v
   +---------+
   | Failed  |
   +---------+
```

| Stage | What happens |
|-------|--------------|
| Fetching | Source connector reads CSV / Sheet / URL(s) and produces a structured context block. |
| Generating | The generation pipeline (chapter 2) runs with the user's prompt and that context. |
| Saving | The resulting deck is stored as a new deck under the user. |
| Notifying | A best-effort email tells the user the deck is ready (failure is logged but does not fail the run). |

## 8.4 Data source connectors

Each connector implements the same interface: "given a source spec,
fetch and return a structured context block." The block is appended
to the generation prompt as a fenced section.

```
                       source_kind?
                            |
            +---------------+---------------+
            |               |               |
           csv             sheet           url
            |               |               |
            v               v               v
       +---------+    +-----------+    +-----------+
       |  Read   |    | Read sheet|    | Fetch URLs|
       |  CSV    |    | via Drive |    |           |
       | (upload)|    | file scope|    | extract   |
       +----+----+    +-----+-----+    | text      |
            |               |          +-----+-----+
            +-------+-------+----------------+
                    |
                    v
            structured context block
                    |
                    v
            generation pipeline
```

### CSV connector

The user uploads a CSV during schedule creation. The file is stored
once; each run reads it. No per-run user interaction.

### Sheet connector

Uses the file-scoped Drive access from
[chapter 7](07-oauth-and-token-vault.md). The schedule stores the
sheet's file id; each run reads the current contents at fire time, so
the deck reflects whatever the sheet says that morning.

If the refresh token has been revoked or the sheet has been
deleted / unshared, the run fails with a clear error and the schedule
is auto-paused after enough consecutive failures (see § 8.6).

### URL connector

A small number of URLs (up to three) can be attached to a schedule.
The fetcher uses a browser-like user agent, follows redirects,
detects Cloudflare challenge pages as failures (rather than treating
the challenge HTML as content), and extracts the main text.

## 8.5 Run-now vs scheduled

Users can trigger a run immediately to test a schedule. The same code
path is used; the difference is only in how it was triggered.

```
   User                Frontend            Backend             Worker
    |                     |                   |                    |
    | "Run now"           |                   |                    |
    | ------------------> |                   |                    |
    |                     | POST /run         |                    |
    |                     | ----------------> |                    |
    |                     |                   | spawn run -------> |
    |                     | <-- 202 (run_id)  |                    |
    |                     |                   |                    |
    |                     | poll /runs        |                    |
    |                     | ----------------> |                    |
    |                     | <-- status        |                    |
    |                     |   ...repeat...    |                    |
    |                     |                   | <-- status update -|
    | <-- success/failure |                   |                    |
```

The HTTP response is 202 (Accepted); the run is asynchronous; the
frontend polls until the latest run has a terminal status. Simpler
than a streaming progress channel and good enough — runs take about
thirty seconds at most.

## 8.6 Guardrails

A recurring AI workload can burn through tokens silently if something
is broken. Two guardrails keep that bounded.

### Auto-pause on streak

Each schedule has an `auto_run_streak` counter that increments on
every run that succeeds *without user intervention*. When the streak
hits a threshold (e.g., ten), the schedule is paused with a clear
message: "this schedule has run ten times in a row without you
looking at the output; confirm you still want it before we keep
running."

This is a *waste* guard, not a failure guard. A schedule that
quietly produces decks the user never opens is consuming compute for
no value.

```
   Successful run
         |
         v
   streak += 1
         |
         v
   streak >= N ? ---- no ---> wait for next tick
         |
        yes
         |
         v
   paused = true
         |
         v
   notify user

   User opens the resulting deck
         |
         v
   streak = 0
```

### Per-source row limit

The Sheet connector caps the number of rows it includes in the
context block. A user pointing a schedule at a million-row sheet
would otherwise produce a prompt larger than any LLM context window.
The cap is enforced at fetch time; a warning is recorded.

## 8.7 Why the same generation pipeline

Scheduled runs reuse the exact pipeline from chapter 2. No
"scheduled-mode" variant. A divergent pipeline would mean two places
to fix bugs, two places to tune prompts, and two places where output
quality could drift.

The only difference: the scheduled path skips the SSE stream. There
is no live user watching, so the run consumes the stream server-side,
waits for `done`, and saves the deck.

## 8.8 Connections

- The generation pipeline reused here is chapter 2.
- The Google Drive file-scope used by the Sheet connector is chapter
  7.
- The `SCHEDULE` and `RUN` tables are described in chapter 6.
- Scheduler-vs-queue trade-off:
  [ADR-005](decisions/ADR-005-in-process-scheduler.md).
