# Title

Mock adapter: scripted fixture replay with real file writes

## Summary

Implement the `mock` adapter that replays a scripted fixture of events with configurable timing and performs scripted file operations in its workspace (so diffs and checks are real), plus three bundled demo fixtures. This unlocks UI development, integration tests, e2e, and the `demo` command without any API keys.

## Context

DESIGN.md §7.7. The mock adapter is a first-class v1 feature (demo mode) and the test double for the whole pipeline.

## Scope

- `packages/core/src/adapters/mock/fixtureFormat.ts`: zod schema `MockFixtureSchema`:

```jsonc
{
  "v": 1,
  "name": "refactor-utils",
  "steps": [
    { "delayMs": 400, "event": { "type": "message", "payload": { "text": "Reading the code…", "format": "markdown" } } },
    { "delayMs": 200, "write": { "path": "src/util.ts", "content": "…" } },     // create/overwrite file in workspace
    { "delayMs": 100, "delete": { "path": "src/old.ts" } },
    { "delayMs": 300, "event": { "type": "tool_call", "payload": { "name": "shell", "input": { "command": "pnpm test" } } } }
  ],
  "outcome": { "exitCode": 0, "outcome": "completed", "summary": "Refactored utils." }
}
```

  - Each step has exactly one of `event | write | delete`; `event` payloads validate against `AdapterEmittedEventSchema` (issue 03). `write`/`delete` steps auto-emit the corresponding `file_change` event. Paths must be workspace-relative, `..` rejected.
- `packages/core/src/adapters/mock/mockAdapter.ts`: implements `AgentAdapter` (issue 08): `probe()` always available; `run(ctx)` replays steps honoring `delayMs / speedFactor` (config `speed`, and per-contestant override via adapterConfig), respects `ctx.timeoutSignal` (abort mid-replay → outcome `canceled`), emits final `metrics` + returns `outcome`.
- Bundled fixtures under `packages/core/src/adapters/mock/fixtures/`: `demo-codex-like.json`, `demo-claude-like.json`, `demo-gemini-like.json` — each 30–60 steps telling a plausible story (read files → run tests → edit 2–3 files → re-run tests → summary), with distinct pacing and at least one `reasoning`, several `tool_call`/`tool_result`, `write` steps producing overlapping-but-different diffs on the demo repo fixture (issue 32 defines the demo repo; fixtures here write self-contained new files plus edits to `src/greeting.ts` whose base content is specified inside each fixture header comment for coordination — keep file set: `src/greeting.ts`, `src/greeting.test.ts`, `README.md`).
- `loadFixture(nameOrPath)` resolving bundled names or absolute paths.

## Detailed Requirements

1. Timing uses one cancellable sleep helper; total replay duration with `speed: 4` ≈ sum(delayMs)/4 (±20 %).
2. File writes go through safe path join (workspace-root prefix assertion).
3. The adapter never touches git (diff capture is the engine's job).
4. Fixture validation errors name the step index.
5. Abort between steps takes effect within 50 ms.

## Acceptance Criteria

- [ ] Unit tests: fixture with `write`+`delete` on a scratch dir produces the files/removals and emits `file_change` events in step order; event `seq` is absent (engine-owned).
- [ ] Replay of a 10-step fixture at `speed: 10` completes < 1 s and emits exactly the scripted events + auto events + no `result` (engine synthesizes `result` from `RunOutcome` — see issue 15 contract).
- [ ] Abort at step 3 → outcome `canceled`, no further writes occur.
- [ ] `..` path in a fixture → validation error naming the step.
- [ ] Three bundled fixtures load and validate; each contains ≥ 1 reasoning, ≥ 3 tool_call, ≥ 2 write steps.

## Validation

`pnpm --filter @agentarena/core test`; visual check deferred to issue 26/32 (demo).

## Dependencies

01, 03, 04, 06 (sleep/abort helpers), 08.

## Non-goals

Demo repo creation and `demo` command (issue 32), engine result synthesis (15), UI (26/27).

## Design References

DESIGN.md §7.7, §14 (`demo`), §17 (fixture policy).
