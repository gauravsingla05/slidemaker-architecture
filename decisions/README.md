# Architecture Decision Records (ADRs)

Short, one-page documents capturing significant architectural decisions
and the reasoning behind them. Each ADR follows the same shape: context,
decision, alternatives considered, consequences, and when to revisit.

| ADR | Topic | Linked chapter |
|-----|-------|----------------|
| [001](ADR-001-websocket-only-transport.md) | WebSocket-only transport | [9. Concurrency](../09-concurrency-model.md) |
| [002](ADR-002-yjs-over-ot.md) | Yjs (CRDT) over Operational Transform | [3. Collaboration](../03-collaboration.md) |
| [003](ADR-003-pessimistic-element-locks.md) | Pessimistic locks for non-text elements | [3. Collaboration](../03-collaboration.md) |
| [004](ADR-004-drive-file-and-picker.md) | `drive.file` + Picker over `spreadsheets.readonly` | [7. OAuth & token vault](../07-oauth-and-token-vault.md) |
| [005](ADR-005-in-process-scheduler.md) | In-process scheduler tick | [8. Scheduled decks](../08-scheduled-decks.md) |

## When to write an ADR

- The decision is non-obvious.
- A future maintainer might reverse it without understanding why.
- The alternative considered is one someone would naturally propose.

If the answer to "why is it this way" is "because it's the only way it
could be," no ADR is needed.
