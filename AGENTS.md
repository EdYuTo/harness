# Harness

Wrapper repo for heterogeneous multi-agent work on Paseo. Read this whether you
are Claude Code, Codex, Cursor, or a Paseo-spawned agent — it is the single
source of truth.

## Skills

Shared skill library at `skills/` — a symlink into the
`vendor/cursor-plugins` git submodule (the pstack plugin, MIT). If the path
dangles, run `git submodule update --init --recursive`. Each
`skills/<name>/SKILL.md` is a standalone playbook. Before starting a task,
check for a matching skill and follow it:

- `poteto-mode` — top-level router: picks the playbook (bug fix, feature, perf,
  refactor, …) and enforces the `principle-*` skills.
- `how`, `why`, `recall` — codebase understanding. `blast-radius` — impact
  analysis. `interrogate` — adversarial pre-ship review.
- `architect`, `arena`, `swarm` — design exploration and parallel fan-out.
- `tdd`, `create-verification-skill` — testing. `technical-writing`, `unslop`,
  `no-comments` — prose and diff hygiene.

The same directory is exposed per-harness via symlinks: `.claude/skills`,
`.codex/skills`, `.cursor/skills` → `skills/`.

**Cursor-specific plumbing translation.** These skills were written for Cursor,
so some reference Cursor internals. Map them as follows:

| Skill says                          | Do instead (non-Cursor)                          |
|-------------------------------------|--------------------------------------------------|
| spawn a `Task` subagent with model X| resolve the nearest role in `harness.json`; spawn via your own subagent tool or `paseo run` |
| `pstack-models.mdc` rule            | `harness.json` roles                             |
| `AskQuestion`                       | your harness's ask-user tool, or proceed with the default and say so |
| `cursor-team-kit` skills (`deslop`, `control-*`) | available at `vendor/cursor-plugins/cursor-team-kit/skills/`; else use `unslop` / `no-comments` |

## Roles and models

Model choice is centralized in `harness.json`. Resolve roles there; do not
hardcode model ids in prompts or scripts. `bin/harness <role> "<prompt>"`
launches a role agent through `paseo run` (see `bin/harness --help` via
`bin/harness roles`).

## Orchestration

Multi-task work follows the plan → orchestrate → execute → review loop in
`ORCHESTRATION.md`.

## Ground rules

- Executors work in isolated worktree workspaces, one task per branch.
- Nothing merges without a review-role pass against the task's done-criteria.
- Commit only when asked; never push or open PRs unless the kickoff prompt
  says to.
