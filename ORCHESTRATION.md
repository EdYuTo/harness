# Orchestration loop

Heterogeneous multi-agent loop on Paseo. Three roles, three model families, one
review gate. Role → model resolution lives in `harness.json`; never hardcode a
model in a prompt — name the role.

```
 user task
    │
    ▼
 PLAN  (harness plan — Claude Fable 5.1, plan mode, xhigh)
    │  writes plans/<slug>.md: numbered tasks, each independently verifiable,
    │  with explicit done-criteria and files touched
    ▼
 ORCHESTRATE  (harness orchestrate — Claude Opus 5, acceptEdits)
    │  per task, loop:
    │    1. spawn EXEC (GPT-5.6-Luna) in an isolated worktree workspace
    │    2. when it finishes, REVIEW the diff against the task's done-criteria
    │    3. verdict:
    │         pass  → merge/keep branch, next task
    │         fail  → send_agent_prompt back to the same Luna agent with the
    │                 review findings; re-review. 2 strikes → reassign the task
    │                 to orchestrate's own model and note why.
    ▼
 done: orchestrator writes SUMMARY (what shipped, what's left, deviations from plan)
```

## Kicking off a run

```sh
# 1. Plan (blocks until the plan file exists)
bin/harness plan --cwd "$PWD" "write plans/<slug>.md for: <task>. \
Number the tasks; each gets done-criteria and the files it touches."

# 2. Orchestrate (runs long; background it)
bin/harness orchestrate -d --cwd "$PWD" "$(cat ORCHESTRATION.md) \
--- Execute plans/<slug>.md following the loop above."
```

## Rules for the orchestrator agent

You are a Paseo agent: use your agent-scoped Paseo tools, not the CLI.

- **Spawning executors**: `create_agent` with provider/model from the `exec`
  role in `harness.json`, in a fresh worktree workspace
  (`create_workspace` → isolation `worktree`, mode `branch-off`,
  `branchName: task-<n>-<slug>`, `baseBranch: main`). One task per agent, one
  agent per worktree. Independent tasks run in parallel; leave
  `notifyOnFinish` on and pick up each one as it lands.
- **Executor prompts** must be self-contained: paste the task text and its
  done-criteria from the plan; never say "see the plan".
- **Reviewing**: review diffs yourself (you are the `review` role's model), in
  read-only terms: diff vs. done-criteria, tests actually run, no scope creep,
  no dead code. For a huge diff, spawn a `review`-role agent on
  `claude-opus-4-8[1m]` instead of paging through it yourself.
- **Retries**: send findings back to the same Luna agent via
  `send_agent_prompt` — it has the context. Two failed re-reviews on one task
  → do the task yourself and record the failure mode in the summary.
- **Investigation**: unknowns about the codebase go to an `investigate`-role
  agent (see `harness.json`), prompted with the relevant pstack skill from
  `skills/` (`how`, `why`, `blast-radius`, `recall`).
- **Never** push, merge to main, or open PRs unless the kickoff prompt says to.

## Role cheat sheet

| Role        | Model                      | Why                                      |
|-------------|----------------------------|------------------------------------------|
| plan        | claude/claude-fable-5-1    | strongest long-horizon planner           |
| plan-alt    | codex/gpt-6-astra          | second opinion / quota fallback          |
| orchestrate | claude/claude-opus-5       | judgment + tool use; 4.8[1m] for context |
| exec        | codex/gpt-5.6-luna         | fast, cheap; review gate catches errors  |
| review      | claude/claude-opus-5       | same brain as orchestrator, consistent   |
| investigate | claude/claude-sonnet-5     | provisional — see harness.json notes     |
