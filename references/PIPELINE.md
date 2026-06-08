# Pipeline — detailed procedure

This is the full procedure for the six steps. The agent reads the code and **is**
the model: every "ask the LLM" step in the original PocketFlow pipeline becomes
the agent reasoning directly over file contents it has read. No API calls, no
network.

Throughout, files are referenced by their **integer index** from the manifest.
Abstractions are referenced by their **position in the abstractions list** (0-based).

---

## Step 1 — Fetch

**Goal:** a stable manifest of the relevant source files.

1. Resolve `target` to an absolute path. Treat it as read-only for the whole run.
2. Enumerate files under `target` matching the `include` globs and not matching
   the `exclude` globs, skipping any file larger than the max file size.
3. **Max file size:** default **100 KB**. Skip larger files (record them as
   skipped if useful, but do not read them).
4. **Default exclude** (always skipped, even if include globs would match):
   ```
   **/.git/**            **/node_modules/**     **/dist/**
   **/build/**           **/out/**              **/.next/**
   **/.venv/**           **/venv/**             **/__pycache__/**
   **/target/**          **/vendor/**           **/.idea/**
   **/.vscode/**         **/coverage/**         **/*.min.js
   **/*.lock            **/*.map               **/.terraform/**
   **/bin/**             **/obj/**              **/Pods/**
   ```
   Also skip binary/asset files (images, fonts, archives, compiled artifacts).
5. **Default include** when the user gives none — common source extensions:
   ```
   **/*.py  **/*.js  **/*.jsx **/*.ts  **/*.tsx **/*.go  **/*.rs
   **/*.java **/*.kt **/*.rb  **/*.php **/*.c   **/*.h   **/*.cpp
   **/*.cs  **/*.swift **/*.scala **/*.ex **/*.exs **/*.sh
   **/*.md  **/*.yaml **/*.yml **/*.toml **/*.json
   ```
   Prefer the project's primary language; include READMEs and key config
   (`package.json`, `pyproject.toml`, `go.mod`, `Cargo.toml`) — they reveal entry
   points and structure.
   - **`include`/`exclude` must contain only valid globs** — never annotate a
     glob with parenthetical notes (e.g. `**/*.json (config only)`). An annotated
     string is not a real glob and will break any stricter or resumable re-run
     that re-parses the manifest. Express intent through patterns instead: to keep
     only config JSON and drop data/seed JSON, rely on `exclude` (e.g.
     `**/fixtures/**`, `**/*.seed.json`) rather than a comment inside `include`.
6. Sort the kept paths deterministically (lexicographic) and assign indices
   `0, 1, 2, …`. Read each kept file's contents.
7. Write `manifest.json` (see SCHEMAS.md). Store the path **relative to `target`**.

**Curate to tutorial-relevant source.** The manifest is not required to list every
file the globs match. After applying include/exclude and the size cap, keep the
files a newcomer actually needs to understand the system — entry points, core
modules, the data model, public APIs, key config, READMEs — and drop boilerplate
and noise (framework scaffolding, generated migrations, large seed fixtures,
near-empty `__init__` files). Record anything notable you dropped under `skipped`
with a short `reason`, so the choice is auditable. A focused 15–40 file manifest
usually makes a better tutorial than an exhaustive one.

**Cap:** if the manifest exceeds ~100 files or the total content is very large,
prefer the most structurally significant files (entry points, core modules,
public APIs, config) and note the cap in the project summary later. Use the
large-repo strategy below.

---

## Step 2 — Identify abstractions

**Goal:** 5–10 core abstractions a newcomer must understand.

For each abstraction record:
- `name` — short, beginner-friendly (e.g. "Request Router", "Task Queue").
- `description` — plain language, **using an analogy** ("like a post office that
  sorts letters into pigeonholes…"). Avoid jargon; explain it if unavoidable.
- `file_indices` — the manifest indices of files that implement it.

Guidance:
- Aim for the *conceptual building blocks*, not every class. Group related code.
- Favour things with clear responsibilities and clear boundaries.
- Stay within `5`–`max_abstractions` (hard ceiling 10). If you find more,
  merge the smallest/overlapping ones.

Write `abstractions.json` (see SCHEMAS.md). The order here is the *discovery*
order; teaching order is decided in step 4.

---

## Step 3 — Analyse relationships

**Goal:** the big picture and how abstractions interact.

1. Write a **one-paragraph** `summary` of what the project does and its overall
   shape — beginner-friendly, analogy-friendly.
2. Produce `relationships`: a list of `{from_index, to_index, label}` where the
   indices are **positions in the abstractions list** and `label` is a concise
   verb phrase describing the interaction ("sends jobs to", "reads config from",
   "renders output of"). Keep labels ≤ ~5 words. **The edge must read naturally
   left-to-right as "{from} {label} {to}"** — pick the direction and verb so the
   sentence is true (e.g. `Engine "totals up" Price Row`, not `Price Row "is built
   from" Engine`). If the natural sentence runs the other way, swap `from`/`to`.
3. Aim for enough edges that every abstraction is connected to at least one
   other; avoid a dense hairball — capture the *primary* interactions.

Write `relationships.json` (see SCHEMAS.md).

---

## Step 4 — Order chapters

**Goal:** a teaching order.

Order the abstraction indices so that:
- Foundational concepts and **entry points** come first (the thing a reader meets
  first when the program runs, or the core data model everything builds on).
- Dependents come after the things they depend on.

Write `order.json` — an array of abstraction indices, each appearing **exactly
once** (see SCHEMAS.md).

---

## Step 5 — Write chapters (map-reduce)

**Goal:** one Markdown chapter per abstraction, in `order.json` sequence.

For each abstraction (the "map"):
- Use the chapter template in TEMPLATES.md.
- **Motivate "why it exists" before "how it works".** Open with the problem it
  solves and an analogy.
- Ground everything in the **actual code**: cite files by their manifest path and
  include **short** snippets (a few lines, not whole files). Explain snippets
  line-by-relevant-line.
- Cross-link to other chapters using their `NN_name.md` filenames.
- **Running summary (the "reduce"):** maintain a short running summary of chapters
  written so far. Feed it into each new chapter's context so transitions read
  naturally ("In the previous chapter we saw the Task Queue; now we follow a job
  into the Worker…"). Keep the running summary in working context; you may also
  append it to a `progress.json` scratch file if helpful for resuming.

File naming: zero-padded order position + slugified abstraction name, e.g.
`01_request_router.md`, `02_task_queue.md`.

Chapters can be re-run without redoing analysis because steps 1–4 are persisted.

---

## Step 6 — Combine

**Goal:** the `index.md` landing page.

1. Create `output_dir` if needed.
2. Write `index.md` using the index template in TEMPLATES.md, containing:
   - The project `summary`.
   - A Mermaid **`flowchart TD`** built from `relationships`: one node per
     abstraction (label = abstraction name, **sanitised**), one edge per
     relationship (`from -->|label| to`, label sanitised).
   - An **ordered list of links** to the chapters, following `order.json`.
3. Validate the Mermaid parses (see TEMPLATES.md sanitisation rule) before writing.
4. **Validate internal links.** Scan `index.md` and every chapter for Markdown
   links to `NN_*.md` files; confirm each target exists in `output_dir`. Repair any
   dangling link (usually a slug or zero-pad mismatch) before finishing.

Finish by telling the user the output location and pointing them at `index.md`.

---

## Large-repository strategy

For steps 1–2 on a big repo, offload the read-heavy work to a **forked read-only
context** so the main conversation stays small:

- In Claude Code: spawn the **Explore** subagent (read-only) to enumerate files,
  apply the globs/size cap, and propose abstractions. Ask it to return **only**
  the JSON for `manifest.json` and `abstractions.json` — not file dumps.
- In other runtimes: run the exploration in a separate/forked agent context and
  collect the same two JSON artifacts.

Persist those artifacts to `output_dir`, then run steps 3–6 in the main context
reading the scratch files (and re-reading only the specific files a chapter needs).
