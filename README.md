# teammate

teammate lets one AI coding agent hand a task to a second one and keep an eye
on it. It is for people who run several Claude Code sessions at once and want
every handed-off task to have a written brief, a status log they can read, and
a close step that will not throw away changes the worker has not yet saved to
git (committed).

The first agent (the captain) runs `bin/teammate.mjs` inside
[herdr](https://herdr.dev), a terminal manager that runs many coding-agent
sessions side by side in tabs and reports what each one is doing. Each tab
holds one or more panes (terminal areas). The tool opens a new tab in the same
herdr window, writes the task into a brief file, starts Claude Code there as
the worker, and from then on the worker reports by appending one-line updates
to a status file.

Claude Code can also start helpers inside one session (subagents). A teammate
worker is different: it is a full, separate Claude Code session in its own
tab. It keeps its own memory of the conversation, a human can open its tab and
type into it, it keeps running after the captain's session ends, and the
captain spends no AI tokens while the `wait` command watches it.

A typical run: hire a worker in its own copy of the repository (`--tree`),
wait up to 30 minutes (1800 seconds) until it needs the captain, read its log,
send a follow-up, and close it. `new` prints the worker's id, here
tm-0814-231502 (the month, day and time of the hire).

```
node bin\teammate.mjs new --tree "Fix the flaky date test; npm test must pass"
node bin\teammate.mjs wait --timeout 1800
node bin\teammate.mjs log tm-0814-231502
node bin\teammate.mjs say tm-0814-231502 "Keep the fix inside src/dates.ts"
node bin\teammate.mjs close tm-0814-231502
```

## What it does

- Opens a worker tab. `new` creates a tab in the captain's own herdr window
  without taking focus, writes the brief, and launches Claude Code with the
  instruction to read that brief and carry out the task.
- Can give each worker its own copy of the repository. With `--tree` the
  worker gets a git worktree leased from treehouse, a small companion tool
  that keeps a pool of such copies ready. The copy sits on a branch named
  after the worker and cut from the captain's current commit. Parallel workers
  never touch each other's files; the captain merges their branches afterwards.
- Watches without spending AI tokens. `check` reads the status file and asks
  the herdr CLI about the pane in one pass; no Claude call is involved. `wait`
  repeats that pass every 15 seconds and prints nothing until a worker needs
  the captain.
- Keeps the conversation in one session. `say` sends more text into the
  worker's running Claude Code session and records the message in the status
  file as a `captain:` line.
- Closes only finished work. `close` refuses until the worker wrote `done:` or
  `failed:`, and for a `--tree` worker it also refuses while the worker's copy
  holds uncommitted changes.
- Keeps a plain-text record. Every worker has one card (a small JSON file with
  its state and ids), one brief and one status file under `state/teammates/`.

## How a worker reports, and how `check` reads it

The worker appends one short line to `state/teammates/<id>.status` whenever
its situation changes. Only the first word before the colon is parsed.

| Word | Meaning |
|---|---|
| `working` | going fine, nothing needed from anyone |
| `needs-decision` | the captain must choose something |
| `blocked` | cannot continue (no access, tool broken) |
| `paused` | stopped on purpose, will resume |
| `done` | finished, and the result is saved |
| `failed` | cannot be finished, with the reason |

The brief tells every worker to write `working: ...` as its very first action,
one line at every real change of situation, and exactly one `done:` or
`failed:` line at the end. After the last line the worker stays in its tab; the
captain reads the work, may `say` more into the same session, and closes the
tab itself.

`check` combines three sources into one verdict: the last recognised status
line, `herdr agent get` for the pane, and the time since the status file last
changed. It flags a worker when:

- the worker says `done` or `failed`;
- the worker says `needs-decision` or `blocked`, and the captain has not yet
  replied with `say`;
- the worker's pane is gone;
- the card is stuck in `failed-to-create` or `launch-problem`;
- the worker has shown no sign of life for more than 4 minutes.

The silence rule has two exceptions. A worker that herdr reports as `working`
counts as alive even when it writes nothing. A worker whose last line is
`paused` gets no silence alarm. herdr reporting a worker as `idle` never
counts as finished; only the worker's own `done:` or `failed:` line does.

## What a card holds

A card moves through the states `creating`, `launching`, `running` and
`closed`, or stops at `failed-to-create` or `launch-problem`. It holds the
worker's task, working directory, captain ids, tab and pane ids, timestamps,
the lease and branch of its copy for `--tree` workers, and the worker's final
line after closing. The full field list is in
[docs/teammate.md](docs/teammate.md#what-a-card-holds).

## How a worker is started

`new` runs these steps in order (all in `bin/teammate.mjs`):

1. It checks that it runs inside a herdr pane. It requires `HERDR_ENV=1` plus
   `HERDR_PANE_ID`, `HERDR_TAB_ID` and `HERDR_WORKSPACE_ID` (herdr calls a
   window a workspace), and confirms those ids with `herdr pane get`. A pane
   that is itself an open worker is refused, so workers cannot hire workers.
2. It claims an id of the form `tm-<MMDD-HHMMSS>` by creating the card
   `state/teammates/<id>.json` with an exclusive-create flag, so two hires in
   the same second get different ids. The card is on disk before the tab
   exists; a crash mid-launch leaves a record instead of an orphan tab.
3. With `--tree`, it runs `treehouse get --lease --json --lease-holder <id>`
   and then `git switch -c <id> <captain's current commit>` inside the leased
   copy. If treehouse refuses to lease, the card is marked `failed-to-create`.
   If the branch cannot be created, the copy goes back to the pool first and
   the card is marked the same way.
4. It writes `<id>.brief.md` (the task, the working directory, the reporting
   rules, and for `--tree` workers a section saying that only commits survive)
   and an empty `<id>.status`. It then runs `herdr tab create` with
   `--no-focus` and passes `TEAMMATE_ID`, `TEAMMATE_BRIEF` and
   `TEAMMATE_STATUS` to the new tab. A new tab starts with herdr's own
   environment, not the captain's, so the worker could end up on a different
   Claude account or on none. To prevent that, teammate also passes whichever
   of `CLAUDE_CONFIG_DIR`, `ANTHROPIC_API_KEY`, `ANTHROPIC_AUTH_TOKEN`,
   `ANTHROPIC_BASE_URL` and `ANTHROPIC_MODEL` are set in the captain's
   environment.
5. It waits up to 15 seconds for a shell prompt, types
   `claude --dangerously-skip-permissions "Read <brief> and carry out the task in it, fully."`
   with `herdr pane run`, presses Enter on Claude Code's "do you trust this
   folder" question, and gives the worker up to 60 seconds to reach
   `agent_status: working` in `herdr agent get`. A "Not logged in" screen or
   no activity marks the card `launch-problem`; otherwise the card moves to
   `running`.

## What close refuses to do

`close <id>` removes the worker's tab and retires its card. Before it acts it
checks, in this order:

1. The caller must be the captain that hired the worker. A pane that is itself
   an open worker may not close anyone, and a different herdr pane must add
   `--steal`. A plain terminal outside herdr counts as the human and passes.
   `close` also refuses when the card points at the caller's own pane or tab.
2. The worker's last status line must be `done:` or `failed:`. Otherwise the
   work may be unfinished and unsaved; `close` prints the last line and points
   to `log`. It also refuses when the captain sent a `say` after that final
   line, because the worker may be mid-follow-up.
3. For a `--tree` worker, `git status --porcelain` in the worker's copy must be
   empty, even after a `done:` line, because the pooled copy is wiped on
   return. If git itself fails, `close` stops too, because it cannot tell
   whether the copy is clean.
4. The recorded pane must still sit in the same tab, the same window and the
   same working directory, as reported by `herdr pane get`. If any of that
   changed, another session has taken the pane, and `close` leaves it alone.
   `close` also refuses the last tab of a window.
5. After `herdr tab close`, the card is marked closed only once `herdr pane get`
   answers with the error code `pane_not_found`. Any other answer leaves the
   card intact.
6. For a `--tree` worker, the copy is checked a second time after the pane is
   confirmed gone, so edits the worker slipped in during closing are caught
   while the files are still on disk. Only then is the copy returned to
   treehouse, and `close` reports how many new commits the branch holds with a
   `git merge` hint. If the return fails, the card stays open and running
   `close` again retries just that part.

`--force` skips check 2 and both copy checks (check 3 and the second look in
check 6), so uncommitted changes are wiped with the copy. Checks 1, 4 and 5,
the return of the copy to the pool and the commit count always run.

## What you need

- Windows and Node.js 16 or newer. The tool is one file with no dependencies
  and no `package.json`.
- [herdr](https://herdr.dev), the tab and pane manager the captain runs inside.
- [Claude Code](https://claude.com/claude-code), the agent CLI the worker runs.
- Optional: treehouse, the companion tool behind `--tree`. It is not published
  yet; without it, workers share the captain's folder.

## Run it

Every command runs as `node bin\teammate.mjs <command>` or through
`bin\teammate.cmd`. The seven commands and their options are listed below.

```
new "<task>" [--tree] [--cwd <path>] [--file <brief.md>] [--model <name>] [--label <text>]
list [--all] [--json]
check [--json]
wait [--every S] [--timeout S]
log <id> [--lines N] [--tail]
say <id> "<text>" [--steal]
close <id> [--force] [--steal]
```

Exit codes are meant for scripts and hooks. `check` returns 0 when all workers
are quiet, 1 when a worker needs the captain, 2 when teammate itself refused
or hit an error. `wait` returns 1 when a worker needs the captain, 0 when no
workers are left, 3 when the timeout passed with nothing to report.

`--every` defaults to 15 seconds (minimum 3); `--timeout 0`, the default,
waits forever. `log` prints the worker's status lines and then the last
60 lines of its screen; `--tail` prints the status lines only.

## What teammate does not do

- teammate does not restrict the worker. It is started with
  `claude --dangerously-skip-permissions`, so it acts without asking anyone.
  The brief's rules and the `--cwd` you chose are its only limits.
- teammate makes no network calls. It only runs the herdr command (which
  reaches the herdr server over a local socket), git and treehouse, and it
  reads and writes files under `state/`, plus the brief you pass with
  `--file`.
- `state/` is excluded from git by `.gitignore`. A card records the names of
  the environment variables handed to the tab, never their values.
- teammate does not merge branches. After a `--tree` worker closes, it prints
  the `git merge` command and leaves the decision to the captain.
- teammate never finds a pane by its label. Every action targets the pane, tab
  and window ids that herdr itself reported.

## Repository layout

```
bin/teammate.mjs    the whole tool (seven commands, the herdr and treehouse calls, the brief template)
bin/teammate.cmd    Windows wrapper that runs teammate.mjs with the same arguments
docs/teammate.md    field guide with every command, sample output, the card schema and the refusals
docs/herdr-api.md   notes on the herdr CLI surface this tool builds on, verified against herdr 2026.08
state/teammates/    created at first use and ignored by git; holds <id>.json, <id>.brief.md, <id>.status
```

## Related project

[sheepdog](https://github.com/patraianton/sheepdog) is a live kanban board of
every agent session running in herdr. It shows one card per session, in lanes
the operator files by hand, and marks the sessions that are waiting on that
one human operator. teammate grew out of it and reads herdr the same way. The
herdr session had 44 windows open when [docs/herdr-api.md](docs/herdr-api.md)
was written.

## License

MIT, see `LICENSE`.
