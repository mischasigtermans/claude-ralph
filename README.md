# Claude Ralph

[![Version](https://img.shields.io/badge/dynamic/json?url=https://raw.githubusercontent.com/mischasigtermans/claude-ralph/main/.claude-plugin/plugin.json&query=$.version&label=version&prefix=v)](https://github.com/mischasigtermans/claude-ralph)
[![License](https://img.shields.io/github/license/mischasigtermans/claude-ralph)](LICENSE)

An opinionated take on the Ralph autonomous loop for Claude Code. Describe a feature, walk away, come back to commits.

The concept comes from [Geoffrey Huntley](https://ghuntley.com/ralph/), who named it after Ralph Wiggum from The Simpsons. Ralph is the kind of contributor who keeps showing up, picks the next thing off the list, and gets it done, with a project manager looking over the work every few iterations to keep things on track.

## What makes this Ralph different

The core concept comes from Geoffrey Huntley: a contributor that keeps showing up, picks the next thing off the list, gets it done, with a PM looking over the work every few iterations. The implementation choices that distinguish this version from other Ralph runners:

- **A prose brief, not a ticket list.** `.ralph/brief.md` is plain prose: what's being built, what's out of scope, the invariants. Every iteration loads it as plan-time context. Compressing the planning conversation into JSON bullets is fast to write but loses the *why*, and an agent without the why drifts.
- **The PM is a separate agent on a fixed cadence.** Every Nth iteration the loop swaps in a PM-style reviewer (Opus) instead of the usual builder (Sonnet). The planner reads the brief, checks recent commits, may rewrite the story list, append learnings, or halt for a replan. It doesn't write code.
- **State files, not log scraping.** `.ralph/state.json` is the source of truth for iteration count, cadence, cost, status, and PID. Subcommands (`ralph status`, `ralph stop`, `ralph tail`) read it. No grepping logs to find out where the loop is.
- **One flow.** No `--roadmap`, no `--pause`, no 'Loopception' mode, no phase setup. Describe a feature, run `ralph`, monitor or walk away.
- **Per-iteration logs.** `.ralph/logs/iter-NN-{role}.log[.json]` captures every iteration's full output. Audit trail and debugging in one.
- **Auto-migration from 0.x.** If you ran an earlier version of this Ralph, `.ralph/` directories migrate on first run.

If you used 0.x, see [Migration](#migration-from-0x).

## Installation

### Step 1. Install the plugin

```bash
# Add the marketplace (one-time)
/plugin marketplace add mischasigtermans/by-mischa

# Install
/plugin install ralph@by-mischa
```

### Step 2. Install the bash script

The plugin ships with a bash runtime. Run the installer once:

```bash
~/.claude/plugins/cache/by-mischa/ralph/*/scripts/install.sh
```

This symlinks `~/.local/bin/ralph` → the plugin's `ralph.sh`. Make sure `~/.local/bin` is on your `PATH`:

```bash
echo 'export PATH="$PATH:$HOME/.local/bin"' >> ~/.zshrc
source ~/.zshrc
```

### Updating

After updating the plugin in Claude Code, run:

```bash
ralph update
```

That re-runs the installer and refreshes the symlink.

### Requires

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
- `jq` for JSON state management (`brew install jq` on macOS)
- `bash` (zsh users: the script runs under `bash`, not `zsh`)

## Usage

### 1. Set up a project

In Claude Code, in your project directory:

```
/ralph "add a user dashboard with auth, profile editing, and an activity feed"
```

This triggers a short PM interview (max three questions) and writes:

- `.ralph/brief.md`: prose brief
- `.ralph/stories.json`: story list
- `.ralph/state.json`: initial loop state
- `.ralph/progress.txt`, `.ralph/learnings.txt`: loop journals

It also picks a **planner cadence** (3, 5, or 10 iterations) based on the scope and writes that into `state.json`.

The skill never starts coding. It exits and tells you to run `ralph`.

### 2. Run the loop

From a terminal in the project directory:

```bash
ralph                          # foreground
nohup ralph &                  # detached, output to nohup.out
tmux new -s ralph 'ralph'      # in a tmux session
ralph --max-iter 20            # cap at 20 iterations
```

The loop runs as a separate process. It does not need Claude Code to stay open.

### 3. Monitor

From any terminal, in the same project directory:

```bash
ralph status                   # current iteration, cost, last commit, eval note
ralph tail                     # follow .ralph/progress.txt
```

### 4. Stop

```bash
ralph stop                     # SIGTERM the loop, exits after current iteration
```

If the loop is unresponsive after 30 seconds, `ralph stop` offers to send SIGKILL.

## How it works

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
│  5. Write .ralph/logs/iter-NN-{role}.log[.json]               │
│  6. Detect sentinels: COMPLETE / replan-needed / plan-note     │
└────────────────────────────────────────────────────────────────┘
```

### Builder agent (Sonnet by default)

Runs every iteration except when the cadence calls for a review. Reads `brief.md`, `learnings.txt`, `progress.txt`, picks the highest-priority incomplete story, implements it, runs quality checks, commits with `feat: [Story ID] - [Story Title]`, and stops.

### Planner agent (Opus by default)

Runs every Nth iteration (cadence set by the brief). Treats `brief.md` as read-only ground truth. Reads recent commits, progress, learnings, and (optionally) recent logs from `.ralph/logs/`. May:

- Rewrite `.ralph/stories.json` (add, remove, reorder, split, merge)
- Append to `.ralph/learnings.txt`
- Halt the loop with `<replan-needed>` if the brief itself looks wrong

It does not write code. It always ends with a one-line `<plan-note>` that gets captured into `state.json.last_planner_note`.

### Why two agents

0.x was one prompt grinding forward with no checkpoint. If a story drifted from the original intent, nothing caught it until the user reviewed at the end. 1.0's planner is the 'PM at 2am'. The human can't be present every iteration, so a separate prompt (different model, different mandate, no code-writing privileges) plays that role.

## The brief

The most important thing `/ralph` produces is a two-paragraph prose brief. It's the contract every iteration reads. Example:

```markdown
# User Dashboard

## What we're building

A logged-in dashboard at /dashboard for the existing Laravel app, served by a
new DashboardController. Three sections: profile editing (name, email, avatar),
activity feed (last 30 days of the user's own actions, paginated), and account
settings. UI follows the existing Tailwind components in resources/views/components/.

## What's out of scope

No admin views, no impersonation, no multi-user/team features. The activity
feed reads from existing audit_log entries; we are not adding new audit hooks.
Avatar upload uses the existing Spatie Media Library setup. No new storage code.

## Invariants

- All routes go through the existing `auth` middleware
- No changes to the User model migration; new fields go in a separate table
- Tailwind classes only; no new CSS files
- `php artisan test --compact` must pass on every commit

## Cadence

Planner runs every 5 iterations. Moderate scope with a few cross-cutting
concerns (validation, auth) that warrant a mid-run sanity check.
```

If you can't write the brief in two paragraphs, the planning isn't done. Go back to the user with one more question or push back to narrow the scope.

## Subcommands

| Command | Purpose |
|---|---|
| `ralph` | Run the loop until complete, halted, or stopped |
| `ralph --max-iter N` | Cap at N iterations |
| `ralph status` | Pretty-print `.ralph/state.json` |
| `ralph stop` | Graceful stop (SIGTERM, exits after current iteration) |
| `ralph tail` | `tail -f .ralph/progress.txt` |
| `ralph update` | Re-run installer (refresh symlink after plugin update) |
| `ralph -v` | Print version |
| `ralph -h` | Show help |

## Environment variables

| Variable | Default | Purpose |
|---|---|---|
| `RALPH_BUILDER_MODEL` | `claude-sonnet-4-6` | Override the builder model |
| `RALPH_PLANNER_MODEL` | `claude-opus-4-7` | Override the planner model |

## State files

```
.ralph/
  brief.md                       prose brief, written by /ralph (read-only after that)
  stories.json                   story list (builder flips passes:true; planner may rewrite)
  state.json                     loop state (atomic writes by ralph)
  progress.txt                   append-only human log
  learnings.txt                  cross-iteration patterns
  ralph.pid                      PID of running loop (deleted on exit)
  logs/
    iter-NN-builder.log        human-readable result text
    iter-NN-builder.log.json   raw JSON envelope (cost, stop_reason, session_id, full result)
    iter-NN-planner.log
    iter-NN-planner.log.json
  archive/
    roadmap.v1.json              (only present if migrated from v1)
```

## When to intervene

Ralph works best on greenfield features with clear acceptance criteria. Step in when:

- The same story fails twice → it needs splitting
- A story needs context not derivable from the brief or the codebase → add it to `brief.md` and re-run `/ralph`
- The planner outputs `<replan-needed>` → run `/ralph` again to write a new brief
- Tests require manual setup, secrets, or external services → handle outside the loop

## Migration from 0.x

When you upgrade and run `ralph` in an old project:

- `roadmap.json` archived to `.ralph/archive/roadmap.v1.json` (filename retained for backwards compatibility).
- `brief.md` auto-generated as a stub, with a header asking you to re-run `/ralph` for a real brief.
- `state.json` created with cadence defaulted to 5, status `initialized`.
- `progress.txt`, `learnings.txt`, `stories.json` preserved as-is.

The auto-migration keeps things runnable but produces a weak brief. Re-run `/ralph` for a proper one before committing to a long run.

## See also

- [Blog post](https://mischa.sigtermans.me/my-simplified-ralph-loop-setup-for-claude-code) on the v1 setup
- [Geoffrey Huntley's original Ralph post](https://ghuntley.com/ralph/): the concept
- [snarktank/ralph](https://github.com/snarktank/ralph): Ryan Carson's full implementation

## Credits

- [Mischa Sigtermans](https://github.com/mischasigtermans)
- Concept: [Geoffrey Huntley](https://ghuntley.com/ralph/)

## License

MIT
