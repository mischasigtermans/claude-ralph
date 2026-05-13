# Migration from 0.x

When you upgrade and run `ralph` in an old project:

- `roadmap.json` archived to `.ralph/archive/roadmap.v1.json` (filename retained for backwards compatibility).
- `brief.md` auto-generated as a stub, with a header asking you to re-run `/ralph` for a real brief.
- `state.json` created with cadence defaulted to 5, status `initialized`.
- `progress.txt`, `learnings.txt`, `stories.json` preserved as-is.

The auto-migration keeps things runnable but produces a weak brief. Re-run `/ralph` for a proper one before committing to a long run.
