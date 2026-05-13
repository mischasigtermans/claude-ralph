# Claude Ralph

[![Version](https://img.shields.io/badge/dynamic/json?url=https://raw.githubusercontent.com/mischasigtermans/claude-ralph/main/.claude-plugin/plugin.json&query=$.version&label=version&prefix=v)](https://github.com/mischasigtermans/claude-ralph)
[![License](https://img.shields.io/github/license/mischasigtermans/claude-ralph)](LICENSE)

An opinionated take on the Ralph autonomous loop for Claude Code. Describe a feature, walk away, come back to commits.

The concept comes from [Geoffrey Huntley](https://ghuntley.com/ralph/), who named it after Ralph Wiggum from The Simpsons. Ralph is the contributor who keeps showing up, picks the next thing off the list, gets it done, with a project manager looking over the work every few iterations to keep things on track.

## Installation

### Step 1. Install the plugin

```
/plugin marketplace add mischasigtermans/by-mischa
/plugin install ralph@by-mischa
```

### Step 2. Install the bash script

```bash
~/.claude/plugins/cache/by-mischa/ralph/*/scripts/install.sh
```

This symlinks `~/.local/bin/ralph` to the plugin's `ralph.sh`. Make sure `~/.local/bin` is on your `PATH`. After future plugin updates, run `ralph update` to refresh the symlink.

### Requires

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
- `jq` for JSON state management (`brew install jq` on macOS)
- `bash` (zsh users: the script runs under `bash`)

## Quick start

In Claude Code, in your project directory:

```
/ralph "add a user dashboard with auth, profile editing, and an activity feed"
```

This runs a short PM interview (max three questions), writes `.ralph/brief.md`, `stories.json`, `state.json`, and picks a planner cadence. The skill never starts coding. It exits and tells you to run `ralph`.

From a terminal in the project directory:

```bash
ralph                          # foreground
nohup ralph &                  # detached, output to nohup.out
tmux new -s ralph 'ralph'      # in a tmux session
ralph --max-iter 20            # cap at 20 iterations
```

Monitor with `ralph status` or `ralph tail`. Stop with `ralph stop`. The loop runs as a separate process and does not need Claude Code to stay open.

## Features

- Prose brief, not a ticket list. Every iteration loads `.ralph/brief.md` as plan-time context.
- Two-agent loop: Sonnet builder every iteration, Opus planner every Nth.
- State files (`state.json`, `progress.txt`, `learnings.txt`), not log scraping.
- Per-iteration logs with cost, stop reason, and session ID captured.
- Subcommands: `ralph status`, `ralph stop`, `ralph tail`, `ralph update`.
- Auto-migration from 0.x `.ralph/` directories on first run.

## Documentation

- [Architecture](docs/architecture.md): what makes this Ralph different, how the loop works, when to intervene.
- [The brief](docs/brief.md): contract format and an example.
- [Commands and configuration](docs/commands.md): subcommands and environment variables.
- [State files](docs/state.md): the `.ralph/` layout.
- [Migration from 0.x](docs/migration.md): upgrading older projects.

## Related

- [Geoffrey Huntley's original Ralph post](https://ghuntley.com/ralph/): the concept.
- [snarktank/ralph](https://github.com/snarktank/ralph): Ryan Carson's full implementation.
- [Blog post on the v1 setup](https://mischa.sigtermans.me/my-simplified-ralph-loop-setup-for-claude-code).

## Changelog

See [CHANGELOG.md](CHANGELOG.md).

## Credits

- [Mischa Sigtermans](https://mischa.sigtermans.me)
- Concept: [Geoffrey Huntley](https://ghuntley.com/ralph/)

## License

MIT. See [LICENSE](LICENSE).
