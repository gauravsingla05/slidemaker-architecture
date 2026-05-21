# 1. System Overview

SlideMaker turns a natural-language prompt (and optionally an attached
document) into a slide deck that can be edited, shared, and
collaborated on in real time. This chapter sketches the moving parts at
a high level: what processes exist, how they talk to each other, and
where the synchronous and asynchronous paths run.

## 1.1 Big picture

There is a browser application, an application server, a database, and
a small set of external services.

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
       +---------+ +----+ +-------------------+
       | MySQL   | |Vault| | External:        |
       |         | |     | |   LLM providers  |
       |         | |     | |   Google OAuth   |
       |         | |     | |   Google Drive   |
       +---------+ +-----+ +-------------------+
```

A short tour of each box:

| Box | Role |
|-----|------|
| Edge proxy | Terminates TLS, caches the marketing site, routes WebSocket connections through to the application origin without buffering. |
| Browser | Renders the Next.js application. Holds the editor SPA, the rendered slides, and the Socket.IO client for collaboration. |
| Application server | A single process running the HTTP handlers, the Socket.IO server, and the in-process scheduler tick. All business logic lives here. |
| MySQL | The durable store: decks, slides, themes, users, schedules, runs. |
| Vault | The encrypted store for refresh tokens of external providers (see chapter 7). |
| External services | LLM providers for generation, Google for OAuth and per-file Drive access. |

The application server is intentionally one process. A microservice
split was considered and not pursued; the operational cost is greater
than the benefit at this size.

## 1.2 Three request paths

The system speaks three different "shapes" of request depending on the
workload.

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
```

### Sync HTTP

Used for: auth, profile, listing decks, fetching a deck, saving a deck,
template metadata. Short critical sections, idempotent or near-
idempotent.

### Server-Sent Events (SSE)

Used for: AI-driven generation, where the user sees slides appear one
at a time as the LLM produces them. SSE gives the user feedback after
about three seconds (when the first slide arrives) rather than after
the full thirty seconds of the run. Full discussion in
[chapter 4](04-streaming-protocol.md).

### WebSocket (Socket.IO)

Used for: real-time collaboration. Cursors, awareness, text deltas,
and pessimistic lock requests for non-text elements all flow through a
single Socket.IO session per editor tab. The server is configured to
accept only WebSocket, not the long-polling fallback; reasons in
[chapter 9](09-concurrency-model.md).

## 1.3 The five end-to-end flows

The product compresses into five flows. Most user value lives in flow
1 and flow 3.

```
   +-----------+     +----------+     +---------------+
   | Generate  | --> |   Edit   | --> | Collaborate   |
   +-----------+     +----------+     +-------+-------+
        ^                                     |
        |                                     v
        |                              +---------------+
        |                              |    Export     |
        |                              +---------------+
        |
        |  (scheduled, recurring)
   +----+------+
   | Schedule  |
   +-----------+
```

| Flow | Dominant path | Chapter |
|------|---------------|---------|
| Generate a deck | SSE | [2](02-generation-pipeline.md) |
| Edit slides | Sync HTTP | [5](05-theme-and-brand-kit.md) |
| Collaborate | WebSocket | [3](03-collaboration.md) |
| Export | Sync HTTP + OAuth | [7](07-oauth-and-token-vault.md) |
| Schedule | Scheduler tick + SSE (server-side) | [8](08-scheduled-decks.md) |

## 1.4 Where state lives

A common confusion in editor-style web apps is *where* the document
state of record lives. The answer here: the server is the source of
truth; the browser holds a projection that can diverge during editing
and is reconciled back.

```
            BROWSER                                   SERVER
   +---------------------------+           +------------------------+
   |  Zustand store            |           |  MySQL                 |
   |  (slides, theme,          |   save    |  (deck of record)      |
   |   brand kit)              | --------> |                        |
   |                           | <-------- |                        |
   |  Yjs document             |   load    |                        |
   |  (per-room text deltas,   |           |                        |
   |   in-memory only)         |           |                        |
   |                           |           |  Room state            |
   |  Quill editor             |   ws      |  (lock table,          |
   |  (rich text per element)  | <-------> |   awareness, sessions) |
   +---------------------------+           +------------------------+
```

- **Zustand store** holds the working copy of slides, theme, brand kit
  in the browser. Saved to MySQL on user save and via auto-save.
- **Yjs document** is per-room ephemeral state for text. *Not* durable.
  On a clean reload, slides re-hydrate from MySQL and a fresh Yjs doc
  starts.
- **Room state** lives in the application server's memory. Tracks who
  is in a room, who holds which element lock, and is throw-away on
  server restart.

## 1.5 What this system is not

To set expectations for later chapters:

- Not horizontally scaled. There is one application origin.
- Not agentic. AI calls are bounded, structured prompts that produce
  JSON. No autonomous loop, no tool use beyond document extraction.
- Not offline-first. Browser editor needs a network for save and
  collaboration.

Each chapter takes one slice of this picture and goes deeper.
