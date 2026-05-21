# Decision Notes

Short, one-page notes capturing significant design decisions and the
reasoning behind them. Each follows the same shape: context,
decision, alternatives considered, consequences, and when to
revisit.

| Note | Topic | Linked chapter |
|------|-------|----------------|
| [1](ADR-001-websocket-only-transport.md) | WebSocket-only transport | [9. Concurrency](../09-concurrency-model.md) |
| [2](ADR-002-yjs-over-ot.md) | Yjs (CRDT) over Operational Transform | [3. Collaboration](../03-collaboration.md) |
| [3](ADR-003-pessimistic-element-locks.md) | Pessimistic locks for non-text elements | [3. Collaboration](../03-collaboration.md) |
| [4](ADR-004-drive-file-and-picker.md) | `drive.file` + Picker over `spreadsheets.readonly` | [7. OAuth & token storage](../07-oauth-and-token-vault.md) |
| [5](ADR-005-in-process-scheduler.md) | In-process scheduler tick | [8. Scheduled decks](../08-scheduled-decks.md) |

## When to write a note

- The decision is non-obvious.
- A future maintainer might reverse it without understanding why.
- The alternative considered is one someone would naturally propose.

If the answer to "why is it this way" is "because it's the only way
it could be," no note is needed.
