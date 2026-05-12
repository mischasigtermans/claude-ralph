# Changelog

## [2.0.0] - 2026-05-09

Major rewrite. Two-agent loop, prose brief, state-aware bash runtime.

### Added
- `.ralph/brief.md` — prose brief written by `/ralph`, loaded as plan-time context every iteration
- Planner agent (default Opus) running every Nth iteration as a PM/reviewer; may rewrite `stories.json` and append to `learnings.txt`
- `.ralph/state.json` — atomic, machine-readable loop state; tracks iteration, cadence, cost, status, PID, last commit, last planner note, last log path
- `.ralph/logs/iter-NN-{builder|planner}.log` and `.log.json` — per-iteration audit trail
- `.ralph/ralph.pid` — written on startup, deleted on exit; backs `ralph stop`
- Subcommands: `ralph status`, `ralph stop`, `ralph tail`
- `--max-iter N` flag (replaces v1's positional iteration count)
- `RALPH_BUILDER_MODEL` / `RALPH_PLANNER_MODEL` env vars for model overrides
- `claude --print --output-format json` parsing — captures `cost_usd`, `stop_reason`, `session_id` per iteration
- CLAUDE.md context discovery during `/ralph` PM interview (generic — no vendor coupling)
- Sentinels: `<promise>COMPLETE</promise>` (both roles), `<replan-needed>` (planner only), `<plan-note>` (planner only, captured into state)
- SIGTERM/SIGINT trap on the loop — graceful exit after current iteration
- Auto-migration from v1 `.ralph/` directories (archives `roadmap.json`, generates stub `brief.md`, initializes `state.json`)
- Fail-fast prompt-file existence check before any work

### Changed
- Two prompt files (`scripts/builder.md`, `scripts/planner.md`) replace the single `scripts/ralph-prompt.md`
- Agents are defined inline at runtime via `claude --agents <json>` — they never register as plugin agents and stay hidden from the user's `Agent` tool list
- `SKILL.md` rewritten — PM interview capped at 3 questions, exit criterion is "can I write the brief in two paragraphs?"
- `SKILL.md` frontmatter version corrected (was lagging at 1.1.0)
- Builder stop wording strengthened — "STOP IMMEDIATELY" after one story, no second-iteration drift
- Plugin manifest gains `keywords` field

### Removed
- `--roadmap` mode and "Loopception"
- `--pause` flag
- `roadmap.json` state file (auto-archived on migration)
- Multi-phase scope detection in the SKILL
- `<ready>PHASE_READY</ready>` sentinel
- Positional iteration argument (`ralph 50` — use `ralph --max-iter 50`)
- Interactive update prompt during loop startup (didn't work in detached mode)

### Migrated
- v1 `.ralph/` directories run a one-time migration on first `ralph` invocation. Existing `stories.json`, `progress.txt`, `learnings.txt` are preserved. `roadmap.json` is moved to `.ralph/archive/roadmap.v1.json`. A stub `brief.md` is generated; users are urged to re-run `/ralph` for a proper one.

## [1.1.2] - 2026-01-28
- Fixed version check comparing against oldest cached version instead of newest
- Update prompt asks to run update now (defaults to yes)

## [1.1.1] - 2026-01-28
- Auto-detect roadmap.json and prompt to run in roadmap mode (defaults to yes)

## [1.1.0] - 2026-01-28
- Added Loopception: `--roadmap` flag for multi-phase project orchestration (a loop within a loop)
- Added `--pause` flag to pause between phases for review
- `/ralph` now detects scope and offers to create a roadmap for large tasks
- Roadmap stored as `.ralph/roadmap.json` for reliable jq parsing
- Default iterations changed to infinite (use `ralph 50` to limit)
- Added `ralph update` to refresh symlink after plugin updates
- Added `ralph --version` and `ralph --help`
- Refactored bash script with reusable functions

## [1.0.1] - 2026-01-27
- Installer now creates symlink instead of copying files
- Bash script reads prompt directly from plugin cache (with fallback to ~/.claude/)
- Broken symlink detection removed (no longer needed)
- Re-run installer after plugin updates to fix symlinks

## [1.0.0] - 2026-01-27
- Initial release with `/ralph` skill and autonomous loop bash script
