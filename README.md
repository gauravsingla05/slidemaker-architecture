# SlideMaker — Architecture Notes

A walk-through of how [SlideMaker.app](https://slidemaker.app) is put
together. SlideMaker is a personal learning project that turns a
prompt into a slide deck using AI. This repository explains the
design with diagrams, walks through the moving parts, and writes up
a few of the more instructive bugs that came up along the way.

The intent is educational. If you are learning how AI-assisted apps,
real-time collaboration, or streaming pipelines fit together, the
walk-through below should be readable without prior context. Each
section ends with a link to a deeper chapter.

All diagrams are plain ASCII so they render the same way in any
viewer.

---

## What SlideMaker does

```
   user types a prompt
   (optionally attaches a document)
              |
              v
   +---------------------------+
   |   AI generation pipeline  |
   |   outline -> per slide    |
   +-------------+-------------+
                 |
                 v
        a 10-slide deck
                 |
                 v
   +---------------------------+
   |   edit it, share it,      |
   |   collaborate live,       |
   |   export to PPT / PDF /   |
   |   Google Slides           |
   +---------------------------+
```

A single feature you might also use: the user can configure recurring
runs ("every Monday morning, build a deck from this Google Sheet")
that fire on their own.

---

## The big picture

Three long-running pieces — a browser app, an application server, and
a database — plus a small set of external services.

```
                +----------------+
                |   Edge proxy   |
                |  (TLS + CDN)   |
                +-------+--------+
                        |
            +-----------+-----------+
            |                       |
            v                       v
    +----------------+      +----------------+
    |    Browser     |      |   Browser      |
    | (Next.js app)  | ...  | (Next.js app)  |
    +--------+-------+      +--------+-------+
             |                       |
             |  HTTP / SSE / WS      |
             +-----------+-----------+
                         |
                         v
                +------------------+
                |  Application     |
                |  server          |
                |                  |
                |  + HTTP handlers |
                |  + Socket.IO    |
                |  + Scheduler    |
                +-+-------+------+-+
                  |       |      |
                  v       v      v
           +---------+ +-----+ +-------------------+
           | MySQL   | |Token| | External:        |
           |         | |store| |  LLM providers   |
           |         | |     | |  Google OAuth    |
           |         | |     | |  Google Drive    |
           +---------+ +-----+ +-------------------+
```

The application server is one process. A microservice split was
considered and not pursued; the operational cost of more processes is
greater than the benefit at this size.

→ See [01. System overview](01-system-overview.md) for the full tour.

---

## Three request paths

The system speaks three different "shapes" of request depending on
the workload.

```
    sync HTTP                SSE stream              WebSocket
   -----------              -----------             -----------
        |                        |                       |
    ~ 100 ms                  ~ 30 s                  ongoing
    request                end-to-end             round-trip ~30 ms
        |                        |                       |
        v                        v                       v
    one JSON           many event lines         many small frames
    response           ("outline_ready",        ("yjs_update",
                       "slide_ready", ...)     "awareness", ...)
        |                        |                       |
        v                        v                       v
   auth, save,            AI generation        real-time collab
   load deck              (chapter 2)          (chapter 3)
```

Sync HTTP is for everything short. SSE is for AI generation, so the
user sees progress instead of staring at a spinner for thirty
seconds. WebSocket is for the editor while collaboration is active.

---

## How a deck gets generated

The pipeline is five stages. Two stages call the LLM; three are pure
deterministic transforms.

```
   +--------+    +---------+    +-----------+    +-----------+    +-----------+
   | Ingest | -> | Outline | -> | Per-slide | -> |  Layout   | -> |  Render   |
   |        |    |  (LLM)  |    |  Expand   |    |  Select   |    |   JSON    |
   |        |    |         |    |  (LLM xN) |    |           |    |           |
   +--------+    +---------+    +-----------+    +-----------+    +-----------+
    ~ 0-2 s        ~ 3 s         ~ 2 s each       < 100 ms         < 200 ms
                                 (parallel)
```

The outline is sent to the client as the *first* event so the user
sees the shape of the deck about three seconds in. The expansion
calls then run in parallel and stream slides as they finish, in
completion order rather than outline order.

```
                +-----------+
                |  Outline  |    emit "outline_ready"
                +-----+-----+
                      |
                      | fan out (parallel)
       +--------+-----+-----+--------+
       |        |           |        |
       v        v           v        v
   +------+ +------+    +------+ +------+
   |Slide | |Slide | .. |Slide | |Slide |
   |  1   | |  2   |    |  N-1 | |  N   |
   +--+---+ +--+---+    +--+---+ +--+---+
      |       |            |        |
      v       v            v        v
       emit each as "slide_ready"
       in completion order
       (client fills slots in the outline)
```

A timeline view of a ten-slide run:

```
   second:   0    5    10    15    20    25    30
             |    |     |     |     |     |     |
   Ingest    XX
   Outline       XXXX
   Slides            XXXXXXXXXXXXXXXX
                     ^ first slides emit here
   Save                                      XX
   Repaint                                    X
```

→ See [02. Generation pipeline](02-generation-pipeline.md) for stage
details, retry logic, and the schema-validation envelope.

---

## How slides are streamed to the browser

SSE over HTTP. Five event types: `start`, `outline_ready`,
`slide_ready` (many), `done`, and `error`.

```
         start
           |
           v
       outline_ready
           |
     +-----+-----+-----+----------+
     |     |     |     |          |
     v     v     v     v          v
  slide  slide slide slide  ...  (on error)
  _ready _ready _ready _ready    error
   (1)    (2)   (N-1)   (N)
     \    |     |     /
      \   |     |    /
       \  |     |   /
        v v     v v
           done
```

Reconnection is built into the browser. If the connection drops mid-
stream, it auto-reconnects with `Last-Event-Id` set to the last id
received; the server skips events the client already has.

→ See [04. Streaming protocol](04-streaming-protocol.md) for the wire
format, reassembly state machine, and timeouts.

---

## How live collaboration works

The hardest part of the system. The editor combines two very
different surfaces: rich text inside text elements (mergeable,
character-level edits) and everything else (positions, charts,
tables — coarse-grained edits that don't have a sensible auto-merge).

The system uses two patterns side by side:

```
                  user edit
                     |
                     v
            +------------------+
            | kind of edit?    |
            +---+----------+---+
                |          |
        text    |          |  position / data / structure
         in     |          |
       element  v          v
        +----------+   +-------------+
        |   Yjs    |   |    Lock     |
        | (CRDT,   |   |  manager    |
        |   merge) |   | (acquire,   |
        +-----+----+   |  mutate,    |
              |        |  release)   |
              |        +------+------+
              |               |
         broadcast       acquire ->
         via Yjs         mutate ->
         protocol        release
              |               |
              +-------+-------+
                      |
                      v
                room broadcast
                to other clients
```

Yjs is a CRDT library. Concurrent text edits merge cleanly without a
central arbiter. For everything that *isn't* text, a pessimistic lock
table on the server gates edits one-at-a-time per element.

```
   Client                Lock Manager                Room
     |                        |                       |
     | acquire(elem_id)       |                       |
     | ---------------------> |                       |
     |                        |                       |
     |  if free:              |                       |
     |     granted            |                       |
     | <--------------------- |  -- element_locked -> |
     |                        |                       |
     |  loop while held:      |                       |
     |     heartbeat -------> |                       |
     |                        |                       |
     |     release ---------> |  -- element_unlock -> |
     |                                                |
     |  if held by other:                             |
     |     denied (owner)                             |
     | <--------------------- |                       |
```

A surprisingly tricky part of this is preventing *echo loops*: a
remote update applied locally must not be re-broadcast back to the
server as if it were a fresh local edit. The guard is a flag set
during the "apply remote" path, checked by the broadcast subscription.

→ See [03. Real-time collaboration](03-collaboration.md) for the room
model, the four channels, the echo-loop guard, and limits.

---

## How a slide actually gets its colors

A slide's final look is a layered resolution. Four sources of style
stack; the highest one with a declared value wins.

```
              highest priority
                    ^
                    |
         +------------------------+
         |   Brand kit override   |   per-user
         +------------------------+
                    |
         +------------------------+
         |    Theme override      |   per-deck
         +------------------------+
                    |
         +------------------------+
         |    Layout default      |   per-layout
         +------------------------+
                    |
         +------------------------+
         |  Component default     |   global
         +------------------------+
                    |
                    v
              lowest priority
```

Elements store styles by *role*, not by literal value. A title
element declares `roleColor: "title"`, not `color: "#0033AA"`. The
resolver maps the role to a concrete value at render time using the
stack above. This is what makes a brand kit change visibly retint
every deck on next view, without rewriting any slides.

→ See [05. Theme and brand kit](05-theme-and-brand-kit.md) for the
layout registry, role catalogue, and why "brand primary should not be
the title color."

---

## What gets stored, and where

A small set of long-lived entities. Most reads and writes touch
fewer than three of them.

```
                                +-------------+
                                |    USER     |
                                +-+-----+-----+
                                  |     |
                       owns       |     |  has
                     +------------+     +-------------+
                     |                                |
                     |                                v
                     v                          +-----------+
                +---------+                     | BRAND_KIT |
                |  DECK   |                     +-----------+
                +----+----+
            +--------+--------+
            |        |        |
   contains |        | uses   | shared via
            v        v        v
       +--------+ +-------+ +--------+
       | SLIDE  | | THEME | | SHARE  |
       +--------+ +-------+ +--------+


       +-----------+      +-------+
       | SCHEDULE  | ---> |  RUN  |
       +-----------+      +---+---+
                              |
                              | produces
                              v
                            DECK


       +-------------+        +---------------+
       | OAUTH_TOKEN |        |  EXPORT_JOB   |
       +-------------+        +---------------+
```

Slides are their own table — not a JSON blob on the deck — so editing
one slide rewrites one row, and per-slide reorder / duplicate /
theme-apply queries stay simple.

→ See [06. Storage model](06-storage-model.md) for the indexing
strategy, soft-delete behavior, and what *is not* stored.

---

## How external auth works

Google integrations (Sheets, Slides export) use incremental consent.
The user authenticates once with basic scopes; each feature requests
its own additional scope the first time it is used.

```
        Sign in
   (openid + email + profile)
           |
           v
       Authenticated
           |
   +-------+--------+
   |                |
   v                v
 "Pick a sheet"   "Export to Slides"
   click             click
   |                |
   |                |
   v                v
   Incremental    Incremental
   consent:       consent:
   drive.file     slides + drive.file
```

Refresh tokens are encrypted at rest (Fernet) with a key held only by
the application process. A leaked database backup alone does not
yield usable tokens.

```
    plaintext refresh_token
            |
            v
       +-----------+      +--------------------+
       |  Fernet   | <--- |  Application-only  |
       |  encrypt  |      |        key         |
       +-----+-----+      +--------------------+
             |
             v
        ciphertext
             |
             v
      stored in MySQL
```

The Sheets integration deliberately uses `drive.file` (non-sensitive
scope, per-file access) instead of `spreadsheets.readonly`
(sensitive scope, all-sheets access). This is why the user has to
confirm a specific sheet via the Picker — `drive.file` cannot list
files it has not been granted access to.

→ See [07. OAuth & token storage](07-oauth-and-token-vault.md) for
the `inc:` state convention, key rotation, and the Picker workaround.

---

## How scheduled decks work

An in-process tick scans for due schedules every minute and spawns
runs as background tasks. Each run reuses the exact generation
pipeline from above; it just consumes the SSE stream server-side
instead of forwarding it to a browser.

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

A run progresses through five states:

```
          Pending
              |
              v
           Fetching   --(source fails)--> Failed
              |
              v
          Generating  --(LLM fails)----> Failed
              |
              v
           Saving     --(DB fails)-----> Failed
              |
              v
          Notifying
              |
              v
           Success
```

A guardrail counter auto-pauses any schedule that has produced ten
consecutive successful runs without the user looking at the result.
The intent is to avoid silently consuming tokens for output nobody
reads.

→ See [08. Scheduled decks](08-scheduled-decks.md) for source
connectors (CSV, Sheet, URL), the run state machine, and guardrails.

---

## How one process handles everything

The application server runs three concurrent workloads side by side
in one process: synchronous HTTP, long-lived Socket.IO sessions, and
short-lived SSE streams. Plus the scheduler tick.

```
   +---------------------------------------------------+
   |             single application process            |
   |                                                   |
   |   +---------+   +-----------+   +---------+       |
   |   |  HTTP   |   | Socket.IO |   |   SSE   |       |
   |   | handler |   |  sessions |   | streams |       |
   |   |  pool   |   |  (idle    |   | (active |       |
   |   | (fast)  |   |   thou-   |   |  during |       |
   |   |         |   |   sands)  |   |  gen)   |       |
   |   +----+----+   +-----+-----+   +----+----+       |
   |        |              |               |           |
   |        +------+-------+------+--------+           |
   |               |                                   |
   |          +----+-----+                             |
   |          | scheduler|                             |
   |          |   tick   |                             |
   |          +----------+                             |
   |                                                   |
   +---------------------------------------------------+
```

The dominant scaling constraint is *socket count*, not CPU. gevent
green threads make thousands of mostly-idle sessions cheap.

One subtle but important detail: Socket.IO is configured to use
*only* the WebSocket transport, not the polling fallback. With
polling enabled and multiple workers, the GET and POST halves of a
poll cycle can land on different workers; the second has no record of
the session created by the first and returns "Invalid session."

```
   With polling + multiple workers (BAD):

      Client
       /   \
      /     \
    GET     POST
     |       |
     v       v
   Worker A  Worker B
     |         |
     |       (no session
   session    record here)
     |         |
     +---------+
                  |
                  v
            "Invalid session"
```

→ See [09. Concurrency model](09-concurrency-model.md) for the async
runtime detection, single-worker constraint, and the rare ways it
would break.

---

## What happens when things go wrong

A short snapshot of the recovery hierarchy:

```
   Prefer:
       automatic per-step retry   -> transient errors
       graceful degradation       -> partial failure
       user-visible escalation    -> everything else

   Avoid:
       silent retries
       best-effort error swallowing
       raw exception text to users
```

The chapter walks through LLM timeouts, OAuth revocation, socket
disconnects mid-collaboration, and the database write failing at the
end of a run.

→ See [10. Failure modes](10-failure-modes.md).

---

## Decision notes (short)

Five one-page notes capturing single design choices and the
alternatives that were considered.

- [Note 1 — WebSocket-only transport](decisions/ADR-001-websocket-only-transport.md) — why polling fallback is disabled.
- [Note 2 — Yjs over Operational Transform](decisions/ADR-002-yjs-over-ot.md) — why CRDT, not OT, for text.
- [Note 3 — Pessimistic locks for non-text elements](decisions/ADR-003-pessimistic-element-locks.md) — why locks instead of OT for shapes / charts / tables.
- [Note 4 — `drive.file` + Picker](decisions/ADR-004-drive-file-and-picker.md) — non-sensitive Google scope and its quirks.
- [Note 5 — In-process scheduler tick](decisions/ADR-005-in-process-scheduler.md) — why APScheduler, not Celery / SQS.

---

## Case studies (real bugs)

Three bugs that were instructive once understood.

- [The infinite echo loop in collaborative editing](case-studies/echo-loop.md) — how React StrictMode double-mount made the "fix" worse.
- [Late login leaking anonymous identity](case-studies/late-login-anonymous-id.md) — why identity is a join key, not an attribute.
- [The brand kit color that ate every title](case-studies/brand-kit-color-override.md) — when the system follows the design and the design is wrong.

---

## Chapter index

| # | Topic | One-line summary |
|---|-------|------------------|
| 01 | [System overview](01-system-overview.md) | Process boundaries, request paths, where state lives. |
| 02 | [Generation pipeline](02-generation-pipeline.md) | Prompt → outline → per-slide expansion → layout → render. |
| 03 | [Real-time collaboration](03-collaboration.md) | Yjs for text + pessimistic locks for everything else. |
| 04 | [Streaming protocol](04-streaming-protocol.md) | SSE event types, replay, reassembly. |
| 05 | [Theme and brand kit](05-theme-and-brand-kit.md) | Four-layer style precedence resolved at render time. |
| 06 | [Storage model](06-storage-model.md) | Entities, denormalization choices, soft delete. |
| 07 | [OAuth and token storage](07-oauth-and-token-vault.md) | Incremental consent, Fernet at rest, `drive.file` Picker. |
| 08 | [Scheduled decks](08-scheduled-decks.md) | In-process tick, source connectors, guardrails. |
| 09 | [Concurrency model](09-concurrency-model.md) | gevent, WebSocket-only, single-worker constraint. |
| 10 | [Failure modes](10-failure-modes.md) | What goes wrong, how the system recovers. |

---

## License

Documentation is licensed under [CC-BY-4.0](LICENSE). You are free to
share and adapt the material with attribution.
