# State files

Everything Ralph tracks lives under `.ralph/`:

```
.ralph/
  brief.md                       prose brief, written by /ralph (read-only after that)
  stories.json                   story list (builder flips passes:true; planner may rewrite)
  state.json                     loop state (atomic writes by ralph)
  progress.txt                   append-only human log
  learnings.txt                  cross-iteration patterns
  ralph.pid                      PID of running loop (deleted on exit)
  logs/
    iter-NN-builder.log          human-readable result text
    iter-NN-builder.log.json     raw JSON envelope (cost, stop_reason, session_id, full result)
    iter-NN-planner.log
    iter-NN-planner.log.json
  archive/
    roadmap.v1.json              (only present if migrated from v1)
```
