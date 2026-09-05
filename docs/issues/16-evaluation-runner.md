# Title

Evaluation runner: execute task checks per workspace and persist results

## Summary

Implement the check execution subsystem per DESIGN.md §9.2: run each task-defined check command sequentially in a contestant's workspace with sanitized env and timeouts, capture capped+redacted output tails, and persist `eval.json`.

## Context

Checks are the objective half of the verdict (build/test pass-fail). They are user-authored shell commands — trusted by design (§11.2 T4) but resource-bounded and isolated to the workspace cwd.

## Scope

`packages/core/src/eval/runner.ts`:

- `runEval({match, contestant, workspaceDir, redactor, store}): Promise<EvalResult>`:
  - Iterate `match.task.checks` in declared order. For each: `bash -c <run>` via the process runner (issue 06) with `cwd = workspaceDir`, env = `sanitizedCheckEnv` (issue 06: + `AGENTARENA=1`, `CI=1`, `NO_COLOR=1`), timeout `check.timeoutSec ?? options.checkTimeoutSec`.
  - Status mapping: exit 0 → `passed`; non-zero → `failed`; spawn error → `error`; timeout → `timeout` (DESIGN.md §5.4).
  - Output capture: rolling tail buffers per stream (32 KB each), `truncated` flag when overflowed; tails passed through `redactor.redactText` before persistence.
  - Empty `checks` array → `EvalResult` with empty `results` (valid; scoreboard shows 0/0).
  - Skip path: `runEvalSkipped(match, contestant)` producing all-`skipped` results (canceled matches, DESIGN.md §9.2).
  - Persist via `store.putEval`; return the result.
- Concurrency across contestants is the engine's job (issue 15 uses `maxParallel`); this module is single-contestant sequential.
- `evalSummary` computation helper: `{passed, total}` (total excludes `skipped`? No — total = all declared checks; document in JSDoc; skipped counts in total, not in passed).

## Detailed Requirements

1. A check failure does NOT stop subsequent checks (all checks always run; independent signals).
2. Timeout kills the whole process group (runner semantics); `durationMs` recorded from spawn to close.
3. Tails are UTF-8 safe (reuse shared truncation helpers where applicable).
4. No shell profile loading (`bash -c`, not `-lc`) for determinism; PATH inherited from sanitized env. Document that user PATH tools (pnpm, npm) are available because env is inherited-minus-AGENTARENA_*.
5. Module has zero knowledge of HTTP/UI.

## Acceptance Criteria

- [ ] Tests on a scratch dir: checks `[echo ok, exit 3, sleep 5 (timeoutSec 1), /nonexistent-bin]` yield statuses `[passed, failed, timeout, error]`, all four executed, durations sane.
- [ ] stdout of 1 MB → `stdoutTail` exactly ≤ 32 KB, `truncated: true`, tail content is the END of the output.
- [ ] Secret in check output (planted env var value) → redacted in persisted eval.json.
- [ ] `CI=1` and `AGENTARENA=1` visible to the check; `AGENTARENA_HOME` not.
- [ ] Empty checks → `results: []` persisted; summary `{passed: 0, total: 0}`.
- [ ] Skipped path produces all-`skipped` with `exitCode: null, durationMs: 0`.
- [ ] `eval.json` on disk validates against `EvalResultSchema`.

## Validation

`pnpm --filter @agentarena/core test`.

## Dependencies

02, 04, 05, 06, 09.

## Non-goals

Scoreboard aggregation (17), re-running evals via API (v2), sandboxing checks (trusted by design, documented).

## Design References

DESIGN.md §5.4, §9.2, §11.2 T4, §16 F4/F5.
