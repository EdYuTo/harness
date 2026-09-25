# harness

Heterogeneous agent harness on [Paseo](https://paseo.dev): one shared skill
library and one role→model matrix, usable from Claude Code, Codex, Cursor, and
Paseo-spawned agents.

Skills come from the [pstack](https://github.com/cursor/plugins/tree/main/pstack)
plugin (MIT), pulled in as a git submodule — nothing is copied into this repo.

## Install

```sh
git clone --recurse-submodules https://github.com/EdYuTo/harness.git
```

Already cloned without submodules? Fetch them:

```sh
git submodule update --init --recursive
```

The submodule is marked `shallow`, so init only fetches the tip commit. Without
it, `skills/` and the `.claude/.codex/.cursor` skill links dangle — if skills
seem missing, run the `submodule update` line above.

## Layout

```
harness.json          role → provider/model/thinking/mode matrix (edit here)
bin/harness           CLI: `harness <role> [paseo flags] "<prompt>"` → paseo run
ORCHESTRATION.md      the plan → orchestrate → execute → review loop
AGENTS.md             instructions every agent reads (CLAUDE.md symlinks to it)
vendor/cursor-plugins git submodule → github.com/cursor/plugins (pins the version)
skills                → vendor/cursor-plugins/pstack/skills
.claude/skills        → skills   (Claude Code)
.codex/skills         → skills   (Codex)
.cursor/skills        → skills   (Cursor)
.cursor/rules/pstack-models.mdc   Cursor-native pstack role rule (template)
plans/                plan-role output lands here
```

## Quick start

```sh
export PATH="$PWD/bin:$PATH"

harness roles                       # show the matrix
harness plan "…task…"               # Fable 5.1 writes plans/<slug>.md
harness orchestrate -d "…"          # Opus 5 runs the loop (see ORCHESTRATION.md)
harness exec "…one scoped task…"    # GPT-5.6-Luna, direct
harness investigate "how does X work? use skills/how/SKILL.md"
```

Extra flags pass through to `paseo run` (`--new-workspace worktree`,
`--wait-timeout 30m`, `--workspace <id>`, …).

## Using in other projects

The harness is project-agnostic — agents run wherever you point them:

```sh
# one-time, in your shell profile
export PATH="$HOME/Desktop/Projects/harness/bin:$PATH"

# then from any project directory
cd ~/code/my-app
harness plan "…"            # paseo run defaults --cwd to the current directory
harness exec --cwd ~/code/other-app "…"   # or target explicitly
```

To get the pstack skills inside another project's own agent sessions, symlink
the library:

```sh
# per project (Claude Code / Codex / Cursor pick these up)
ln -s ~/Desktop/Projects/harness/skills .claude/skills
ln -s ~/Desktop/Projects/harness/skills .codex/skills
ln -s ~/Desktop/Projects/harness/skills .cursor/skills

# or once, user-level, for every Claude Code session
ln -s ~/Desktop/Projects/harness/skills/how ~/.claude/skills/how   # per skill
```

For the orchestration loop in another repo, paste `ORCHESTRATION.md` into the
orchestrator's kickoff prompt (see that file's "Kicking off a run").

## Model matrix (current)

| Role        | Agent                     | Notes                                   |
|-------------|---------------------------|------------------------------------------|
| plan        | claude/claude-fable-5-1 xhigh, plan mode | plan-alt: codex/gpt-6-astra |
| orchestrate | claude/claude-opus-5 high, acceptEdits   | swap to claude-opus-4-8[1m] for 1M ctx, or cursor/claude-opus-5 |
| exec        | codex/gpt-5.6-luna medium                | review gate compensates      |
| review      | claude/claude-opus-5 high, plan mode     |                              |
| investigate | claude/claude-sonnet-5 high, plan mode   | provisional; alt codex/gpt-5.6-terra |

Change models by editing `harness.json` only.

## Updating pstack

```sh
git -C vendor/cursor-plugins pull origin main
git add vendor/cursor-plugins && git commit -m "Bump pstack"
```
