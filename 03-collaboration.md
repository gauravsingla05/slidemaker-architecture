# 3. Real-Time Collaboration

Multiple users can edit the same deck simultaneously, with live
cursors, presence, and per-element protection against conflicting
edits. The collaboration layer is a *hybrid* of two patterns: a CRDT
for text and pessimistic locks for everything else. This chapter
explains why and how.

## 3.1 The two-mode problem

Editing a slide deck combines two different editing surfaces:

1. **Text inside a text element.** Many small mergeable edits, often
   typed into different parts of the same paragraph at once.
2. **Everything that isn't text.** Moving a shape, resizing an image,
   replacing a chart's data, changing a table's structure.

The right tool for (1) is a CRDT: it merges concurrent insertions
cleanly without needing a central arbiter. The right tool for (2) is
a lock: two people dragging the same rectangle in opposite directions
has no sensible "merged" result.

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
        | delta    |   |  manager    |
        +-----+----+   +------+------+
              |               |
         broadcast       acquire ->
         via awareness   mutate ->
         channel         release
              |               |
              +-------+-------+
                      |
                      v
                room broadcast
                to other clients
```

## 3.2 Room model

A room is one editing session for one deck. Joining a room opens a
Socket.IO session, sends a `join_room` event with the deck id and a
user identity, and receives back the room's current state.

```
   Client                    Server                    Room state
     |                          |                           |
     | connect (WebSocket)      |                           |
     |------------------------> |                           |
     |     "connected"          |                           |
     | <----------------------- |                           |
     |                          |                           |
     | join_room(deck_id, user) |                           |
     |------------------------> |                           |
     |                          | register session ----->   |
     |                          | <----- existing users     |
     |     room_users           |                           |
     | <----------------------- |                           |
     |                          | notify others             |
     |                          | --- user_joined --->      |
     |                          |                           |
     |     yjs_sync_full        |                           |
     | <----------------------- |                           |
```

Four channels of traffic exist after the join:

| Channel | Direction | Purpose |
|---------|-----------|---------|
| Yjs sync | bidirectional | text deltas + initial state catch-up |
| Awareness | bidirectional | cursor position, color, focused element |
| Slide update | bidirectional | non-text mutations |
| Element locks | request/response + broadcast | lock acquire / release / heartbeat |

## 3.3 Yjs sync

Yjs is a CRDT library. Each client and the server hold a copy of the
same Yjs document. Local edits produce binary deltas that are
broadcast to every other client in the room; remote updates are
applied to the local doc. The server is a pure broadcaster; it does
not transform Yjs updates.

```
   Alice                    Server                       Bob
     |                        |                          |
     | types "Hello"          |                          |
     | -- yjs_update -------> |                          |
     |                        | -- yjs_update ---------> |
     |                        |                          | applies
     |                        |                          | "Hello"
     |                        |                          |
     | types "!" at end       |                          | types " world"
     | -- yjs_update -------> |                          | -- yjs_update -->
     |                        | -- yjs_update ---------> |
     |                        | <------ yjs_update ----- |
     | applies " world"       |                          | applies "!"
     |                        |                          |
     |          both converge to "Hello world!"         |
```

The CRDT does the heavy lifting; the server only routes bytes.

## 3.4 Awareness (cursors and presence)

Awareness is everything that *isn't* the document: who is in the room,
their color, where the cursor is, which element is focused. Awareness
is broadcast and never persisted; on a server restart it is lost and
rebuilt as clients send their next update.

```
   Alice               Server               Bob       Carol
     |                   |                   |          |
     | move cursor       |                   |          |
     | -- awareness ---> |                   |          |
     |                   | --- fanout -----> |          |
     |                   | --- fanout ----------------> |
     |                                       |          |
                                  Render Alice's cursor + label
```

Each user gets a deterministic color from a hash of their user id, so
they look the same across sessions and across other users' screens.

## 3.5 Pessimistic locks for non-text edits

For non-text mutations a pessimistic lock model is used:

1. Client requests a lock when the user focuses an element.
2. Server grants or denies based on the current owner.
3. While the lock is held, the client may mutate freely; other clients
   see a "locked by Alice" overlay on that element.
4. The client heartbeats every few seconds; a missed heartbeat expires
   the lock.
5. On blur or unmount, the client releases the lock.

```
   Client                Lock Manager                Room
     |                        |                       |
     | acquire(elem_id)       |                       |
     | ---------------------> |                       |
     |                        | check owner table     |
     |                        |                       |
     |  if free:              |                       |
     |     granted            |                       |
     | <--------------------- |  ---- element_locked  |
     |                        |       ------------->  |
     |                        |                       |
     |  loop while held:      |                       |
     |     heartbeat -------> |                       |
     |                        |                       |
     |     release ---------> |  ---- element_unlock  |
     |                        |       ------------->  |
     |                                                |
     |  if held by other:                             |
     |     denied (owner)                             |
     | <--------------------- |                       |
```

### Why pessimistic, not optimistic

An optimistic model — let everyone mutate, reconcile after — could
work for non-text edits with operational transforms. In practice, the
UX of "your move was undone because someone else moved it first" is
worse than "you can't move this right now because someone else is."

## 3.6 The echo-loop guard

Yjs updates and slide_update events flow through the same socket. When
a local edit is broadcast, the same change must not bounce back from
the server and be re-applied locally as if it came from a remote
client.

The guard is a pair of module-scoped flags:

- `isApplyingRemote` — set while a remote update is being applied; the
  broadcast subscription checks this and skips re-broadcasting.
- `lastRemoteUpdate` — a recent `{ slideIndex, timestamp }`; if a
  local change matches the slide of a remote update received within
  200 ms, the broadcast is suppressed as an echo.

```
        Idle
         |  remote update arrives
         v
   ApplyingRemote ----- 150 ms timer -----+
         |                                |
         |  any "local" change            v
         v  suppressed                  Idle
       Idle                             |
         |  user edit                   |
         v                              |
    Broadcasting --- emit to server --> Idle
```

The 150 ms / 200 ms windows are just long enough to cover the round
trip and the store-subscription tick, and just short enough to not
suppress a genuine quick local edit after a remote one.

The history of this guard, including the React StrictMode bug that
made the original implementation insufficient, is in
[case-studies/echo-loop.md](case-studies/echo-loop.md).

## 3.7 Broadcast path vs save path

Two write paths in collaboration:

```
                    local change
                         |
              +----------+----------+
              |                     |
              v                     v
        broadcast              save (debounced)
        (other clients          (MySQL)
         in real time)
              |                     ^
              |                     |
              +---- debounced ------+
                    batch flush
```

The broadcast path runs immediately; collaborators see the change in
under fifty milliseconds. The save path is debounced and batched;
the database catches up in a few seconds. If the user closes their
tab after broadcasting but before saving, collaborators still have
the change, and it reaches the database via the next save from any of
them.

## 3.8 Limits

- **Single application origin.** Lock state lives in process memory;
  a multi-origin setup would need a shared lock store.
- **No offline editing.** A disconnected client cannot edit.
- **No cross-deck awareness.** Awareness is per-room, by design.

## 3.9 Connections

- Transport choice and worker affinity are in
  [chapter 9](09-concurrency-model.md).
- Bug stories: [echo-loop](case-studies/echo-loop.md),
  [late-login-anonymous-id](case-studies/late-login-anonymous-id.md).
- Why Yjs over OT:
  [ADR-002](decisions/ADR-002-yjs-over-ot.md).
- Why pessimistic locks for non-text:
  [ADR-003](decisions/ADR-003-pessimistic-element-locks.md).
