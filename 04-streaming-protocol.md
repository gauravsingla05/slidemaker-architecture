# 4. Streaming Protocol

The generation pipeline (chapter 2) emits slides one at a time as they
finish. This chapter documents the wire protocol used to deliver them
to the browser: Server-Sent Events (SSE) over HTTP.

## 4.1 Why SSE, not WebSocket

Two natural choices for incremental delivery are WebSocket and SSE.
SSE is the right fit here:

1. **Generation is one-way.** Server emits; client consumes.
2. **Plain HTTP.** Works through every CDN and proxy without special
   configuration. The collaboration WebSocket is separate and only
   has to pay its configuration cost once.
3. **Reconnection is built into the browser.** SSE auto-reconnects
   and replays from the last received event id if the server
   cooperates.

## 4.2 Event types

Five event types, roughly in this order:

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

| Event | Payload (conceptual) |
|-------|----------------------|
| `start` | run identifier, target slide count |
| `outline_ready` | title and the ordered slide titles + intents |
| `slide_ready` | index + the rendered slide JSON |
| `done` | run id, final slide count |
| `error` | stage, message, whether fatal |

All payloads are JSON.

## 4.3 Wire format

Events are emitted in the standard SSE format. A blank line is the
event delimiter.

```
   event: outline_ready
   id: 2
   data: { "title": "Q1 Plan", "slides": [ ... ] }
   <blank line>
   event: slide_ready
   id: 3
   data: { "index": 0, "slide": { ... } }
   <blank line>
   event: slide_ready
   id: 4
   data: { "index": 2, "slide": { ... } }
   <blank line>
   event: done
   id: 12
   data: { "run_id": "...", "slide_count": 10 }
   <blank line>
```

The `id` field is monotonically increasing per stream. It is what the
browser echoes back in `Last-Event-Id` on reconnect.

## 4.4 Reconnection and replay

If the connection drops mid-stream, the browser automatically
reconnects to the same URL with `Last-Event-Id` set to the last id it
received. The server skips any events the client already has.

```
   Client                                          Server
     |                                               |
     | GET /generate                                 |
     | --------------------------------------------> |
     | <-- id:1  start ----------------------------- |
     | <-- id:2  outline_ready --------------------- |
     | <-- id:3  slide_ready (0) ------------------- |
     |                                               |
     |   connection drops                            |
     |                                               |
     | GET /generate (Last-Event-Id: 3)              |
     | --------------------------------------------> |
     | <-- id:4  slide_ready (1) ------------------- |
     | <-- id:5  slide_ready (2) ------------------- |
     |              ... etc.                         |
     | <-- id:12 done ------------------------------ |
```

Replay requires the server to hold the emitted event log for the
lifetime of the run. The log is bounded (a ten-slide deck produces
about thirteen events) and discarded shortly after `done`.

## 4.5 Client reassembly

The outline arrives first and tells the client how many slides to
expect and in what order. Subsequent `slide_ready` events arrive in
*completion* order, not outline order. The client maintains a sparse
array indexed by slide position and fills slots as they arrive.

```
       Waiting
          |
          |  outline_ready
          v
       Outlined  <-- slots: [_, _, _, _, _]
          |
          |  slide_ready (any index)
          v
       Filling
          |
          |  more slide_ready
          v
       Filling
          |
          +---- done (all filled)  --->  Done
          |
          +---- done (some missing) -->  PartialDone
          |
          +---- error (fatal)       -->  Error
```

"Completed" requires both:

1. `done` event received, and
2. Every slot in the outline filled or explicitly skipped.

A `done` with missing slots transitions to `PartialDone`, which the
UI renders with a "some slides failed; retry?" affordance.

## 4.6 Backpressure

LLM expansion is the bottleneck and naturally rate-limits the stream.
The backend never emits faster than the LLM finishes a slide; the
client never needs to buffer more than the size of one slide's JSON.
No explicit backpressure protocol is needed.

## 4.7 Timeouts

| Scope | Order of magnitude | On expiry |
|-------|--------------------|-----------|
| Per-LLM-call | ~30 s | retry once, then `error` |
| Per-slide | ~60 s | mark slot skipped, continue |
| Whole run | ~5 min | abort, emit `error fatal` |

The whole-run timeout exists to cover the case where the LLM provider
stalls on a single call. Without it, a stuck stream could hold a
request slot indefinitely.

## 4.8 Why not a single blocking response

Three reasons SSE beats one blocking response for this workload:

1. **Time-to-first-byte.** The outline can be on screen in three
   seconds. A blocking call delivers nothing until the last slide
   finishes (~thirty seconds).
2. **Visible progress.** The user can start reviewing slide 1 while
   slide 9 is still generating.
3. **Partial success.** A single error in slide 7 does not waste the
   completed work in slides 1–6.

## 4.9 Connections

- The pipeline that *produces* these events:
  [chapter 2](02-generation-pipeline.md).
- Recovery paths for generation errors:
  [chapter 10](10-failure-modes.md).
- Scheduled decks reuse the same pipeline but discard the SSE stream
  server-side; see [chapter 8](08-scheduled-decks.md).
