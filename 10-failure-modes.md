# 10. Failure Modes

A system is only as honest as its failure handling. This chapter
enumerates the failure classes the system actually experiences, how
they manifest, and what the recovery path is.

## 10.1 Taxonomy

```
                          Failure
                             |
       +---------+--------+--+-----+---------+
       |         |        |        |         |
       v         v        v        v
   External   Internal  User    System
   dependency logic    input    resource
       |         |        |        |
       v         v        v        v
   LLM time-  Schema    Mal-     Socket
   out/error  invalid   formed   discon-
              races     input    nect
   OAuth     Logic     Unauth   Memory
   provider  bug       /expired pressure
   error
   Network
   partition
```

Each class has different visibility, different blast radius, and
different recovery semantics. The sections below pick the most
interesting from each branch.

## 10.2 LLM failures

The single most common external failure. Three subtypes:

```
                LLM call
                   |
                   v
              outcome?
                   |
       +-----------+-----------+
       |           |           |
   timeout    schema       hard
              invalid      error
       |           |           |
       v           v           v
   retry once  retry with   emit error event
   smaller     schema-      (per-slide or
   context     repair       fatal)
       |           |           |
   still         still         |
   times out  invalid?         |
       |           |           |
       +-----------+-----------+
                   |
                   v
              emit error event
```

- **Timeout.** The per-call timeout bounds wait. On expiry the call
  is retried once with a shorter input. A second timeout escalates
  to an error event for the affected slide.
- **Schema invalid.** A targeted retry includes the schema and the
  bad output in the next system message. Catches almost all
  formatting mistakes.
- **Hard error.** Provider 4xx/5xx or a malformed response that
  retries cannot fix. Pipeline emits a per-slide error and continues;
  if the outline call fails this way, the run aborts with
  `error fatal`.

## 10.3 Partial generation

A ten-slide run that finishes with eight slides is a partial success.
The client sees `done` with `slide_count: 8` and renders the deck
with two slots marked failed. The user is offered a per-slide
"retry" affordance that re-runs the expansion for just those slides
without re-running the outline.

```
       Generating
           |
       +---+---+
       |       |
     done    done
     (full) (short)
       |       |
       v       v
     AllOK   Partial
       |       |
       |     user clicks retry
       |       |
       |       v
       |    Retrying
       |       |
       |   +---+---+
       |   |       |
       | filled  still
       |   |    missing
       |   v       v
       | AllOK  Partial
       v
     done
```

The deck is saved either way. A partial deck with two missing slides
is strictly better than no deck at all.

## 10.4 OAuth provider failures

Three subtypes:

- **Refresh token revoked.** User changed their password or revoked
  access. Next exchange returns 400; row marked expired; feature
  re-consents on next use.
- **Scope downgrade.** User re-consented but unchecked a scope.
  Attached scopes are updated; features requiring the missing scope
  re-consent on next use.
- **Provider transient error.** 5xx from the provider. Retry with
  backoff up to a small bound, then surface as user error.

```
        Token exchange
              |
              v
           result?
              |
   +----------+-----------+--------------+
   |          |           |              |
   200      400        5xx transient   401
   |          |           |              |
   v          v           v              v
   Use      Mark        Backoff       Treat as
   access   expired,    retry         revoked
   token    prompt      (still
            re-consent  failing
                        -> surface)
```

## 10.5 Socket disconnects during collaboration

A collaboration session can drop for any of: network blip, laptop
sleep, mobile tab eviction, server restart. The recovery flow is the
same in all cases.

```
   Client                                       Server
     |                                            |
     |  established session                       |
     |                                            |
     |  X --- disconnect (any cause)              |
     |                                            |
     |  socket.io auto-reconnect                  |
     |                                            |
     |  connect ---------------------------->     |
     |                                            |
     |  join_room (same deck_id) ----------->     |
     |                                            |
     |  <--- room_users (current set) ------      |
     |                                            |
     |  <--- yjs_sync_full (full state) ----      |
     |                                            |
     |  session restored                          |
```

Locks held by the disconnected session are *not* immediately
released. They expire when the lock manager notices the missing
heartbeat. This is intentional: a brief disconnect should not let
another user swoop in mid-edit.

If the server itself restarts, all in-memory state is gone. Every
client reconnects, every room rebuilds, every lock is briefly free.
The user-visible artifact is half a second of "stale data" before
the `yjs_sync_full` arrives.

## 10.6 The echo loop

A class of bug particular to bidirectional sync: a remote update is
applied locally; the local change subscription detects "a slide
changed" and broadcasts it back; the server fans it out to every
other client (including the original sender); they apply it as a
remote update and re-broadcast it; ad infinitum.

```
   Alice                Server                 Bob
     |                    |                    |
     | local edit ------> | -- broadcast ----> | apply
     |                    |                    | re-broadcast
     |                    | <----------------- | as "local change"
     | <--- broadcast --- |                    |
     | apply              |                    |
     | re-broadcast ----> | --- broadcast ---> | ...
     |                    |                    |
     |                  forever                |
```

The guard (chapter 3) suppresses the re-broadcast for a short window
after a remote update is applied. The historical failure — React
StrictMode double-mount creating two independent guards that did not
see each other's flag — is in
[case-studies/echo-loop.md](case-studies/echo-loop.md).

## 10.7 Stale identity in collab rooms

If a user opens the editor while signed out and signs in moments
later, the collab session must rejoin with the new identity.
Otherwise the server has them as `anonymous` and refuses subsequent
owner-only operations.

The fix is to re-key the auto-join effect on the user identity, not
just on the deck id. A late login triggers a re-join with the real
user id. See [case-studies/late-login-anonymous-id.md](case-studies/late-login-anonymous-id.md).

## 10.8 Database write failures during a run

The save stage at the end of a generation run is the last place
where the run can fail. If the database write fails, the slides
exist in memory and have been emitted to the client, but the deck
row was not (fully) created.

Recovery is to retry with the same payload. If retry also fails, the
user sees an error and the deck content is offered as a download
(so the work is not lost while the database issue is investigated).

```
   Slides in memory
         |
         v
       Save
         |
       fail?
         |
   +-----+-----+
   |           |
  OK         fail
   |           |
   v           v
  Done    Retry once
           |
        +--+--+
        |     |
       OK   fail
        |     |
        v     v
      Done  Offer download
             + log incident
```

## 10.9 Recovery hierarchy

A one-line summary:

> Prefer **automatic per-step retry** for transient errors,
> **graceful degradation** for partial failure,
> **user-visible escalation** with a clear next action for
> everything else.

What the system does *not* do:

- Silent retries that mask root causes.
- Best-effort error swallowing.
- Surfacing raw exception text to users.

## 10.10 Connections

- Each chapter's "what happens on failure" sections feed into this
  one.
- The post-mortems in `case-studies/` are concrete instances of
  failures the system has lived through.
