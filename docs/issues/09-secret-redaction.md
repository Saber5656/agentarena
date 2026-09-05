# Title

Secret redaction filter for events, eval output, and diffs

## Summary

Implement the best-effort redaction module applied at the engine emit boundary and to eval tails and diff text before persistence/broadcast: masks parent-env secret values and a built-in secret-shaped pattern list, per DESIGN.md §11.4.

## Context

Agent transcripts routinely echo environment variables and file contents; exports leave the machine (threat T3). Redaction is defense-in-depth, explicitly documented as best-effort — it must be deterministic, fast, and never crash the pipeline.

## Scope

`packages/core/src/redaction/`:

- `redactor.ts`: `createRedactor({env, extraPatterns}: {env: NodeJS.ProcessEnv, extraPatterns: string[]}): Redactor`.
  - Env rule: collect values of env vars whose **name** matches `/(KEY|TOKEN|SECRET|PASSWORD|CREDENTIAL)/i` and whose value length ≥ 8; replace every occurrence with `[REDACTED:<VAR_NAME>]`. Longest values replaced first (substring shadowing).
  - Pattern rules (built-in, case-sensitive): `sk-[A-Za-z0-9_-]{16,}`, `github_pat_[A-Za-z0-9_]{20,}`, `gh[pousr]_[A-Za-z0-9]{20,}`, `AKIA[0-9A-Z]{16}`, `xox[abprs]-[A-Za-z0-9-]{10,}`, `eyJ[A-Za-z0-9_-]{10,}\.[A-Za-z0-9_-]{10,}\.[A-Za-z0-9_-]{5,}` → replacement `[REDACTED:pattern]`.
  - `extraPatterns` from config compiled with a 100 ms per-pattern compile guard; invalid regex → warning, skipped.
  - `redactText(s: string): string`; `redactEvent(e: AdapterEmittedEvent): AdapterEmittedEvent` — walks every string field of the payload (deep, including `origin` stringified fields) via a generic deep-string-map helper; returns same reference if untouched.
- Performance guard: single pass per rule using precompiled global regexes; env values matched via literal `String.split/join` (no regex escaping bugs).

## Detailed Requirements

1. Pure functions; no IO; never throws (malformed events pass through unchanged — schema enforcement happens elsewhere).
2. Deterministic: same input → same output (no randomness in replacements).
3. Replacement preserves surrounding text exactly; overlapping matches resolved left-to-right after env-value replacements.
4. The redactor is constructed once per process start (engine holds it); env snapshot taken at construction, documented in JSDoc.
5. Export the built-in pattern list as a named constant (SECURITY.md issue 35 references it).

## Acceptance Criteria

- [ ] Unit tests: env `MY_API_KEY=supersecret123` → "supersecret123" in message text, tool output, nested origin string, diff text all become `[REDACTED:MY_API_KEY]`.
- [ ] Short env value (`PIN=1234`, name matches KEY? it doesn't — use `SHORT_KEY=1234`) is NOT redacted (length rule).
- [ ] Each built-in pattern: one positive and one near-miss negative case (e.g. `sk-short` untouched).
- [ ] JWT-shaped string in a diff is redacted.
- [ ] Idempotent: redacting twice equals redacting once.
- [ ] 1 MB text with no matches processes in < 100 ms on CI (soft perf test with generous bound).
- [ ] Invalid `extraPatterns` entry produces a warning and does not break other rules.

## Validation

`pnpm --filter @agentarena/core test`.

## Dependencies

01, 03, 04 (config `redaction.extraPatterns`).

## Non-goals

Guaranteeing zero leakage (documented limitation), scanning binary diff hunks (text-level only), token/config scrubbing of the arena's own token (never enters event paths by design).

## Design References

DESIGN.md §11.2 T3, §11.4; issue 35 (SECURITY.md documents the limitation).
