# Diagrams

Most diagrams in this repository are embedded directly in the chapter
markdown files using [Mermaid](https://mermaid.js.org/) so they render
natively on GitHub and stay versioned alongside the prose.

This folder is reserved for larger architecture posters or hand-drawn
figures that do not fit Mermaid's syntax (e.g., exported from Excalidraw
as SVG). At the time of writing, all figures live inline in the chapter
markdown.

## Why Mermaid for most things

- **GitHub renders it natively.** No external service, no broken
  preview, no dead images after a year.
- **Source-controlled.** A pull request that updates prose can also
  update its diagram in the same commit.
- **Diff-friendly.** Changes show up in code review as text diffs.
- **Cheap to author.** A sequence diagram is six lines.

## When Mermaid is not enough

- Free-form architecture posters with annotations and call-outs.
- Diagrams that need precise typographic control.

For those, the recommendation is Excalidraw, exporting to SVG, and
checking the SVG in alongside its `.excalidraw` source so future edits
are possible.
