# Commands and configuration

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

If the loop is unresponsive after 30 seconds, `ralph stop` offers to send SIGKILL.

## Environment variables

| Variable | Default | Purpose |
|---|---|---|
| `RALPH_BUILDER_MODEL` | `claude-sonnet-4-6` | Override the builder model |
| `RALPH_PLANNER_MODEL` | `claude-opus-4-7` | Override the planner model |
