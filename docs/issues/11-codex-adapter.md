# Title

Codex CLI adapter: spawn `codex exec --json` and translate its JSONL events

## Summary

Implement the `codex` adapter per DESIGN.md §7.3: spawn `codex exec` with the fixed flag set (JSON output, workspace-write sandbox, clean profile, ephemeral), parse the JSONL event stream into `AgentEvent`s using recorded fixtures, and probe availability.

## Context

Codex is a flagship contestant. Flags are verified against codex-cli 0.141.0 (docs/research/cli-agent-interfaces.md §1); exact per-item payload fields are UNVERIFIED and must be pinned by capturing real fixtures during this issue.

## Scope

- `packages/core/src/adapters/codex/codexAdapter.ts`:
  - Command assembly exactly:
    `<command> exec --json -C <workspaceDir> --sandbox <sandbox> [--ignore-user-config --ignore-rules](cleanProfile) --ephemeral --color never [-m <model>] [...extraArgs] -`
    with the prompt written to stdin (arg `-`), via the process runner (issue 06) with `sanitizedChildEnv`, `cwd = workspaceDir`.
  - `probe()`: `probeCliVersion` on `<command> --version` (issue 08 helper); add problem hint "run `codex login` if commands fail with auth errors" (auth not cheaply probeable — research §1).
- `packages/core/src/adapters/codex/parser.ts`: `parseCodexLine(line: string): AdapterEmittedEvent[]` implementing the mapping table in DESIGN.md §7.3:
  - `thread.started` → `status(running)`; `turn.completed` → `metrics` from `usage` (`input_tokens→inputTokens`, `cached_input_tokens→cachedInputTokens`, `output_tokens→outputTokens`); `turn.failed` → `error(fatal)`.
  - `item.started`/`item.completed` dispatch on item type: `agent_message`→`message`, `reasoning`→`reasoning`, `command_execution`→`tool_call(name:"shell")` on started / `tool_result` on completed, `file_change`→ one `file_change` per file, `mcp_tool_call`/`web_search`→`tool_call`+`tool_result`, `plan_update`→`message` (with origin).
  - Every event carries `payload.origin` = the parsed line (cap handled by shared caps).
  - Any unrecognized/unparseable line → `raw(stdout)` (ADR-004 rule 3). stderr lines → `raw(stderr)`.
- **Fixture capture task (part of this issue)**: run a real `codex exec --json` session on a scratch repo with a trivial task; scrub secrets; commit as `fixtures/session-basic.jsonl` plus a hand-written `fixtures/edge-cases.jsonl` (unknown event type, malformed line, huge output line). Update docs/research/cli-agent-interfaces.md §1 UNVERIFIED items with the confirmed field names.
- Register in the adapter registry (replacing the issue 08 placeholder).

## Detailed Requirements

1. Config handling per DESIGN.md §7.3 keys; `sandbox` restricted by config schema (issue 04) — adapter must still assert it defensively.
2. Adapter returns `RunOutcome` from process exit: 0→`completed`, non-zero→`failed`, runner `timedOut`→`timeout`, `aborted`→`canceled`.
3. Parser is pure and total: never throws on any string input.
4. Do not emit a `result` event (engine synthesizes it).
5. `cliVersion` from probe surfaced to the engine via `ProbeResult.version`.

## Acceptance Criteria

- [ ] Parser unit tests over `session-basic.jsonl`: every line maps per the table; token counts from `turn.completed` land in a `metrics` event; command execution produces paired `tool_call`/`tool_result` with the command string in `input.command`.
- [ ] `edge-cases.jsonl`: unknown type → `raw`; malformed JSON → `raw`; no throw for any line.
- [ ] Command-assembly unit test (spawn mocked): flags exactly as specified for `cleanProfile: true/false`, model set/unset; prompt delivered via stdin; env has no `AGENTARENA_*`.
- [ ] Integration test (skipped when `codex` binary or auth is unavailable; env-gated `AGENTARENA_TEST_CODEX=1`): real run on a scratch repo task "create HELLO.md containing hello" → contestant events include ≥1 `message`, process completes, workspace diff non-empty.
- [ ] research doc updated: UNVERIFIED items resolved or narrowed, with codex version noted.

## Validation

`pnpm --filter @agentarena/core test`; the env-gated integration test run once locally by the implementer with output pasted into the PR.

## Dependencies

01–04, 06, 08; 03 (event schema). Engine not required (adapter testable standalone).

## Non-goals

Cost estimation (no price table v1), resume/sessions, approval flows, non-clean-profile fairness policy changes (fixed by config).

## Design References

DESIGN.md §7.1–§7.3, §16 F1–F5; ADR-002, ADR-004; docs/research/cli-agent-interfaces.md §1.
