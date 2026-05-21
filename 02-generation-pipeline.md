# 2. Generation Pipeline

The user types a topic, optionally attaches a document, and roughly
thirty seconds later has a ten-slide deck. This chapter describes what
happens in those thirty seconds.

## 2.1 The five stages

A generation run flows through five distinct stages. Each stage's
output is the next stage's input.

```
  +--------+    +---------+    +-----------+    +-----------+    +-----------+
  | Ingest | -> | Outline | -> | Per-slide | -> |  Layout   | -> |  Render   |
  |        |    |  (LLM)  |    |  Expand   |    |  Select   |    |   JSON    |
  |        |    |         |    |  (LLM xN) |    |           |    |           |
  +--------+    +---------+    +-----------+    +-----------+    +-----------+
   ~ 0-2 s        ~ 3 s         ~ 2 s each       < 100 ms         < 200 ms
                                (parallel)
```

| Stage | Kind | Latency | If it fails |
|-------|------|---------|-------------|
| Ingest | deterministic | tens of ms (no doc) to seconds (with doc) | retry once, then surface |
| Outline | LLM call | a few seconds | retry once with smaller context |
| Per-slide expand | parallel LLM calls | a few seconds each | per-slide retry, skip on failure |
| Layout select | deterministic | sub-100 ms | fall back to text-only |
| Render JSON | deterministic | sub-200 ms | skip slide, continue |

### Ingest

If a document is attached (PDF, DOCX, URL, text), the ingest stage
extracts text and compresses it to a budget. Compression uses a
position-aware chunking method (described in a separate research
paper). Output is a structured input bundle: the prompt, the target
slide count, an audience hint, and an optional context block.

### Outline

A single LLM call produces the structural skeleton: a title, a list
of slide titles, and a one-sentence "intent" for each slide. Output
is strict JSON; the response is parsed and schema-validated.

```
   prompt + context
         |
         v
   +-----------+
   |  Outline  |   strict JSON output
   |    LLM    |   { title, slides: [
   +-----------+       { title, intent }, ...
         |             ]}
         v
   parse + validate
         |
         v
   emit "outline_ready" SSE event
```

If parsing fails, the request is retried once with the schema injected
into the system message; a second failure aborts the run.

The outline is sent to the client as the *first* SSE event so the user
sees the shape of the deck before any individual slide finishes
generating. This single event is responsible for most of the system's
perceived responsiveness.

### Per-slide expand

For each entry in the outline, an LLM call expands the one-sentence
intent into the slide's body content: bullets, paragraphs, optional
chart data, optional table rows. Calls run concurrently with a bounded
limit.

```
                +-----------+
                |  Outline  |
                +-----+-----+
                      |
                      | fan out (parallel)
       +--------+-----+-----+--------+
       |        |           |        |
       v        v           v        v
   +------+ +------+    +------+ +------+
   |Slide | |Slide | .. |Slide | |Slide |
   |  1   | |  2   |    |  N-1 | |  N   |
   +--+---+ +--+---+    +--+---+ +--+---+
      |       |            |        |
      v       v            v        v
       emit each as "slide_ready"
       in completion order
       (not outline order)
```

Slides are emitted in the order their responses arrive. The client
keeps the outline as the ordered skeleton and fills each slot as the
matching `slide_ready` event arrives.

### Layout select

The body content for each slide is matched to a layout from the active
theme's registry. The matcher is a deterministic function: given
`(slide kind, bullet count, has chart, has table, has image intent)`
it returns the layout id with the best fit score. No LLM in this
stage — structured decisions with a small search space are faster and
more reliable in code.

```
   body
    |
    v
   +-------------------+
   | extract features  |
   +---------+---------+
             |
             v
   +-------------------+      for each layout
   | score against     |  --> score(layout) =
   | each registered   |       weighted match of
   | layout            |       features to slots
   +---------+---------+
             |
             v
        pick highest
             |
             v
       layout id + slot
       assignments
```

### Render JSON

The final stage emits the slide as the JSON shape the client renders.
This includes element placements, text as Quill Deltas, chart data,
table data, colors resolved against the active brand kit (chapter 5),
and references to logos and images. Output is plain data: it can be
saved, broadcast, or repainted.

## 2.2 Why split outline from expansion

A natural alternative is "one big LLM call for the whole deck." That
was tried and discarded:

1. **Latency feel.** One blocking call shows zero feedback for thirty
   seconds. The split version gives feedback at three seconds.
2. **Token efficiency.** Per-slide calls can run in parallel; one call
   cannot.
3. **Error isolation.** If slide 7 fails, only slide 7 retries.

The cost is that the outline must commit before the slides do: the
"intent" the outline sets for slide 7 cannot be revised based on what
slide 6 ended up containing. In practice this rarely matters because
the outline is high-level enough to leave room for the expander.

## 2.3 Shape of an LLM call

All LLM calls follow the same envelope:

```
   +-----------------+   +-----------------+   +-----------------+
   | system prompt   |   |  user prompt    |   |   JSON schema   |
   +--------+--------+   +--------+--------+   +--------+--------+
            \                    |                     /
             \                   |                    /
              +---->  provider call  <----+
                          |
                          v
                 +-----------------+
                 | parse + validate|
                 +--------+--------+
                          |
                +---------+---------+
                |                   |
              valid               invalid
                |                   |
                v                   v
              output         retry once with
                             schema-repair message
                                    |
                                  still invalid
                                    |
                                    v
                                 abort call
```

Every LLM response is parsed against a strict schema before it is
allowed into the rest of the pipeline. The single retry catches almost
all single-shot formatting mistakes.

## 2.4 Where the time goes

A rough timeline for a ten-slide run. LLM latency dominates;
everything else is sub-second.

```
   second:   0    5    10    15    20    25    30
             |    |     |     |     |     |     |
   Ingest    XX
   Outline       XXXX
   Slides            XXXXXXXXXXXXXXXX
                     ^ first slides emit here
   Save                                      XX
   Repaint                                    X
```

## 2.5 Determinism around the LLM

LLMs are non-deterministic by construction, but the pipeline around
them is fully deterministic. Given the same outline output and the
same expansion outputs, the layout selection, render JSON, and final
slide structure are byte-identical. This matters for:

- **Reproducibility.** Replaying stored LLM responses through the
  deterministic layers produces the same output, so a bug can be
  isolated to the LLM step or to one of the post-LLM steps.
- **Caching.** Identical outline + expansion outputs are cached by
  hash, so repeated runs with the same inputs skip the post-LLM
  stages.

## 2.6 Connections to other chapters

- The streaming format used by `outline_ready` and `slide_ready` is
  defined in [chapter 4](04-streaming-protocol.md).
- The render JSON references theme layout slots and brand kit colors,
  defined in [chapter 5](05-theme-and-brand-kit.md).
- Scheduled decks run this same pipeline on a cron tick; see
  [chapter 8](08-scheduled-decks.md).
- Failure recovery for each stage is in
  [chapter 10](10-failure-modes.md).
