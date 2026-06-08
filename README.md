# codebase-tutorial

An Agent Skill that turns a codebase into a beginner-friendly, multi-chapter
Markdown tutorial with an architecture diagram. Built on the open `SKILL.md`
standard, so it's portable across agent runtimes.

See [`SKILL.md`](./SKILL.md) for the full instructions the agent follows.

## Install

Copy this whole `codebase-tutorial/` folder into one of the following:

| Runtime                  | Location                                       |
|--------------------------|------------------------------------------------|
| Claude Code (project)    | `.claude/skills/codebase-tutorial/SKILL.md`    |
| Claude Code (personal)   | `~/.claude/skills/codebase-tutorial/SKILL.md`  |
| GitHub Copilot CLI       | `~/.copilot/agents/codebase-tutorial/SKILL.md` |
| Repo-portable            | `.github/skills/codebase-tutorial/SKILL.md`    |

## Reference files

Loaded on demand to keep `SKILL.md` lean:

- `references/PIPELINE.md` — full step-by-step procedure
- `references/TEMPLATES.md` — chapter / index / Mermaid templates
- `references/SCHEMAS.md` — JSON schemas for the scratch files

## Hints & tips: steering what the tutorial covers

By default the skill decides what to teach by reading the code and giving balanced
weight to the codebase's main parts. For most repos that's exactly right — **just
point it at the repo and let it run.** You don't need any of the options below to
get a good tutorial.

The thing to know: a tutorial can lean two ways, and which one you want depends on
*why you're reading*.

- **Essence** — what the project is *for*: its domain model, core algorithm, the
  thing it uniquely does. This is the codebase-specific knowledge.
- **Plumbing** — auth, sessions, middleware, config, logging, framework bootstrap.
  Necessary, but generic; you could learn most of it from any tutorial for the
  same stack.

The skill now leans toward **essence-first** by default and condenses generic
plumbing so it doesn't crowd out the domain. But where a codebase's mass sits
still influences the result — a thin app on a heavy framework template can still
come out plumbing-heavy. When you want to steer it, just say so in plain language;
the skill takes a `focus`:

```
# Default — balanced, essence-first. Good for most repos.
Generate a tutorial for this codebase.

# Emphasise purpose over plumbing
Explain this codebase, but focus on what the app actually does — its domain and
data — and keep auth, sessions, and middleware to brief background.

# Deep-dive one area (great when one part is large or is the reason you're here)
Walk me through this repo with a focus on the pricing engine — I want several
chapters on how fees are calculated, everything else as context.

# Onboard onto the infrastructure instead (the opposite lean, when that's the job)
Onboard me to this codebase's platform wiring: the middleware stack, auth flow,
and session handling.
```

Other levers for sharper control:

- **Scope to a subtree** with `target` or `include` — e.g. point `target` at
  `src/journeys` to tutorialise just that area. Blunt but effective for a true
  deep-dive; you lose the surrounding context, so prefer `focus` unless you really
  want only that subtree.
- **`max_abstractions`** (5–10) — raise it when a focused area has many distinct
  parts worth their own chapters; lower it for a tighter overview.
- **Two-pass approach** — run once with defaults for the big picture (into
  `./codebase-tutorial`), then again with a `focus` and a different `output_dir`
  (e.g. `./codebase-tutorial-pricing`) for a deep-dive on the part that matters.

Rule of thumb: **don't reach for these on the first run.** Read the default
tutorial, and if it spent too many chapters on generic plumbing or skimmed the
part you came to understand, re-run with a `focus`.

## Permissions

When you run the skill, the working directory is the **target codebase** being
tutorialised. The skill's own reference files live wherever you installed the
skill (see the Install table above) — typically *outside* that working
directory. Many agent runtimes guard file reads outside the project and will ask
you to approve each read of `PIPELINE.md`, `TEMPLATES.md`, or `SCHEMAS.md`. This
is the runtime's generic "reading a file outside the project" guard, not a flaw
in the skill.

How you silence those prompts depends on your runtime. **In Claude Code**, add a
standing allow-rule for your skills directory to your **global**
`~/.claude/settings.json` (applies in every project):

```json
{
  "permissions": {
    "allow": [
      "Read(//<your-home>/.claude/skills/**)"
    ]
  }
}
```

Note the double-slash `//` prefix — that's the absolute-path form Claude Code's
permission matcher expects (e.g. `Read(//Users/alex/.claude/skills/**)` on macOS,
`Read(//home/alex/.claude/skills/**)` on Linux). Reads under the skills directory
are read-only and self-authored, so allowing them broadly is safe. Restart Claude
Code (or start a new session) for the change to take effect.

**Other runtimes handle this differently.** GitHub Copilot CLI and other agents
have their own permission models, config locations, and rule syntax — the
settings file and `Read(...)` rule above are Claude Code specific and won't apply
verbatim. Consult your runtime's documentation for how to allow reads from its
skills/agents directory.

The skill only ever **writes** into its `output_dir` (inside your project), so no
write permissions outside the working directory are needed.
