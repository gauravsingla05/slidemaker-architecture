# Note 2 — Yjs (CRDT) over Operational Transform for text

**Status:** Accepted
**Context chapter:** [3. Collaboration](../03-collaboration.md)

## Context

Real-time collaborative text editing requires conflict resolution for
concurrent edits. The two dominant families are:

- **Operational Transform (OT).** Each operation is transformed
  against concurrent operations to preserve intent. Used by Google
  Docs.
- **CRDT (Conflict-free Replicated Data Types).** Operations carry
  enough metadata to merge deterministically without a central
  arbiter. Used by Figma (custom CRDT), Notion (some surfaces), and
  others.

Both can produce correct results. The choice is about engineering
cost.

## Decision

Use [Yjs](https://yjs.dev/), a mature open-source CRDT library, for
text content inside slide text elements. The server is a pure
broadcaster for Yjs binary updates; it does not transform them.

## Alternatives considered

| Option | Pros | Cons | Rejected because |
|--------|------|------|------------------|
| Operational Transform | Battle-tested (Google Docs) | Requires a central transform server; correctness proofs are hard; few good libraries | Engineering cost outweighs benefits at this scale |
| Custom CRDT | Tailored to slide structure | Building and maintaining a CRDT is a multi-quarter effort | Yjs already exists and is good |
| Last-write-wins | Trivial | Loses concurrent edits; visibly wrong | Bad UX |
| **Yjs (chosen)** | Off-the-shelf, mature, well-documented, fits the "server is dumb broadcaster" model | Binary wire format; debugging needs tooling | Best fit |

## Consequences

- The server does not need to understand text operation semantics.
  It only routes updates and provides initial state catch-up.
- Yjs documents are not the durable store. Slides live in MySQL; Yjs
  state is reconstructed when a room opens and lives in memory while
  active.
- "Binary updates + dumb broadcaster" makes it hard to inspect
  collaboration traffic without a Yjs-aware decoder.

## Why not Yjs for everything

Yjs (and CRDTs broadly) work best when concurrent operations have a
sensible merge. Text insertions concurrently typed in two places
merge cleanly. Two users dragging the same rectangle in different
directions do not — the only "merge" is to pick one. For non-text
elements the system uses pessimistic locks; see
[Note 3](ADR-003-pessimistic-element-locks.md).

## Revisit when

- A user-visible inconsistency is traced to Yjs behavior we cannot
  work around (none seen so far).
- Storage of Yjs-native documents becomes attractive (e.g., offline
  editing).
