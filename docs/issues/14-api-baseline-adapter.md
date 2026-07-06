# Title

API baseline adapter: one-shot diff generation via OpenAI-compatible endpoint

## Summary

Implement the `api-baseline` adapter per DESIGN.md §7.6: build a deterministic repo-context prompt, call an OpenAI-compatible chat-completions endpoint once (plus one repair round), extract a unified diff, apply it with `git apply --3way`, and emit synthesized events including token metrics.

## Context

The baseline contestant answers "what does an agent harness add over the raw model?" and proves the adapter interface supports non-CLI participants (product decision Q&A 2026-07-07). The API key is read from an env var at run time and never persisted (DESIGN.md §11.4).

## Scope

`packages/core/src/adapters/baseline/`:

- `context.ts`: `buildContext(workspaceDir, taskPrompt, config)`:
  - File tree via `git ls-files` (cap 400 paths, sorted).
  - README head (first 4 KB) when `README*` exists.
  - Up to `maxContextFiles` (default 30) file contents ≤ 24 KB each; selection order: (1) paths literally mentioned in the task prompt, (2) remaining source files smallest-first; total context budget 192 KB (stop adding when exceeded). Deterministic given the same tree.
- `prompts.ts`: system + user prompt templates (versioned constants): instruct the model to reply with exactly one fenced ```diff block containing a unified diff against the given tree; repair-round template embedding the `git apply` stderr and rejected hunks.
- `diffExtract.ts`: `extractDiff(reply: string): string | null` — first ```diff fence; fallback: whole reply if it starts with `diff --git` or `---`/`+++` pair; else null.
- `client.ts`: `chatComplete({baseUrl, model, apiKeyEnv, messages, temperature?, signal})` using global `fetch`; reads `process.env[apiKeyEnv]` at call time; missing key → typed error "env var <name> not set"; 60 s request timeout tied to `ctx.timeoutSignal`; returns `{text, usage?: {prompt_tokens, completion_tokens}}`; non-2xx → error with status + body head (512 B, redacted).
- `baselineAdapter.ts`: orchestration: `status` → `tool_call(name:"chat.completions", input:{model, baseUrl})` → call → `message(reply)` + `metrics(tokens)` → extract → apply (`git apply --3way --whitespace=nowarn` via process runner, cwd=workspace) → on success `file_change` per file (parse `git apply --numstat` output or re-derive via `git status --porcelain` delta) → outcome `completed`. On extract/apply failure → one repair round (config `maxRepairRounds: 1`) → then `error(fatal)` + outcome `failed`.
  - `probe()`: `available` = env var set (name from config) AND `model` non-empty; problems list the missing piece. No network call in probe.
- Register in the registry.

## Detailed Requirements

1. Metrics: map `usage.prompt_tokens→inputTokens`, `completion_tokens→outputTokens`; `costUsd` omitted (no price table v1).
2. Repair round re-sends: original user prompt + model reply + apply stderr (capped 8 KB).
3. All network and git failures produce `error(fatal)` events with actionable messages; adapter never throws out of `run()`.
4. No streaming (single response); `signal` abort → outcome `canceled`.
5. Binary files: instruct the model not to produce binary patches; apply failures on binary hunks follow the normal failure path.

## Acceptance Criteria

- [ ] Unit tests with a stubbed fetch: happy path emits the exact event sequence above and applies a real diff on a scratch repo (file actually changed, exit `completed`).
- [ ] Malformed reply (no diff) → repair round issued; second malformed → `failed` with `error(fatal)`; exactly 2 requests made.
- [ ] Apply-conflict path: stub returns a diff against stale content → `--3way` failure captured → repair prompt contains stderr → success on round 2 applies cleanly.
- [ ] Missing env key → probe `available: false` naming the var; `run()` fails fast with the same message.
- [ ] Context builder determinism test: same scratch tree twice → byte-identical context; budget test: many files → context ≤ 192 KB and ≤ 400 tree paths.
- [ ] Usage tokens land in a `metrics` event.

## Validation

`pnpm --filter @agentarena/core test` (all network stubbed; no live-API test in CI). Optional env-gated live smoke (`AGENTARENA_TEST_BASELINE=1`).

## Dependencies

01–04, 06, 07 (scratch-repo helpers in tests), 08, 03.

## Non-goals

Multi-turn agent loops, tool use, streaming, provider price tables, Anthropic/Gemini native (non-OpenAI-compatible) API schemas.

## Design References

DESIGN.md §7.6, §11.4; ADR-004; docs/research/prior-art.md (baseline rationale).
