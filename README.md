# teammate

One coding agent hires another — in its own [herdr](https://herdr.dev) tab,
with a written brief, a one-line status contract, and a close command that
refuses to destroy unfinished work.

## What it is

`bin/teammate.mjs` lets an agent session (the **captain**) delegate a task to
a **worker**: a fresh Claude Code session in a new tab of the captain's own
herdr window. The captain writes the task down as a brief file, teammate opens
the tab, starts the agent there, and from then on the worker reports by
appending one-line updates to its status file. Watching is cheap by design:
`check` is a single pass that costs no agent tokens, and `wait` is a slow
background poll that prints nothing until a worker actually wants the captain.

```
node bin\teammate.mjs new --tree "Fix the flaky date test; npm test must pass"
node bin\teammate.mjs wait --timeout 1800
node bin\teammate.mjs log tm-0814-231502
node bin\teammate.mjs say tm-0814-231502 "Keep the fix inside src/dates.ts"
node bin\teammate.mjs close tm-0814-231502
```

## The pieces

- **Brief** — the full task, written to a file the worker is told to read
  first. It carries the working directory, the reporting rules, and what the
  worker must never touch.
- **Status contract** — the worker appends one short line per change of
  situation: `working: …`, `needs-decision: …`, `blocked: …`, `paused: …`,
  and exactly one `done: …` or `failed: …` at the end. Only the first word is
  parsed; silence longer than a few minutes raises attention.
- **Pooled worktrees (`--tree`)** — the worker gets its own copy of the
  repository, leased from treehouse, on a branch named after it and cut
  from the captain's current commit. Parallel workers never collide on
  files; their work meets only when the captain merges the branches. The
  copy is wiped when returned to the pool — **only commits survive** — and
  the brief says so.
- **Cards** — all state lives in `state/teammates/` as small JSON files you
  can read by hand: one card, one brief, one status file per worker.

## What keeps it safe

Four refusals, spelled out in [docs/teammate.md](docs/teammate.md):

1. It will not run outside a herdr pane, and workers do not hire workers.
2. `say` and `close` belong to the hiring captain; another pane must
   explicitly `--steal`.
3. It will not close a tab it cannot prove is still the one it created —
   same tab, same window, same working directory — and never the last tab
   of a window.
4. A record is erased only after herdr itself confirms the pane is gone,
   and a `--tree` copy goes back to the pool only clean: uncommitted
   changes keep the card open and the files salvageable.

Know before delegating: the worker is started with
`claude --dangerously-skip-permissions`, so it acts without asking anyone.
The brief's rules and the `--cwd` you chose are the only fence — scope both
accordingly.

## Requirements

- Windows, Node.js 16+ — plain Node, zero dependencies, no `package.json`.
- [herdr](https://herdr.dev) — the tab/pane manager the captain runs inside.
- [Claude Code](https://claude.com/claude-code) — the agent CLI the worker runs.
- Optional: treehouse (a worktree-pool CLI, not published yet) for `--tree`
  pooled worktrees; without it, workers share the captain's folder.

Everything is local: teammate talks only to the herdr CLI, git, and the
files under `state/`.

## Docs

- [docs/teammate.md](docs/teammate.md) — the field guide: every command,
  the card schema, the status contract, the refusals.
- [docs/herdr-api.md](docs/herdr-api.md) — notes on the herdr CLI surface
  this tool builds on.

## Sibling project

[sheepdog](https://github.com/patraianton/sheepdog) — a live kanban board
over the same herdr fleet: card per agent session, columns by state, one
human operator. teammate grew out of it and shares its philosophy: the
tools watch, the human decides.

## License

MIT — see `LICENSE`.
