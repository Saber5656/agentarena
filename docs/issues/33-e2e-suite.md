# Title

End-to-end suite: full mock-match pipeline assertions and Playwright browser flows

## Summary

Add the two top-of-pyramid test layers from DESIGN.md §17: a node-level E2E that drives a real server + engine + mock adapters through the public API/WS and asserts on-disk artifacts, and a Playwright chromium suite covering spectate → results → vote → replay → export-open-from-file.

## Context

Unit/integration tests land inside earlier issues; this issue proves the assembled product works end-to-end without API keys (CI-safe) and guards regressions across the seams (engine↔server↔UI↔export).

## Scope

- `e2e/` workspace folder (own package `@agentarena/e2e`, private):
  - `api.e2e.test.ts` (vitest): boot the real server (built packages, tmp data dir, scratch git repo) → `POST /api/matches` (2 mock contestants, 1 passing + 1 failing check) → subscribe WS → assert protocol sequence (hello → backfill → live → match frames to terminal) → assert storage tree exists and validates (match.json/events.jsonl/diff.patch/eval.json/verdict.json all schema-checked) → vote via API → replay bundle completeness (event-count equality) → export.html bytes start with doctype.
  - `cli.e2e.test.ts`: `agentarena run` (mock agents) exit codes + `--json` schema; `agentarena export` file creation. (Overlaps issue 32's tests deliberately at a coarser grain — built-artifact level.)
- Playwright (`e2e/playwright/`, chromium only):
  - `spectate.spec.ts`: launch `agentarena demo --no-open` on a random port; open launch URL → lanes render 3 contestants; events stream (await specific mock message text); match finishes → results tabs; scoreboard rows = 3; open Diffs tab → file rows; vote → saved indicator.
  - `replay.spec.ts`: open `/m/:id/replay` → play ×8 → lane content appears; seek to end → terminal chips.
  - `export.spec.ts`: download export via button (accept privacy modal) → open the downloaded file via `file://` in a new page → title + lanes render; page emits zero network requests (`page.on('request')` allowlist: the file URL itself only).
  - CI wiring: `.github/workflows/ci.yml` gains a `e2e` job (ubuntu, after build): install chromium (`playwright install --with-deps chromium`, pinned version), run both layers; artifacts (traces on failure) uploaded.
- Flake policy: retries 1 for Playwright, 0 for node E2E; every wait is condition-based (no bare sleeps; helper `waitFor`).

## Detailed Requirements

1. No real agent CLIs and no network beyond localhost in any E2E path (grep-level guard in test setup asserting mock-only adapter ids).
2. Total suite budget: node E2E < 90 s, Playwright < 4 min on CI (demo fixtures at high speed).
3. Tests run against BUILT packages (`pnpm build` prerequisite) — catches bundling breaks.
4. Tmp data dirs per test file; cleanup on pass, retained on failure (path printed).
5. Port allocation: ephemeral (0 → read actual) to allow parallel CI shards.

## Acceptance Criteria

- [ ] All specs above implemented and green locally (macOS) and in CI (ubuntu).
- [ ] Deliberate-break canaries verified once during development (documented in PR): killing redaction wiring fails api.e2e (planted fake secret asserted redacted in events.jsonl); breaking WS backfill order fails the protocol assertion; breaking export escaping fails export.spec (crafted `</script>` message in a mock fixture — add such a step to one bundled fixture in coordination with issue 10).
- [ ] CI uploads Playwright traces on failure.
- [ ] `pnpm e2e` root script runs the node layer; `pnpm e2e:browser` the Playwright layer.

## Validation

CI run on the PR (both jobs green); reviewer replays the demo flow locally.

## Dependencies

All of 01–32 (final integration gate); specifically consumes 10 (fixtures incl. the XSS-canary step), 32 (demo/run).

## Non-goals

Real-CLI matrix testing (env-gated per-adapter tests live in 11–13), performance benchmarking, cross-browser coverage (chromium only v1).

## Design References

DESIGN.md §17 (E2E rows + fixture policy), §10.3, §13, §16 F15.
