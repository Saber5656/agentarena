# Title

CLI entrypoint: serve, run, ls, export, doctor, demo

## Summary

Implement the `agentarena` command per DESIGN.md §14 with commander: default `serve` (launch server + print/open token URL), headless `run`, `ls`, `export`, `doctor`, and `demo` (temp fixture repo + mock match), with the specified exit codes.

## Context

The CLI is the product's front door and the only place the launch URL (with token) is printed. `run` makes the arena scriptable/CI-usable with mock adapters; `demo` is the zero-config first-run experience.

## Scope

`packages/cli/src/` (package name `agentarena`, `bin: {agentarena: "dist/main.js"}` wiring finalized in issue 34):

- `main.ts`: commander program, global options `--data-dir <path>` (overrides `AGENTARENA_HOME`), `--port <n>`; subcommands below; unhandled errors → stderr one-liner + exit 1 (stack only with `DEBUG=agentarena`).
- `serve` (default when no subcommand): load config (issue 04) → startup recovery (15) → build+start server (18) with web dist path resolution → print exactly:
  `agentarena v<version>\n  ➜ http://127.0.0.1:<port>/?token=<token>` → `open` the URL unless `--no-open` → SIGINT: cancel running matches (engine), close server, exit 0.
- `run --repo <path> --task <file> --agents <spec> [--json] [--keep-workspaces]`:
  - `<spec>` = comma list `adapterId[:model]` (e.g. `codex,claude-code:fable,mock`); validates 2–4.
  - Reads the task file, runs via engine directly (no HTTP). Progress to stderr: one line per contestant status change (`[codex] running…`, `[claude-code] completed in 1m 12s`), checks progress, final scoreboard as an aligned text table to stdout (or `--json`: `{match, verdict}` document).
  - Exit codes: 0 finished; 2 match failed/interrupted; 130 on SIGINT after cancel (DESIGN.md §14 table).
- `ls [--json]`: match summaries table (id, title, status, created, adapters).
- `export <matchId> [-o <file>]`: assemble bundle + HTML (issues 22/31 core functions), default filename `agentarena-<matchId>.html` in cwd; prints the §13 privacy warning to stderr; refuses non-terminal (exit 1, message).
- `doctor`: table of checks — node version ≥ 22, git version ≥ 2.30 and on PATH, data dir writable, configured port free, each adapter `probeAll({fresh: true})` row (id, available, version, problems). Exit 0 all green (adapter unavailability = yellow warning, not failure, unless ALL adapters unavailable); blockers (node/git/dir) → exit 1.
- `demo [--no-open]`: create temp git repo fixture (fixed small TS project: `src/greeting.ts` with a bug, `src/greeting.test.ts`, `package.json` with a `test` script runnable via node — NO network install: test script uses `node --test` so the demo needs no pnpm/npm install; coordinate file set with issue 10 fixtures), create a match with the three bundled mock fixtures (`speed: 4`) and one check (`node --test`), then `serve` + open directly into `/m/<id>`.

## Detailed Requirements

1. No token ever printed except in the serve/demo launch URL line.
2. `run` works without the server package loaded (import surface: core only) — keeps headless lean.
3. All user-facing strings centralized in a `messages.ts` (single place; English).
4. Ctrl-C during `run` cancels the match and still writes storage consistently (engine cancel path).
5. `--data-dir` propagates everywhere (config, storage, workspaces).

## Acceptance Criteria

- [ ] `agentarena demo --no-open` (test: spawn the built CLI) starts, match finishes (mock), `ls` shows it `finished`, exit 0 on SIGINT.
- [ ] `run` with 2 mock agents on a scratch repo + a passing and a failing check: stdout table shows `1/2` per contestant, exit 0; `--json` output validates against shared schemas.
- [ ] `run` with 1 agent → usage error exit ≠ 0 with clear message; unknown adapter → lists valid ids.
- [ ] `export` on the demo match writes a file starting with `<!doctype html>`; non-terminal id → exit 1.
- [ ] `doctor` on a healthy env exits 0 and shows five adapter rows; with `PATH=/nonexistent` git check fails → exit 1.
- [ ] SIGINT during `run` → match `canceled` on disk, exit 130.
- [ ] Launch line format exactly as specified (snapshot).

## Validation

`pnpm --filter agentarena test` (CLI spawned as subprocess against built output in tmp data dirs); manual demo spectating.

## Dependencies

04, 05, 07, 08, 10, 15, 16, 17, 18, 22, 31 (export assembly), 23 (web dist for serve/demo).

## Non-goals

npm packaging/publish (34), shell completions (v2), interactive TUI.

## Design References

DESIGN.md §14, §7.7, §13 (privacy warning), §16 F8.
