# 9. Concurrency Model

The application server hosts three concurrent workloads: synchronous
HTTP requests, long-lived Socket.IO sessions, and short-lived SSE
streams. This chapter explains how they share one process and why a
few unusual choices (WebSocket-only transport, single worker for
collaboration) hold.

## 9.1 The three workloads

```
   +---------------------------------------------------+
   |             single application process            |
   |                                                   |
   |   +---------+   +-----------+   +---------+       |
   |   |  HTTP   |   | Socket.IO |   |   SSE   |       |
   |   | handler |   |  sessions |   | streams |       |
   |   |  pool   |   |  (idle    |   | (active |       |
   |   | (fast)  |   |   thou-   |   |  during |       |
   |   |         |   |   sands)  |   | gen)    |       |
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

Resource profiles:

| Workload | CPU | Mem | Wall-clock | Peak concurrent |
|----------|-----|-----|------------|-----------------|
| HTTP | low | low | 50–200 ms | hundreds |
| Socket.IO session | very low | tiny | minutes–hours | thousands idle, dozens active |
| SSE stream | low | low | seconds–~1 min | dozens at peak |
| Scheduler tick | low | low | seconds | a handful concurrent runs |

The dominant scaling constraint is *socket count*, not CPU.

## 9.2 Async server framework

The server uses Flask-SocketIO. At boot it auto-detects which async
runtime is available, preferring the most efficient.

```
   Process boot
        |
        v
   Try gevent
        |
    +---+----+
    |        |
   yes      no
    |        |
    v        v
   Use      Try eventlet
   gevent       |
            +---+----+
            |        |
           yes      no
            |        |
            v        v
           Use      Fall back to
           eventlet threading
```

In production the runtime is gevent. Sockets are cheap "green
threads," which is what makes thousands of idle sessions viable in
one process.

## 9.3 The transport choice: WebSocket only

Socket.IO supports two transports — long polling and WebSocket. The
server is configured to allow **only WebSocket**. The reason is a
subtle multi-worker concern made visible by a real-world incident.

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


   WebSocket only (GOOD):

         Client
            |
       single WS
            |
            v
        Worker A
            |
          session
```

When polling is allowed, the GET and POST halves of a poll cycle can
land on different workers. Each worker maintains its own session map,
so the second has no record of the session and returns "Invalid
session." Workarounds (sticky sessions, shared session store) exist;
the simplest is to forbid polling.

Recorded in [ADR-001](decisions/ADR-001-websocket-only-transport.md).

## 9.4 Why a single worker for collaboration

Locks (chapter 3) live in process memory. Two workers would each
have their own lock table and would happily grant the same lock to
two clients. Solving this requires either:

- **Sticky sessions** — route every client of a given room to the
  same worker. Every layer in front of the application has to
  cooperate.
- **Shared lock store** — Redis or similar, accessed by every
  worker. Adds an operational tier and a network hop on every lock
  operation.

Neither is worth the complexity until traffic forces it. A single
worker with gevent green threads handles thousands of idle sessions
and dozens of active rooms comfortably.

```
   Current:                                Future (if needed):

   +------------------+                    +---------+
   | single worker    |                    | Worker A|---+
   |                  |                    +---------+   |
   |  +------------+  |                    +---------+   |
   |  | in-process |  |                    | Worker B|---+--> Redis
   |  | lock mgr   |  |                    +---------+   |   (shared locks)
   |  +------------+  |                    +---------+   |
   +------------------+                    | Worker C|---+
                                           +---------+
```

The migration path is well understood; the cost is real but bounded.

## 9.5 SSE streams and the request slot

An SSE stream holds the request slot for as long as the stream runs —
twenty to sixty seconds typical. With gevent this is essentially free
(the slot is a green thread, not an OS thread), but two limits still
apply:

1. **Browser per-origin connection limit.** A user with two tabs
   running generation in the same origin will hit the browser's
   per-origin cap (six in most browsers) faster than they would with
   shorter requests.
2. **Proxy buffer behavior.** Some proxies buffer responses before
   flushing to the client. Buffering breaks SSE: events never reach
   the browser until the server closes the stream. The edge proxy is
   configured to *not* buffer SSE responses; this is the single most
   fragile bit of the streaming setup and is checked in deploy smoke
   tests.

## 9.6 Scheduler-tick concurrency

The scheduler runs as its own scheduled job inside the same process.
On each fire it scans due schedules, *spawns* runs as background
tasks, and returns quickly so the next tick is not delayed.

```
   Tick                 +----------------+
   (every minute) ----> | Find due       |
                        | schedules      |
                        +-------+--------+
                                |
                                v
                        +----------------+
                        | Spawn N runs   |
                        | as background  |
                        | tasks          |
                        +-------+--------+
                                |
                                v
                              return

                      Run 1 -- async ----+
                      Run 2 -- async ----+----> share LLM-call concurrency
                       ...                |     limit so no thundering herd
                      Run N -- async ----+
```

## 9.7 In-process shared state

The process holds several pieces of in-memory state shared across
green threads:

| State | Owner | Lifetime |
|-------|-------|----------|
| Socket.IO session map | framework | process |
| Room registry (chapter 3) | collaboration module | process |
| Lock table | lock manager | process |
| Generation cache (input hash → cached output) | pipeline | process |
| Awareness state | collaboration module | per session |

Access is single-threaded by virtue of the green-thread model: no
preemptive thread can yield control inside a non-yielding section.
The discipline is "do not perform network I/O while holding an
invariant."

## 9.8 Where the model would break

Worth being honest about the failure modes:

- **A second worker, naively added.** Two workers with in-memory
  locks hand out duplicate locks. The system would not crash; it
  would produce silent collaboration bugs.
- **Polling re-enabled.** "Invalid session" errors for half of all
  socket events.
- **Edge proxy buffering enabled by default.** Streaming generation
  appears to hang for thirty seconds and deliver all slides at once.
- **A blocking call sneaking into a green thread.** A
  `time.sleep()` or a CPU-heavy hash on the request path would
  freeze every other green thread in the worker.

Each is preventable and visible to operators, but none are caught by
type checking — they require operational care.

## 9.9 Connections

- The collaboration model that depends on in-process locks:
  [chapter 3](03-collaboration.md).
- The SSE protocol carried over the request slot:
  [chapter 4](04-streaming-protocol.md).
- The scheduler tick spawning concurrent runs:
  [chapter 8](08-scheduled-decks.md).
- Transport choice rationale:
  [ADR-001](decisions/ADR-001-websocket-only-transport.md).
