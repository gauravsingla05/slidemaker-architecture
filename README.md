# SlideMaker — Architecture & Engineering Notes

This repository documents the architecture of [SlideMaker.app](https://slidemaker.app),
an AI-powered presentation generator. It contains design notes, sequence
diagrams, decision records, and post-mortems written as a public engineering
journal.

> **Scope.** This repo is **documentation only.** It contains no source code,
> no secrets, no credentials, no infrastructure addresses, and no LLM prompt
> templates. It describes *what* the system does and *why* the choices were
> made, not *how to copy it*.

## Why publish this

Most production systems are described in two ways: glossy marketing pages
("AI-powered") and internal-only design docs. The middle ground — *honest,
diagrammed engineering write-ups of real, running systems* — is rare. This
repo is an attempt to fill that gap for one such system.

The audience is engineers who want to:

- Understand how a real, single-team production AI system is put together.
- See the trade-offs (hybrid CRDT + pessimistic locks; in-process scheduler;
  WebSocket-only transport) with the reasoning, not just the outcome.
- Read post-mortems of bugs that survived code review and shipped to users,
  and the diagnostic paths that found them.

## How to read this

The chapters are ordered roughly from the outside in: system → pipeline →
collaboration → protocol → state → storage → identity → automation →
runtime → failure. Each chapter is self-contained; you do not need to read
them in order.

### Chapters

| # | Document | Focus |
|---|----------|-------|
| 01 | [System overview](01-system-overview.md) | Process boundaries, request flow, sync/async paths |
| 02 | [Generation pipeline](02-generation-pipeline.md) | Prompt → outline → per-slide expansion → render JSON |
| 03 | [Real-time collaboration](03-collaboration.md) | Hybrid CRDT (Yjs) + pessimistic per-element locks |
| 04 | [Streaming protocol](04-streaming-protocol.md) | Server-Sent Events for incremental slide delivery |
| 05 | [Theme & brand kit](05-theme-and-brand-kit.md) | Layout registry, color/font precedence, override layers |
| 06 | [Storage model](06-storage-model.md) | Entity-relationship diagram (abstract), denormalization choices |
| 07 | [OAuth & token vault](07-oauth-and-token-vault.md) | Incremental consent, Fernet-encrypted refresh tokens, scope choice |
| 08 | [Scheduled decks](08-scheduled-decks.md) | In-process scheduler, data-source connectors, guardrails |
| 09 | [Concurrency model](09-concurrency-model.md) | Socket.IO async modes, worker affinity, transport choice |
| 10 | [Failure modes](10-failure-modes.md) | Recovery paths for the most interesting failures |

### Decisions

Short Architecture Decision Records (ADRs), one page each, describe a single
choice and the alternatives considered.

- [ADR-001 — WebSocket-only transport](decisions/ADR-001-websocket-only-transport.md)
- [ADR-002 — Yjs over Operational Transform](decisions/ADR-002-yjs-over-ot.md)
- [ADR-003 — Pessimistic locks for non-text elements](decisions/ADR-003-pessimistic-element-locks.md)
- [ADR-004 — drive.file + Picker over spreadsheets.readonly](decisions/ADR-004-drive-file-and-picker.md)
- [ADR-005 — In-process scheduler tick](decisions/ADR-005-in-process-scheduler.md)

### Case studies

Real bugs, written up after the fact, with diagrams of the failing flow.

- [The infinite echo loop in collaborative editing](case-studies/echo-loop.md)
- [Late login leaking anonymous identity into collab rooms](case-studies/late-login-anonymous-id.md)
- [The brand kit override that swallowed the title color](case-studies/brand-kit-color-override.md)

## Diagrams

Diagrams are written in [Mermaid](https://mermaid.js.org/) inside the markdown
files so they render natively on GitHub and stay versioned with the prose. A
few hand-drawn architecture posters live in [diagrams/](diagrams/) for the
larger system pictures.

## What this repo deliberately omits

- Source code or buildable artifacts.
- LLM prompt templates (the system's main IP).
- Internal hostnames, IPs, or environment variable values.
- Database column names, foreign keys, or schema migrations.
- Business and pricing details.
- SEO strategy.

When a section needs to reference an internal name, it uses an abstract
placeholder (e.g., "outline LLM", "websocket origin", "token vault") rather
than the real identifier.

## License

Documentation is licensed under [CC-BY-4.0](LICENSE). You are free to share
and adapt the material with attribution.

## Author

Written by the SlideMaker author. Corrections and discussions welcome via
GitHub Issues.
