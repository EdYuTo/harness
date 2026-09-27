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

## Setup

### 1. Wrapper on PATH

```sh
# shell profile
export PATH="$HOME/Desktop/Projects/harness/bin:$PATH"
```

Requires `jq` and the [Paseo](https://paseo.sh) daemon running (`paseo status`).

### 2. pstack skills, globally (all repos, all agents)

Installs the skill library machine-wide so Claude Code, Codex, and Cursor see
it in every repo and every Paseo workspace — this repo's submodule symlinks
then only matter for pinning:

```sh
cd ~   # avoid project-level install
S=(); for s in $(ls <path-to-harness>/vendor/cursor-plugins/pstack/skills); do S+=(-s "$s"); done
npx -y skills add cursor/plugins -g -a claude-code -a codex -a cursor -y "${S[@]}"
# two skills only match by display name:
npx -y skills add cursor/plugins -g -a claude-code -a codex -a cursor -y -s 'Poteto Mode' -s 'Make Bot UI'
```

Gotchas (learned the hard way):

- `-s`/`-a` do **not** take comma lists — repeat the flag per value.
- Agent name is `claude-code`, not `claude`.
- Skills land canonically in `~/.agents/skills/`. Codex (≥0.154) and Cursor
  read that path natively; Claude Code gets per-skill symlinks in
  `~/.claude/skills/`. Empty `~/.codex/skills` afterwards is normal.
- Update later with `npx skills update -g`. The global install and this repo's
  submodule pin are independent — bump both when refreshing pstack.

### 3. Paseo agent profiles (optional, for the UI picker and delegating agents)

Paseo → **Settings → your host → Agents → Agent profiles → New profile**.
Profiles are read-only to agents (`list_profiles`) and per-host — recreate on
each daemon. Mirror `harness.json`, one profile per role
(`harness:plan` = Claude / Fable 5.1 / Plan Mode / xhigh, `harness:exec` =
Codex / GPT-5.6-Luna / Default / medium, …), and paste each role's `notes`
into the profile's "When to use" field — orchestrating agents pick delegation
targets by reading those notes. Avoid Bypass mode unless you want fully
unattended runs.

### 4. Cursor-native pstack roles (optional)

Inside Cursor, run `/setup-pstack` once. It detects that install's real model
slugs and rewrites `.cursor/rules/pstack-models.mdc` (committed here only as a
safe template).

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
cd ~/code/my-app
harness plan "…"            # paseo run defaults --cwd to the current directory
harness exec --cwd ~/code/other-app "…"   # or target explicitly
```

Skills are already everywhere after Setup step 2 (global install). For a repo
that should carry its own pinned copy instead — e.g. so teammates get it from
a plain clone — either run a project-level install
(`cd repo && npx skills add cursor/plugins -a claude-code -a codex -a cursor`)
or replicate this repo's submodule + symlink pattern.

For the orchestration loop in another repo, launch through the wrapper —
`harness orchestrate` embeds `ORCHESTRATION.md` and the role matrix into the
kickoff prompt automatically. Only when starting an orchestrator from the
Paseo UI instead (e.g. via the Harness profile) must you paste
`ORCHESTRATION.md` into the first message yourself; a bare prompt there gives
the agent no roles, and it will do everything on its own model.

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
