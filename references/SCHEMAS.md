# Scratch-file schemas

All scratch files live in `output_dir`. They make large runs debuggable and
resumable: before each step, if its file exists and validates, reuse it. Indices
are integers and stable across the whole run.

- **File index** = position in `manifest.files` (0-based), matching `index`.
- **Abstraction index** = position in `abstractions` array (0-based).

---

## `manifest.json` — step 1

```json
{
  "target": "/abs/path/to/repo",
  "generated_by": "codebase-tutorial",
  "max_file_size_bytes": 102400,
  "include": ["**/*.py", "**/*.md"],
  "exclude": ["**/node_modules/**", "**/.git/**"],
  "files": [
    { "index": 0, "path": "src/app.py",        "size": 1843 },
    { "index": 1, "path": "src/queue.py",      "size": 2210 },
    { "index": 2, "path": "src/worker.py",     "size": 3050 }
  ],
  "skipped": [
    { "path": "assets/logo.png", "reason": "binary" },
    { "path": "data/huge.json",  "reason": "exceeds max_file_size" }
  ]
}
```

`path` is relative to `target`. `skipped` is optional but helpful — use it to
record notable files you curated out (large fixtures, generated migrations,
boilerplate) with a short `reason`.

`include` / `exclude` must contain **valid globs only** — no parenthetical
annotations like `"**/*.json (config only)"`, which are not real globs and break
re-parsing on resume. `files` need not be the full set the globs match; it is the
**curated, tutorial-relevant** subset (see PIPELINE step 1).

---

## `abstractions.json` — step 2

```json
{
  "abstractions": [
    {
      "name": "Task Queue",
      "description": "Like a ticket counter where jobs line up and wait their turn, so the program never tries to do everything at once.",
      "file_indices": [1]
    },
    {
      "name": "Worker",
      "description": "The clerk who takes the next ticket from the queue and actually does the job.",
      "file_indices": [2]
    }
  ]
}
```

Validate: 5 ≤ `abstractions.length` ≤ `max_abstractions` (≤ 10); every
`file_indices` entry exists in the manifest.

---

## `relationships.json` — step 3

```json
{
  "summary": "A small job-processing service: requests come in, get queued, are handled by workers, and the results are reported back. Think of it as a kitchen taking orders, cooking them, and sending out plates.",
  "relationships": [
    { "from_index": 0, "to_index": 1, "label": "enqueues jobs in" },
    { "from_index": 1, "to_index": 2, "label": "feeds jobs to" }
  ]
}
```

`from_index` / `to_index` are **abstraction** indices. Validate: every index is a
valid position in `abstractions`; labels are concise (≤ ~5 words).

---

## `order.json` — step 4

```json
{
  "order": [0, 1, 2, 3]
}
```

An array of **abstraction** indices in teaching order. Validate: it is a
permutation of `0..abstractions.length-1` — each index exactly once, none missing,
none duplicated.

---

## `progress.json` — step 5 (optional)

Helps resume chapter writing without redoing analysis.

```json
{
  "running_summary": "So far: Chapter 1 introduced the Task Queue (jobs lining up); Chapter 2 covered the Worker that processes them.",
  "chapters_written": ["01_task_queue.md", "02_worker.md"],
  "next_order_position": 2
}
```
