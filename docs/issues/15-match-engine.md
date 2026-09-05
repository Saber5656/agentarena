# Title

Match engine: orchestration state machine, seq/redaction pipeline, timeouts, cancellation

## Summary

Implement the in-process `MatchEngine` in `@agentarena/core` that creates and runs matches end-to-end: validates input, snapshots the task, provisions workspaces, runs contestants in parallel through their adapters, assigns `seq`/`ts`, applies caps + redaction, persists everything through the match store, captures diffs, triggers evaluation, computes the scoreboard, and enforces every state transition of DESIGN.md §5.3.

## Context

This is the product's core loop (DESIGN.md §4.3). Everything else (server, CLI) is a thin shell around `MatchEngine`.

## Scope

`packages/core/src/engine/`:

- `engine.ts`: `createMatchEngine({config, dataDir, registry, store, redactor})` exposing:
  - `createAndStart(input: CreateMatchInput): Promise<MatchRecord>` — input `{repoPath, baseRef?, taskMarkdown, contestants: [{adapterId, model?}], options?}`; steps:
    1. Parse task frontmatter (`yaml` package, safe schema; DESIGN.md §9.1/§11.3 bounds), validate 2–4 contestants (F1: probe each; all-unavailable → validation error; individually unavailable contestants are admitted but fail immediately at start with `error(fatal)`), inspect repo + `resolveBase` (issue 07).
    2. Build `MatchRecord` (ids issue 02; `options` merged with config defaults), `store.createMatch`, status `created→preparing`.
    3. Workspaces per contestant (issue 07); failure of any workspace → match `failed` with `error` (infrastructure rule §5.3), cleanup started ones.
    4. `running`: launch contestants with concurrency cap `options.maxParallel`; per contestant: `status(preparing→running)` events, adapter `run()` with a `ContestantContext` whose `emit` goes through the pipeline below and whose `timeoutSignal` fires at `options.timeoutSec`.
    5. On adapter return: synthesize the `result` event (`summary` = text of the contestant's last `message` event head 500 chars, else "no summary"; `outcome`/`exitCode`/`durationMs`), transition contestant status per outcome, capture diff (issue 07) → `store.putDiff` + `diffStats` on the record.
    6. Evaluation (issue 16) per contestant on its terminal state (except canceled → skipped results); match `running→evaluating` when all contestants terminal; after all evals: scoreboard (issue 17) → `verdict.json`; `evaluating→finished`.
  - `cancel(matchId)`: idempotent; aborts all contestant signals, waits for adapters, marks non-terminal contestants `canceled`, match `→canceled`, evals recorded `skipped`, scoreboard still computed.
  - `getLiveMatch(matchId)` and `subscribe(matchId, listener)` / `unsubscribe` — listener receives `{kind:'event', contestantId, event}` and `{kind:'match', match}` (feeds WS, issue 20).
  - `startupRecovery()` — run store recovery (issue 05) + workspace orphan cleanup (issue 07).
- Event pipeline (single function, order fixed): `AdapterEmittedEvent` → `applyEventCaps` (issue 03) → `redactEvent` (issue 09) → assign `v:1, seq (per-contestant counter), ts (now ISO)` → append to event log (issue 05) → update `lastSeq`/`metrics` on the in-memory record → notify subscribers. Event-count cap (§6.3): past 50 000, drop non-critical types and emit one `error(warn)`.
- Transition enforcement: every status change goes through `assertTransition` (issue 02) then a single `updateMatch` (ordering rule ADR-003: events appended before the summarizing match.json write) then a `{kind:'match'}` notification.

## Detailed Requirements

1. All contestant work runs in the one Node process; no worker threads v1.
2. Timeout timer per contestant starts at `running`; firing aborts the signal → adapter must return; engine enforces a hard 15 s follow-up (process runner grace) before forcing status `timeout`.
3. Engine is storage-write serialized per match (issue 05 queue) — no interleaved `match.json` corruption.
4. In-flight matches held in a `Map<matchId, LiveMatch>`; `maxConcurrentMatches` (config) enforced at `createAndStart` with typed `conflict` error.
5. Deleting a match (store) must be preceded by `cancel` + workspace removal — expose `deleteMatch(matchId, {force})` orchestrating this (409-equivalent error if running and not force).
6. Every error path ends in a valid terminal state — no match may remain non-terminal after `createAndStart`'s promise chain settles (except genuine `running` in-progress).

## Acceptance Criteria

- [ ] Integration test (mock adapters, scratch repo): 3 contestants → match reaches `finished`; per contestant: status events in order `preparing, running` + terminal; exactly one `result` event (last); `seq` strictly monotonic from 1; events.jsonl on disk validates line-by-line.
- [ ] Diff captured for a mock that writes files: `diffStats.filesChanged` ≥ 1 and diff.patch exists; contestant with no writes → empty patch, zero stats.
- [ ] Timeout test: mock fixture longer than `timeoutSec: 1` → contestant `timeout`, match still `finished`, eval ran.
- [ ] Cancel test: cancel mid-run → all contestants `canceled`, match `canceled`, checks `skipped`, scoreboard exists.
- [ ] Unavailable adapter (placeholder probe false) among 3 → that contestant `failed` immediately with `error(fatal)`, others complete, match `finished` (F1).
- [ ] Workspace-creation failure (invalid baseSha injected) → match `failed` with `error` set; no orphan worktrees remain.
- [ ] Redaction wired: mock fixture echoing a fake `FAKE_API_KEY` env value → persisted event text contains `[REDACTED:FAKE_API_KEY]`.
- [ ] Transition guard: forcing an illegal transition in a unit test throws `InvalidTransitionError`.
- [ ] `maxConcurrentMatches: 1`: second `createAndStart` while one runs → typed conflict error.

## Validation

`pnpm --filter @agentarena/core test` — this issue owns the core integration suite (DESIGN.md §17 row 2).

## Dependencies

02, 03, 04, 05, 06, 07, 08, 09, 10 (mock for tests). Runs before/parallel with 11–14 (real adapters slot in via the registry).

## Non-goals

HTTP/WS exposure (18–20), evaluation internals (16), scoreboard math (17 — engine only invokes), replay/export.

## Design References

DESIGN.md §4.3, §5.3, §6.3, §7.1, §9.3, §16 F1–F9; ADR-002, ADR-003, ADR-004.
