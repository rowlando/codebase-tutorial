---
name: codebase-tutorial
description: >-
  Turn a codebase into a beginner-friendly, multi-chapter Markdown tutorial with
  an architecture diagram. Use this when the user says "explain this codebase",
  "generate a tutorial for this repo", "onboard me to this codebase", "walk me
  through how this project works", or asks for a guided/teaching walkthrough of a
  repository. Reads the source read-only, writes only into an output directory,
  makes no network calls.
---

# Codebase Tutorial

Generate a guided, beginner-friendly tutorial for a codebase. You read the code
and **are** the model — there is no separate LLM API call and no network access.
You replicate the "PocketFlow Codebase Knowledge" pipeline as six sequential
steps, persisting intermediate state to scratch files so large runs are
debuggable and resumable.

## Arguments

Parse these from the user's request; apply defaults when unstated.

| Argument | Default | Meaning |
|----------|---------|---------|
| `target` | current directory | Root path of the codebase to explain. **Read-only.** |
| `output_dir` | `./codebase-tutorial` | Where all output and scratch files are written. **Only writable location.** |
| `max_abstractions` | `10` | Upper bound on core abstractions (valid range 5–10). |
| `include` | language defaults (see PIPELINE) | Glob(s) of files to consider. |
| `exclude` | vendored/build dirs (see PIPELINE) | Glob(s) to skip. |
| `language` | `English` | Natural language for the generated tutorial prose. |
| `focus` | _none (balanced)_ | Optional emphasis: a topic, concern, or path to centre the tutorial on (e.g. "the pricing domain", "the auth flow", "src/journeys"). Skews curation, chapter order, and depth toward it; demotes everything else to background. |

Confirm the resolved `target` and `output_dir` with the user before writing if
either is ambiguous.

## What to emphasise

By default the tutorial gives balanced weight to the codebase's main parts, which
suits most repos. Two adjustments make it consistently more useful:

- **Lead with what makes this codebase distinct.** Favour the abstractions that
  capture the project's *purpose* — its domain model, core algorithm, the thing it
  uniquely does — over generic scaffolding present in almost every app of its kind
  (framework bootstrap, auth, sessions, config plumbing, logging, error helpers).
  Cover scaffolding too, but **group and condense it** so it never outnumbers or
  crowds out the domain. A reader should finish understanding *what this system is
  for*, not just *how a typical app in this stack is wired*.
- **Honour an explicit `focus`.** When the user names a focus (an area, a path, or
  a concern), centre the tutorial on it: curate the manifest toward its files,
  order its chapters early, give it more and deeper chapters, and demote the rest
  to brief background. Don't drop the connective tissue a newcomer needs to place
  the focus in context — just spend the page budget where they asked.

These are emphasis defaults, not hard rules: if a codebase's scaffolding genuinely
*is* the interesting part (e.g. a framework or a starter template), or the user
asks to onboard onto the infrastructure, weight it accordingly.

## Hard rules

- **Read-only on the source.** Never modify, move, or delete anything under `target`.
- **Write only inside `output_dir`.** Create it if missing. Nothing is written elsewhere.
- **No network.** Never fetch, push, or transmit code or analysis anywhere.
- **Reference files by index.** Build a manifest mapping `path → integer index`
  once, then refer to files by that index in every later step and scratch file.
- **Resumable.** Before each step, check whether its scratch file already exists
  in `output_dir`; if so and it validates, reuse it instead of redoing the work.

## Pipeline (six steps)

Read `references/PIPELINE.md` for the detailed procedure, default globs, and the
large-repo strategy. Summary:

1. **Fetch** — enumerate files under `target` honouring include/exclude globs and
   the max file size; skip vendored/build dirs. Read each kept file. Write
   `manifest.json` (`index → {path, size}`).
2. **Identify abstractions** — derive 5–10 core abstractions. Each has a
   beginner-friendly `name`, an analogy-driven `description`, and `file_indices`.
   Write `abstractions.json`.
3. **Analyse relationships** — write a one-paragraph project `summary` plus
   `relationships` as `{from_index, to_index, label}` (indices into the
   abstractions list). Write `relationships.json`.
4. **Order chapters** — order abstraction indices for teaching: foundational and
   entry-point concepts first, then dependents. Write `order.json`.
5. **Write chapters (map-reduce)** — for each abstraction in order, write one
   detailed Markdown chapter grounded in real code (file references + short
   snippets). Keep a running summary of chapters so far and feed it into each new
   chapter so the sequence reads continuously. Motivate *why it exists* before
   *how it works*; use analogies. Write `NN_name.md` per chapter.
6. **Combine** — write `index.md`: the summary, a Mermaid `flowchart TD` built
   from the relationships (abstractions as nodes, labels as edges), and an
   ordered list of links to the chapters.

Use the chapter/index/Mermaid templates in `references/TEMPLATES.md`. Use the
scratch-file shapes in `references/SCHEMAS.md`.

## Large repositories

For read-heavy exploration (steps 1–2), do the file sweeping and abstraction
discovery in a **forked read-only context** (e.g. the Explore subagent) and have
it return only the `manifest.json` and `abstractions.json` content. This keeps
the main context small. Steps 3–6 run in the main context off those scratch files.

## Validation (before writing chapters and index)

Run these checks; if any fails, fix the offending scratch file and re-validate:

- Abstraction count is within **5–`max_abstractions`** (and ≥5, ≤10).
- Every `relationship.from_index` / `to_index` references a **real abstraction**.
- `order.json` lists **each abstraction exactly once** — no missing, no duplicates.
- Every `file_indices` entry exists in the manifest.
- The generated **Mermaid parses**: sanitise every node label — strip or replace
  stray quotes, parentheses, backticks, pipes and newlines; wrap labels in
  `["..."]`. See the sanitisation rule in `references/TEMPLATES.md`.
- **Internal links resolve.** After writing chapters and `index.md`, every
  Markdown link of the form `](NN_*.md)` (chapter-to-chapter and index-to-chapter)
  must point at a file that actually exists in `output_dir`. Fix any dangling link
  before finishing.

## Output layout

```
<output_dir>/
  manifest.json          # scratch — step 1
  abstractions.json      # scratch — step 2
  relationships.json     # scratch — step 3
  order.json             # scratch — step 4
  index.md               # step 6: summary + Mermaid diagram + chapter links
  01_<name>.md           # step 5: one chapter per abstraction, in order
  02_<name>.md
  ...
```

Tell the user where the tutorial was written and suggest they open `index.md`.
