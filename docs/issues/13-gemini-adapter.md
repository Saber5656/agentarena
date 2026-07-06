# Title

Gemini CLI adapter: headless stream-json run with hard raw fallback

## Summary

Implement the `gemini` adapter per DESIGN.md §7.5: spawn the Gemini CLI in headless mode with `--output-format stream-json --approval-mode yolo`, map what the fixtures confirm, fall back to `raw` for everything else, synthesize `metrics` from the final stats object, and probe availability (including the not-installed case).

## Context

Gemini CLI is NOT installed on the reference machine; the stream-json event schema is UNVERIFIED (docs/research/cli-agent-interfaces.md §3). This issue therefore starts by installing the CLI, capturing fixtures, and pinning the parser to observed reality. The adapter must stay useful (raw lane + final metrics) even if the schema shifts again.

## Scope

- **Fixture-first step**: install Gemini CLI (document the installed version), run a real headless session on a scratch repo (`--output-format stream-json --approval-mode yolo`), scrub, commit as `fixtures/session-basic.jsonl`. Verify and record in docs/research/cli-agent-interfaces.md §3: stdin-vs`-p` prompt delivery behavior, exact stream-json event shapes, whether a final stats/result object appears on stdout, `--sandbox` behavior on macOS. Resolve or re-scope the UNVERIFIED markers.
- `packages/core/src/adapters/gemini/geminiAdapter.ts`:
  - Command assembly: `<command> --output-format stream-json --approval-mode yolo [-m <model>] [--sandbox](config) [...extraArgs]`, prompt via stdin if verified reliable, else `-p <prompt>` (decision recorded in code comment + research doc).
  - `probe()`: `<command> --version`; ENOENT → `available: false` with install hint (`npm install -g @google/gemini-cli` or the then-current official channel — verify during implementation).
- `packages/core/src/adapters/gemini/parser.ts`: `parseGeminiLine(line): AdapterEmittedEvent[]`:
  - Map confirmed event kinds to `message` / `reasoning` / `tool_call` / `tool_result` / `file_change` per observed schema (table to be added to research §3 as part of this issue).
  - Final stats object (per research §3: `stats.models.*.tokens`, `files.totalLinesAdded/Removed`) → `metrics` (+ diff-line info left to arena's own diffStats; do not map).
  - `error` object → `error(fatal)`; anything else → `raw`.
- Register in the registry.

## Detailed Requirements

1. Parser total/pure; the adapter must produce a watchable lane even in worst case (all `raw` + final `metrics`).
2. Outcome mapping identical to issues 11/12.
3. Config keys per DESIGN.md §7.5/§15; `sandbox: true` appends `-s` only after macOS behavior is verified (else config validation warns "unverified on this platform" — implement the warning).
4. No `result` emission (engine-owned).
5. Update the ISSUE_PLAN known-unknowns list status lines for gemini items in the same PR.

## Acceptance Criteria

- [ ] Committed real fixture + parser tests: every fixture line maps or falls back deliberately (test asserts zero unhandled throws and enumerates mapped event counts).
- [ ] Not-installed probe path: `available: false`, actionable install hint, < 6 s.
- [ ] Command-assembly test covering model/sandbox/extraArgs permutations.
- [ ] Env-gated integration test (`AGENTARENA_TEST_GEMINI=1`): trivial task produces a non-empty diff and ≥ 1 `message` or documented raw-only degradation.
- [ ] docs/research §3 rewritten from UNVERIFIED to verified facts (version-stamped); ISSUE_PLAN known-unknowns updated.

## Validation

`pnpm --filter @agentarena/core test`; gated integration evidence in the PR; research diff reviewed.

## Dependencies

01–04, 06, 08, 03.

## Non-goals

Gemini sandbox container mode as default (config-gated only), non-yolo approval flows, Google auth automation (user logs in themselves; probe hints only).

## Design References

DESIGN.md §7.1, §7.2, §7.5, §16 F1–F3; ADR-004; docs/research/cli-agent-interfaces.md §3.
