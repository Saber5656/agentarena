# Title

Claude Code adapter: spawn `claude -p --output-format stream-json` and translate its stream

## Summary

Implement the `claude-code` adapter per DESIGN.md §7.4: spawn `claude -p` with stream-json output and bypassPermissions inside the worktree, translate `system`/`assistant`/`user`/`result` messages into `AgentEvent`s (synthesizing `file_change` from Edit/Write tool uses), capture cost from `result.total_cost_usd`, and probe availability.

## Context

Flags verified against Claude Code 2.1.172 (docs/research/cli-agent-interfaces.md §2). Critical constraints from research: `--bare` must NOT be used (breaks OAuth), stream-json requires `--verbose`, and in `-p` mode a non-permitted tool call aborts the run — hence `bypassPermissions` as the documented v1 posture (ADR-002 consequence: weakest isolation, must stay visible in docs).

## Scope

- `packages/core/src/adapters/claude/claudeAdapter.ts`:
  - Command assembly exactly:
    `<command> -p --output-format stream-json --verbose --permission-mode bypassPermissions --no-session-persistence [--model <model>] [--max-budget-usd <maxBudgetUsd>] [--setting-sources <settingSources>] [--settings <settingsJson>] [...extraArgs]`
    prompt via stdin; `cwd = workspaceDir`; sanitized env.
  - Forbidden flags assertion: constructor throws if `extraArgs` contains `--bare` or `--dangerously-skip-permissions` (config-level foot-gun guard, DESIGN.md §11.3).
  - `probe()`: `<command> --version`.
- `packages/core/src/adapters/claude/parser.ts`: `parseClaudeLine(line): AdapterEmittedEvent[]` per the DESIGN.md §7.4 table:
  - `system/init` → `status(running)` + `metrics{model}`; `system/api_retry` → `error(warn, source:"agent")`; other `system` subtypes → `raw`.
  - `assistant` message content blocks: `text` → `message`; `thinking` → `reasoning`; `tool_use` → `tool_call` (map `Bash` input `{command}` to `name:"shell"`; other tools keep their name and full input). For `tool_use` with name `Edit|Write|MultiEdit|NotebookEdit`, additionally synthesize `file_change` (`kind`: Write→created-or-modified: use `modified` unless input signals creation; document heuristic in code) with workspace-relative path (absolute inputs relativized against `workspaceDir`; outside-workspace paths kept absolute and flagged in the event `origin`).
  - `user` message `tool_result` blocks → `tool_result` (stringify content; `is_error` → append to output).
  - `result` → `metrics{costUsd: total_cost_usd, tokens from usage if present}`; adapter records summary text for the engine (`RunOutcome` carries no text — store nothing; the engine's `result` event uses last `message` — see issue 15 contract note).
  - Unknown lines → `raw`.
- **Fixture capture task**: real `claude -p` session fixture (scrubbed) as `fixtures/session-basic.jsonl` + hand-written `fixtures/edge-cases.jsonl`; update research §2 UNVERIFIED items (exact `result` fields on 2.1.x).
- Register in the registry.

## Detailed Requirements

1. Config keys per DESIGN.md §7.4/§15; `settingsJson` passed as a single `--settings` argument verbatim (documented user-owned risk).
2. Parser pure/total; no throw on any input line.
3. Outcome mapping identical to issue 11 requirement 2.
4. No `--include-partial-messages` (event volume, DESIGN.md §7.4).
5. Do not emit `result` events (engine-owned).

## Acceptance Criteria

- [ ] Parser unit tests over the captured fixture: init → status+metrics(model); text block → message; Bash tool_use → `tool_call{name:"shell", input.command}`; Edit tool_use → tool_call + synthesized `file_change` with relative path; tool_result → `tool_result`; final `result` → metrics with `costUsd` > 0.
- [ ] Edge cases: `api_retry` → `error(warn)`; unknown subtype → raw; malformed line → raw.
- [ ] Command-assembly test: exact flags; `--max-budget-usd` present only when configured; forbidden `extraArgs` rejected with a clear error.
- [ ] Env-gated integration test (`AGENTARENA_TEST_CLAUDE=1`): trivial task on scratch repo completes with non-empty diff and a `metrics` event carrying cost.
- [ ] research doc §2 updated with confirmed `result` field names and Claude Code version.

## Validation

`pnpm --filter @agentarena/core test`; gated integration run evidence in the PR.

## Dependencies

01–04, 06, 08, 03.

## Non-goals

Claude native-sandbox enablement research beyond config passthrough (tracked as known unknown), partial-message streaming, MCP configuration.

## Design References

DESIGN.md §7.1, §7.2, §7.4, §11.3; ADR-002 (risk note), ADR-004; docs/research/cli-agent-interfaces.md §2.
