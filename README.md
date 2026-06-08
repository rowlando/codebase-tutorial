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
