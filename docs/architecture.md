# Architecture

The implementation choices that distinguish this Ralph from other runners.

## A prose brief, not a ticket list

`.ralph/brief.md` is plain prose: what's being built, what's out of scope, the invariants. Every iteration loads it as plan-time context. Compressing the planning conversation into JSON bullets is fast to write but loses the *why*, and an agent without the why drifts.

## Two agents, fixed cadence

Every Nth iteration the loop swaps the usual builder (Sonnet) for a PM-style reviewer (Opus). The planner reads the brief, checks recent commits, may rewrite the story list, append learnings, or halt for a replan. It doesn't write code.

0.x was one prompt grinding forward with no checkpoint. If a story drifted from the original intent, nothing caught it until the user reviewed at the end. The planner is the 'PM at 2am': a separate prompt (different model, different mandate, no code-writing privileges) that plays that role while the human is away.

## State files, not log scraping

`.ralph/state.json` is the source of truth for iteration count, cadence, cost, status, and PID. Subcommands (`ralph status`, `ralph stop`, `ralph tail`) read it. No grepping logs to find out where the loop is.

## One flow

No `--roadmap`, no `--pause`, no 'Loopception' mode, no phase setup. Describe a feature, run `ralph`, monitor or walk away.

## Per-iteration logs

`.ralph/logs/iter-NN-{role}.log[.json]` captures every iteration's full output. Audit trail and debugging in one.

## How a run unfolds

```
┌────────────────────────────────────────────────────────────────┐
│  /ralph "feature description"                                  │
│  PM interview → .ralph/brief.md + stories.json + state.json    │
└────────────────────────────────────────────────────────────────┘
                              ↓
┌────────────────────────────────────────────────────────────────┐
│  ralph (bash loop, detached process)                           │
│                                                                │
│  Each iteration:                                               │
│  1. Read state.json, decide role (builder / planner)           │
│  2. Build inline agent definition via `claude --agents <json>` │
│  3. Run claude --print --output-format json                    │
│  4. Parse result, update .ralph/state.json atomically          │
│  5. Write .ralph/logs/iter-NN-{role}.log[.json]                │
│  6. Detect sentinels: COMPLETE / replan-needed / plan-note     │
└────────────────────────────────────────────────────────────────┘
```

### Builder agent (Sonnet by default)

Runs every iteration except when the cadence calls for a review. Reads `brief.md`, `learnings.txt`, `progress.txt`, picks the highest-priority incomplete story, implements it, runs quality checks, commits with `feat: [Story ID] - [Story Title]`, and stops.

### Planner agent (Opus by default)

Runs every Nth iteration (cadence set by the brief). Treats `brief.md` as read-only ground truth. Reads recent commits, progress, learnings, and (optionally) recent logs from `.ralph/logs/`. May:

- Rewrite `.ralph/stories.json` (add, remove, reorder, split, merge).
- Append to `.ralph/learnings.txt`.
- Halt the loop with `<replan-needed>` if the brief itself looks wrong.

It does not write code. It always ends with a one-line `<plan-note>` that gets captured into `state.json.last_planner_note`.

## When to intervene

Ralph works best on greenfield features with clear acceptance criteria. Step in when:

- The same story fails twice. It needs splitting.
- A story needs context not derivable from the brief or the codebase. Add it to `brief.md` and re-run `/ralph`.
- The planner outputs `<replan-needed>`. Run `/ralph` again to write a new brief.
- Tests require manual setup, secrets, or external services. Handle outside the loop.
