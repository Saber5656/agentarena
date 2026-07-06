# Title

Process runner: group spawn, kill-tree, timeout, line streaming, output caps

## Summary

Implement the single child-process utility used by all CLI adapters and the evaluation runner: detached-group spawn, UTF-8 line-split stdout/stderr streaming with caps, AbortSignal-driven SIGTERM→SIGKILL tree kill, and wall-clock timeout, per DESIGN.md §7.2.

## Context

Adapters (issues 11–13), the eval runner (16), and workspace git calls (07) all need identical, reliable process control. Agents are semi-trusted (DESIGN.md §11.1) so kill must reach grandchildren.

## Scope

`packages/core/src/proc/runner.ts`:

```ts
interface RunProcessOptions {
  cmd: string; args: string[]; cwd: string;
  env: NodeJS.ProcessEnv;
  stdin?: string;                       // written then FD closed
  timeoutMs?: number;
  signal?: AbortSignal;                 // external cancel
  onStdoutLine?: (line: string) => void;
  onStderrLine?: (line: string) => void;
  maxLineBytes?: number;                // default 1 MiB; longer lines are split at the cap
  gracePeriodMs?: number;               // default 10_000
}
interface RunProcessResult {
  exitCode: number | null; signalName: string | null;
  timedOut: boolean; aborted: boolean; durationMs: number;
  spawnError?: string;                  // ENOENT etc.
}
runProcess(opts): Promise<RunProcessResult>
```

- Spawn with `child_process.spawn`, `detached: true`, `stdio: ['pipe','pipe','pipe']`.
- Kill semantics: on timeout/abort → `process.kill(-pid, 'SIGTERM')`; after `gracePeriodMs` → `process.kill(-pid, 'SIGKILL')`; both wrapped (ESRCH tolerated). Resolve only after `close` (all stdio drained).
- Line splitter: incremental UTF-8 decode (`TextDecoder` streaming) buffering to `\n`; flush remainder on close; enforce `maxLineBytes` by emitting capped chunks.
- Env hygiene helper `sanitizedChildEnv(parentEnv)`: copy minus `AGENTARENA_*` keys, plus `NO_COLOR: "1"` (DESIGN.md §7.2).
- `sanitizedCheckEnv(parentEnv)` = above + `AGENTARENA: "1"`, `CI: "1"` (DESIGN.md §9.2).

## Detailed Requirements

1. ENOENT and other spawn errors resolve (not reject) with `spawnError` set and `exitCode: null`.
2. `stdin` write handles EPIPE gracefully (child exits before reading).
3. No orphan processes: after kill, the whole group is gone (validated by test below).
4. macOS + Linux only; guard with a clear error on `process.platform === 'win32'`.
5. No external deps (node builtins only).

## Acceptance Criteria

- [ ] Test: run `node -e "console.log('a'); console.error('b')"` → one stdout line "a", one stderr line "b", exitCode 0.
- [ ] Test: spawn a shell that spawns a grandchild sleeping 60s; abort after 100 ms → result `aborted: true`, and polling `process.kill(grandchildPid, 0)` throws ESRCH within 15 s (grandchild pid communicated via stdout).
- [ ] Test: `timeoutMs: 200` on `sleep 5` → `timedOut: true`, duration < 12 s (SIGTERM honored fast; assert well under grace+margin).
- [ ] Test: command emitting a single 3 MiB line → callback receives capped chunks, none exceeding `maxLineBytes`; process completes.
- [ ] Test: nonexistent binary → `spawnError` contains "ENOENT", no throw.
- [ ] Test: `sanitizedChildEnv` drops `AGENTARENA_HOME`, keeps `PATH`, adds `NO_COLOR`.
- [ ] Multibyte boundary: output of a UTF-8 string written in two chunks split mid-codepoint reassembles correctly.

## Validation

`pnpm --filter @agentarena/core test` on macOS and Linux CI (issue 01 matrix).

## Dependencies

01.

## Non-goals

Event translation (adapters), retry logic, PTY support (CLIs run in non-TTY headless modes by design).

## Design References

DESIGN.md §7.2, §9.2, §11.2 T8, §16 F4.
