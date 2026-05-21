# 6. Storage Model

This chapter describes the persistence design at the *entity* level —
what is stored, how entities relate, and where the system trades
normalization for simplicity.

## 6.1 Entity-relationship overview

The system has a small set of long-lived entities. Most reads and
writes touch fewer than three of them.

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
       +-----+-----+      +---+---+
             |                |
             | creates        | produces
             v                v
          USER             DECK


       +-------------+        +---------------+
       | OAUTH_TOKEN |        |  EXPORT_JOB   |
       +------+------+        +-------+-------+
              |                       |
            user                    deck
```

A short narrative for each:

| Entity | What it stores |
|--------|----------------|
| `USER` | Identity, profile basics. One per person. |
| `BRAND_KIT` | A user's color / font / logo overrides. One per user. |
| `DECK` | A presentation. Many per user. |
| `SLIDE` | One slide of a deck. Many per deck; ordered by position. |
| `THEME` | A reusable visual template. Shared across users. |
| `SHARE` | A token-based access grant for a deck. |
| `SCHEDULE` | A recurring generation specification. |
| `RUN` | One execution of a schedule. |
| `OAUTH_TOKEN` | Encrypted refresh tokens for external providers. |
| `EXPORT_JOB` | An async export to PPTX / PDF / Google Slides. |

## 6.2 Why `SLIDE` is a separate table

A natural alternative is to store every deck as a single
`DECK.payload_json` blob containing all its slides. That model was
tried and abandoned for two reasons:

1. **Write contention.** A ten-slide deck with five collaborators
   produces bursts of small writes. Serializing those into a single
   blob means each save rewrites all ten slides; isolating them to a
   per-slide row means each save touches one row.
2. **Slide-level operations.** Reordering, duplicating, and bulk-
   applying a theme are all per-slide operations. Keeping the row
   boundary aligned with the operation boundary keeps queries simple.

```
   Single-blob model (rejected):

     +--------------------------------+
     | deck.payload                   |
     | = [ slide1, slide2, ..., sN ]  |
     +-----------------+--------------+
                       ^
                       |  edit slide 5
                       |  rewrites all N
                       |

   Per-row model (current):

     +--------+
     |  deck  |
     +---+----+
         |
         +--> +---------+
         |    | slide 1 |
         +--> +---------+
         |    | slide 2 |
         +--> +---------+
         |    | slide N |    <-- edit slide N
              +---------+        rewrites 1 row
```

The deck row stores only deck-level metadata. Heavy slide content
lives on the slides.

## 6.3 What goes in a slide's `payload_json`

Each slide row stores its rendered structure as JSON: layout id, the
element array (chapter 5), and any per-slide overrides. This is the
same shape the generation pipeline produces and the same shape the
editor loads.

JSON is used (instead of fully normalized element tables) because:

- Element content is hierarchical and varied (charts and tables
  differ structurally from text and images); flattening it into rows
  costs more in joins than it saves in indexability.
- Slides are *almost always* read or written as a whole.
- Slide-level queries (e.g., "find slides with charts") are rare and
  acceptable as full scans.

Trade-off: there is no SQL way to query "find decks that have a chart
with a certain data field." That has never been a needed query.

## 6.4 Indexing strategy

Indexes are kept to the minimum required for the actual query paths:

| Index | Purpose |
|-------|---------|
| `deck (owner_id, updated_at)` | List a user's decks, newest first. |
| `slide (deck_id, position)` | Load a deck's slides in order. |
| `share (token)` | Resolve a share link to a deck. |
| `schedule (user_id, next_run_at)` | Scheduler tick: "what is due?" |
| `run (schedule_id, started_at)` | Schedule history listing. |
| `oauth_token (user_id, provider)` | OAuth callback hand-off. |
| `export_job (deck_id, status)` | "Show active exports for this deck." |

Indexes on JSON paths are deliberately not used. When a JSON-path
query becomes hot, the right move is to promote that field to a real
column, not to bolt on a functional index.

## 6.5 Soft delete vs hard delete

Decks are soft-deleted: a `deleted_at` timestamp gates them out of
normal queries. Two reasons:

1. **Recovery.** Users delete decks by accident; soft delete makes
   recovery a query, not a backup restore.
2. **Audit.** Schedules and shares can reference decks; abruptly
   purging would create dangling references.

Slides are *not* soft-deleted. Reordering and deleting happens
constantly during editing, and a tombstone for every intermediate
state would dwarf the live data within hours. When a deck is
soft-deleted, its slides become unreachable through the deck; a
periodic job hard-deletes slides whose deck has been soft-deleted
beyond the recovery window.

```
       +--------+    user delete    +---------------+
       | Active | ----------------> | Soft Deleted  |
       +---+----+ <---------------- +-------+-------+
           ^         undelete                |
           |                                 | GC after window
           |                                 v
        create                       +---------------+
                                     | Hard Deleted  |
                                     +---------------+
```

## 6.6 What is *not* stored

A few things are deliberately ephemeral:

- **Yjs document state.** A live room reconstructs it from MySQL on
  first join, accumulates deltas in memory. On restart, rooms
  re-hydrate.
- **Element lock state.** Reset on restart; clients re-acquire as
  they refocus elements.
- **Awareness.** Per-session, broadcast only.
- **In-flight LLM responses.** Held in memory; only the resulting
  slides are persisted.

## 6.7 The token store: a special kind of storage

The `OAUTH_TOKEN` row stores ciphertext, not plaintext. The
encryption envelope and key management are described in
[chapter 7](07-oauth-and-token-vault.md).

## 6.8 Migrations

Two coexisting systems:

1. **Hand-authored SQL migrations** for older, lower-level schema
   changes.
2. **Alembic versions** for newer schema changes that benefit from
   auto-generated DDL and downgrades.

A pragmatic split, not a clean architecture. Removing the older
system requires rewriting all hand-authored migrations into Alembic,
and the operational benefit is small relative to the rewrite cost.

## 6.9 Connections

- The slide payload JSON shape is described in
  [chapter 5](05-theme-and-brand-kit.md).
- Token store encryption is in [chapter 7](07-oauth-and-token-vault.md).
- Scheduler-tick reads on the `SCHEDULE` table are in
  [chapter 8](08-scheduled-decks.md).
