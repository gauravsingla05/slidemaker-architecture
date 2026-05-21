# SlideMaker — Architecture Notes

A walk-through of how [SlideMaker.app](https://slidemaker.app) is put
together. SlideMaker is a personal learning project that turns a prompt
into a slide deck using AI; this repository explains the design
decisions, sketches the moving parts with simple diagrams, and writes
up a few of the more interesting bugs that came up along the way.

The intent is educational. If you are learning how AI-assisted apps,
real-time collaboration, or streaming pipelines work, the chapters
below describe the patterns at a level that is meant to be readable
without prior context.

## How to read this

The chapters are ordered roughly outside-in: system → pipeline →
collaboration → protocol → state → storage → identity → automation →
runtime → failure. Each chapter is self-contained.

### Chapters

| # | Topic |
|---|-------|
| 01 | [System overview](01-system-overview.md) |
| 02 | [Generation pipeline](02-generation-pipeline.md) |
| 03 | [Real-time collaboration](03-collaboration.md) |
| 04 | [Streaming protocol](04-streaming-protocol.md) |
| 05 | [Theme and brand kit](05-theme-and-brand-kit.md) |
| 06 | [Storage model](06-storage-model.md) |
| 07 | [OAuth and token storage](07-oauth-and-token-vault.md) |
| 08 | [Scheduled decks](08-scheduled-decks.md) |
| 09 | [Concurrency model](09-concurrency-model.md) |
| 10 | [Failure modes](10-failure-modes.md) |

### Decision notes

Short notes capturing a single design choice with the alternatives
that were considered.

- [Note 1 — WebSocket-only transport](decisions/ADR-001-websocket-only-transport.md)
- [Note 2 — Yjs over Operational Transform](decisions/ADR-002-yjs-over-ot.md)
- [Note 3 — Pessimistic locks for non-text elements](decisions/ADR-003-pessimistic-element-locks.md)
- [Note 4 — `drive.file` + Picker for Google Sheets](decisions/ADR-004-drive-file-and-picker.md)
- [Note 5 — In-process scheduler tick](decisions/ADR-005-in-process-scheduler.md)

### Case studies

Three bugs that were instructive once understood.

- [The infinite echo loop in collaborative editing](case-studies/echo-loop.md)
- [Late login and anonymous identity in collab rooms](case-studies/late-login-anonymous-id.md)
- [The brand kit color that ate every title](case-studies/brand-kit-color-override.md)

## Diagrams

All diagrams in this repository are drawn in plain text using ASCII
characters so they render the same way in every viewer (GitHub, plain
text editors, terminals) and stay readable without any external
rendering tool.

## License

Documentation is licensed under [CC-BY-4.0](LICENSE). You are free to
share and adapt the material with attribution.
