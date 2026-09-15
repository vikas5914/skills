---
name: opencode-fast
description: "Claude Code work routing (aliases: opencode-go, opencode-fastscale): delegate implementation, fixing, exploratory sub-agents, rebasing, and PR landing to an OpenCode Go model through OpenCode (opencode2), while the parent Claude session specifies, decides, reviews, and verifies. Apply the native-Claude model gate."
---

# OpenCode Fast

OpenCode Go is a flat-rate plan. Its models are cheap. Claude tokens are metered and
expensive. So: Claude writes the spec, makes the decisions, and reviews the diff.
An OpenCode Go worker types the code.

## Hard gate

Use this skill only when the active agent is Claude Code **and** the session runs on a
native Claude model.

1. Read the model id from the system prompt ("You are powered by the model …").
2. If the id contains `claude`, `fable`, `opus`, `sonnet`, or `haiku`: **delegate hands-on
   work to OpenCode Go.** A router in front of a real Claude model is still expensive Claude.
3. If the id is clearly non-Claude (a GPT or other flat-rate route): work directly.
   Delegation gains nothing there.
4. If you cannot identify the model: fail closed and work directly.

OpenCode itself, Codex, and every other harness: do not self-delegate. Continue the task.

## Route

Delegate to an OpenCode Go worker (default for hands-on work):

- implementation from a frozen spec; refactors; mechanical migrations
- fixing: bug fixes (known repro, or diagnose-then-fix); CI, lint, and type failures;
  test writing; coverage fills
- dependency bumps; scripts and tooling
- exploration: fan out Go workers for read-heavy discovery instead of Claude Explore/Task
  sub-agents when the raw reading is much larger than the answer
- git mechanics: `git rebase`, merge-conflict resolution, and the repo's land workflow.
  Issue ONE self-contained work order that covers rebase, resolve, push, CI green, land.
  The land decision and the gates stay with Claude.
- new work orders go to FRESH sessions. Do not resume a long session for a new order. A
  saturated session reads a work order as configuration and no-ops ("Understood…").
- repo instruction files: point workers at `AGENTS.md`. Edit only `AGENTS.md`.

Keep in Claude:

- design, API design, architecture, naming, UX judgment
- tasks where writing the spec IS the work (ambiguity = design)
- tiny edits (under ~20 lines, one obvious change). Delegation overhead loses.
- mechanical renames across many files. Muse hung 2.5 h on one. Use `perl`/`sd`/`sed`.
- anything that needs session tools: MCP (browser, Linear, Slack, Notion), secrets, 1Password
- releases, publishes, version bumps, and their credentials
- the land decision, pre-land gates (CI green, `opencode-gate` green, proof), and review of
  worker output. Never delegated. Never skipped.

Mixed task: Claude designs first, freezes the spec, then delegates the build-out.
Heuristic: if the prompt reads as a work order, delegate. If writing it forces decisions,
it is design, so Claude does it.

## Models

Provider prefix is `opencode-go/`. List the live catalogue before you pick:

```bash
opencode2 models | rg '^opencode-go/'
```

- Default worker: `opencode-go/muse-spark-1.3-contributor`. Proven on ENG-2704 and the
  modelkit work. Use it unless the user asks for another model.
- Alternatives on the same plan (2026-09-15): `kimi-k2.7-code`, `kimi-k3`, `glm-5.3`,
  `qwen3.8-max`, `deepseek-v4-pro`, `minimax-m3`, `gpt-5.6-luna`, `grok-4.6`. Pick one for a
  second opinion or when Muse loops. State the choice in the work order.
- `opencode/*-free` models are the free tier, not Go. Prefer `opencode-go/*`.
- Do not use a Claude model inside OpenCode for delegated work. That defeats the point.

## Invoke

Two paths. Prefer path A when the Solo MCP is available (see `solo-subagent-hygiene`).

### A. Solo agent tool (preferred)

Solo's OpenCode agent tool runs `opencode2 --yolo`. Find its id with `list_agent_tools`. It starts on
the Go plan with the Solo MCP connected.

1. Write the spec to a file first (see Prompt contract). One spec file per worker.
2. `spawn_agent(agent_tool_id=<opencode id>, cwd=<repo or worktree>, name=<distinct name>)`.
3. `send_input(process_id, "Read <abs spec path> first, then follow it exactly.")`.
4. Wait with `timer_fire_when_idle_any([ids], max_wait_ms, body)`. Never poll in a loop.
5. Read the output with `get_process_output`. Judge the `git diff`, not the summary.
6. `close_process` once you have verified the output.

**Never `send_input` while the agent is busy.** A mid-turn message poisons the session
(`Referenced reasoning item ... was not found or has expired`). Every later prompt then
fails. Wait for idle first. If a session is poisoned, close it and spawn a fresh agent with
a spec for the remainder.

### B. CLI, harness-tracked background (fallback, or when Solo is absent)

```bash
P=$(mktemp); cat >"$P" <<'SPEC'
<goal, repo + key paths, constraints ("do not touch X"), non-goals, proof expected, output shape>
SPEC
cd <repo> && opencode2 run --standalone --auto \
  -m opencode-go/muse-spark-1.3-contributor \
  --title <distinct-name> "$(cat "$P")" > "$LOG" 2>&1
```

- Run the line above as its own Bash `run_in_background: true` call. One tracked chip per
  worker, completion notification included. Chain setup steps (installs, worktree prep)
  INSIDE that tracked command. Never `&`-fork workers from a shared launcher.
- `--standalone` gives each worker a private server. Parallel workers do not share state.
- `--auto` approves permissions that are not explicitly denied. Keep the prompt scoped to
  the target repo.
- `--format json` when you need machine-readable output. Otherwise read the log tail.
- Long runs: do not kill a quiet run under 30 min.
- Parallel independent tasks are fine: separate dirs or worktrees, separate logs, separate
  titles, one tracked background command each.

### Resume (follow-up fixes)

```bash
cd <repo> && opencode2 session list          # find the id
cd <repo> && opencode2 run --standalone --auto \
  -m opencode-go/muse-spark-1.3-contributor \
  -s <session-id> "$(cat "$P2")" > "$LOG" 2>&1
```

- `-s <id>` continues one session. `-c` continues the last session in the cwd. With
  parallel workers on the machine, always pass the explicit id.
- `--fork` branches the session before continuing. Use it for an experiment you may discard.
- The resumed worker keeps its context. The follow-up prompt states only the delta: the
  decision, the amended constraint, what still stands, the required report.
- Model is per launch. Re-pass `-m`.
- A worker that stopped under a spec's escape hatch is not saturated. Resume it with the
  decision. Only start a fresh session for a genuinely new work order.

## When the worker dies instantly

A run that exits in seconds with nothing produced is almost never the task. Read the log
tail before you relaunch.

- `401` or "not authenticated": run `opencode2 auth list`. "OpenCode Go … stored"
  must be present. If not, tell the user. Do not paste a key into argv.
- "model not found": the id is not in `opencode2 models`. Re-list and pick a live id.
- Rate limit or 429 on Go: switch to another `opencode-go/*` model for this worker and say so.
- Hang before any output: check that no other opencode process holds the same data dir
  (`pgrep -fl opencode2`). Use `--standalone`.
- Version drift: `opencode2 --version`. Flags above were verified on
  `v0.0.0-beta-19425`. If a flag is rejected, run `opencode2 run --help` and adapt once.

If the machine's OpenCode setup is broken, say so. Do not silently work around it.

## Prompt contract

The worker starts with zero session context. Every spec has: goal, exact repo and absolute
paths, constraints, non-goals, proof expected (exact test command), output shape ("report
files changed + test output"), and the line "End your final message with `DONE:` and a one
line summary." Spec quality decides success.

- **No git writes by the worker** unless the work order is a git-mechanics order. The
  orchestrator commits, pushes, and opens the PR after review.
- **No `pnpm install`, no repo-wide format.** Muse stalled on pnpm's `approve-builds` prompt
  and wrote a literal placeholder into `pnpm-workspace.yaml`. Pre-install for it.
- **Name the acceptable baseline failures** (exact test names). Otherwise the worker "fixes"
  unrelated tests.
- **Every hard prohibition needs an escape hatch.** "If gate X fails after honest attempts:
  STOP, report exact numbers and diagnosis, do not work around." Treat a stop-report as a
  successful run.
- Multi-PR series: same spec skeleton every PR; cite prior landed PR numbers and their
  idioms. Workers imitate landed precedent far better than abstract style rules.
- End every series work order with an explicit stop: "Do exactly this; do not start PR N+1."

## Coordinator verification (beyond the diff)

Worker reports are often incomplete or stale. Judge from the tree, not the report.

- `git status -sb` and read the full diff. Judge like a contributor PR.
- Run the proof command yourself. Worker claims are advisory.
- Diff-stat the guard, budget, and baseline files the spec forbade touching.
- Test-helper edits are a red-flag class of their own.
- One agent on ENG-2704 skipped the `DONE:` line and still had a complete diff. Another
  wrote a placeholder and reported success. Both were caught only by reading the tree.
- After 2 failed resume rounds, take over and do it directly.
- **Check for a live worker before you edit or commit:** `pgrep -fl opencode2`, or
  `list_processes` in Solo. A run whose deliverable is already in the tree can keep looping
  and overwrite your fixes. Stop it once you have verified its output.
- Normal closeout still applies: the repo's `opencode-gate` CI review must be green before
  merge. Do not ask a Go worker to review its own work in place of that gate.

## Parallel workers, one repo

Disjoint-file tasks parallelize cleanly: one worktree and unique branch per worker (fresh
from the base branch, distinctive branch names), one Solo agent or one tracked background
command each, shared spec body plus a per-target header. Sequential series reuse ONE
worktree and rebranch per PR. Worktree gotcha: a sandboxed worker may fail to commit from a
worktree because the index lock lives under the parent `.git`. The orchestrator publishes.

## Economics

Win = generation and exploration tokens moved to OpenCode Go. Claude spends only on spec
and diff review. Do not ping-pong trivia through delegation. Do not re-read what the worker
already summarized.
