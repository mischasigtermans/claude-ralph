# Ralph

An opinionated take on the Ralph autonomous loop for Claude Code. Describe a feature, walk away, come back to commits.

The concept comes from [Geoffrey Huntley](https://ghuntley.com/ralph/), who named it after Ralph Wiggum from The Simpsons. Ralph is the kind of contributor who keeps showing up, picks the next thing off the list, and gets it done — with a project manager looking over the work every few iterations to keep things on track.

Available through the [Ryde Ventures plugin marketplace](https://github.com/rydeventures/claude-plugins).

## What's in v2.0

Ralph 2.0 is a substantial rethink of v1. The headline differences:

- **One flow.** No more `--roadmap`, `--pause`, "Loopception" mode, or phase setup.
- **A prose brief.** `.ralph/brief.md` is plain prose — what's being built, what's out of scope, the invariants. Every iteration loads it as plan-time context. v1's biggest failure mode was compressing a planning conversation into JSON bullets and then losing the why.
- **An planner agent.** Every Nth iteration the loop swaps in a PM-style reviewer (Opus) instead of the usual builder (Sonnet). The planner reads the brief, checks the recent commits, and may rewrite the story list, append learnings, or halt for a replan.
- **State-aware loop.** `.ralph/state.json` is the source of truth for iteration count, cadence, cost, status, and PID. Subcommands (`ralph status`, `ralph stop`, `ralph tail`) read it.
- **Per-iteration logs.** `.ralph/logs/iter-NN-{role}.log[.json]` captures every iteration's full output — audit trail and debugging in one.
- **Auto-migration.** v1 `.ralph/` directories migrate on first run.

If you used v1, see [Migration](#migration-from-v1).

## Installation

### Step 1 — Install the plugin

```bash
# Add the Ryde Ventures marketplace (one-time)
/plugin marketplace add rydeventures/claude-plugins

# Install
/plugin install ralph@rydeventures-claude-plugins
```

### Step 2 — Install the bash script

The plugin ships with a bash runtime. Run the installer once:

```bash
~/.claude/plugins/cache/rydeventures-claude-plugins/ralph/*/scripts/install.sh
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

### Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
- `jq` for JSON state management (`brew install jq` on macOS)
- `bash` (zsh users — the script runs under `bash`, not `zsh`)

## Usage

### 1. Set up a project

In Claude Code, in your project directory:

```
/ralph "add a user dashboard with auth, profile editing, and an activity feed"
```

This triggers a short PM interview (max three questions) and writes:

- `.ralph/brief.md` — prose brief
- `.ralph/stories.json` — story list
- `.ralph/state.json` — initial loop state
- `.ralph/progress.txt`, `.ralph/learnings.txt` — loop journals

It also picks an **planner cadence** (3, 5, or 10 iterations) based on the scope and writes that into `state.json`.

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
│  1. Read state.json — decide role (builder / planner)       │
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

v1 was one prompt grinding forward with no checkpoint. If a story drifted from the original intent, nothing caught it until the user reviewed at the end. v2's planner is the "PM at 2am" — the human can't be present every iteration, so a separate prompt (different model, different mandate, no code-writing privileges) plays that role.

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
Avatar upload uses the existing Spatie Media Library setup — no new storage code.

## Invariants

- All routes go through the existing `auth` middleware
- No changes to the User model migration; new fields go in a separate table
- Tailwind classes only; no new CSS files
- `php artisan test --compact` must pass on every commit

## Cadence

Planner runs every 5 iterations. Moderate scope with a few cross-cutting
concerns (validation, auth) that warrant a mid-run sanity check.
```

If you can't write the brief in two paragraphs, the planning isn't done — go back to the user with one more question or push back to narrow the scope.

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

## Migration from v1

When you upgrade and run `ralph` in an old project:

- `roadmap.json` → archived to `.ralph/archive/roadmap.v1.json`
- `brief.md` → auto-generated as a stub, with a header asking you to re-run `/ralph` for a real brief
- `state.json` → created with cadence defaulted to 5, status `initialized`
- `progress.txt`, `learnings.txt`, `stories.json` → preserved as-is

The auto-migration keeps things runnable but produces a weak brief. Re-run `/ralph` for a proper one before committing to a long run.

## See also

- [Blog post](https://mischa.sigtermans.me/my-simplified-ralph-loop-setup-for-claude-code) on the v1 setup
- [Geoffrey Huntley's original Ralph post](https://ghuntley.com/ralph/) — the concept
- [snarktank/ralph](https://github.com/snarktank/ralph) — Ryan Carson's full implementation

## Credits

- [Mischa Sigtermans](https://github.com/mischasigtermans)
- Concept: [Geoffrey Huntley](https://ghuntley.com/ralph/)

## License

MIT
