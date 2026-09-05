# Title

Match data read API: event paging, diff text, eval results, replay bundle

## Summary

Implement the read-only data endpoints used by replay, the results view, and the exporter: paged events, raw diff text, eval results, and the assembled `ReplayBundle`, per DESIGN.md §10.2 and §13.

## Context

Live views use WS (20); everything after the fact reads through these routes. The `ReplayBundle` assembly implemented here (in core) is reused verbatim by the HTML exporter (31) and the CLI.

## Scope

- `packages/core/src/replay/bundle.ts`: `assembleReplayBundle({store, matchId, appVersion}): Promise<ReplayBundle>` per DESIGN.md §13: match record, taskMarkdown, verdict, and per contestant `{record, events (full log via readEvents pages), diff?, eval?}`; zod-validate the result before returning (`ReplayBundleSchema` added to shared); typed `conflict` error when match not terminal.
- `packages/server/src/routes/matchData.ts`:
  - `GET /api/matches/:id/contestants/:cid/events?after=0&limit=1000` → `{events, nextAfter, done}` (limit max 5000; after ≥ 0; cid must belong to the match → 404 otherwise).
  - `GET /api/matches/:id/contestants/:cid/diff` → `text/plain; charset=utf-8` patch body; `X-Agentarena-Truncated: 1` header when the truncation marker exists; 404 when absent.
  - `GET /api/matches/:id/eval` → `{evals: Record<cid, EvalResult>}` (missing files → key absent).
  - `GET /api/matches/:id/replay` → ReplayBundle JSON; 409 while non-terminal.
- Shared `ReplayBundleSchema` in `@agentarena/shared` (exact §13 shape).

## Detailed Requirements

1. Events endpoint streams from storage without loading whole logs (issue 05 reader semantics preserved).
2. Diff route sets `Content-Disposition: inline; filename="<matchId>-<cid>.patch"`.
3. Bundle assembly enforces a soft size guard: if serialized bundle > 50 MB → still return but include `truncatedNote` on the bundle? NO — keep schema clean: log a server warning only; the exporter (31) owns user-facing size warnings. (Document this in code.)
4. All params id-validated before path use (§11.2 T7).
5. No mutation anywhere in this issue.

## Acceptance Criteria

- [ ] Inject tests on a finished mock match: events paging walks the full log exactly once (union of pages = seq 1..n, `done` on last); `limit=9999` clamped to 5000.
- [ ] `cid` from another match → 404; traversal-shaped cid → 404.
- [ ] Diff route returns the exact patch bytes written by the engine; truncated fixture sets the header.
- [ ] Eval route returns per-contestant results matching disk; contestant without eval → key absent.
- [ ] Replay route: running match → 409; finished → bundle validates against `ReplayBundleSchema`, contains all events (count equality with storage), diffs, evals, verdict.
- [ ] Bundle assembly unit test in core (no HTTP): terminal guard, event completeness.

## Validation

`pnpm --filter @agentarena/core test` + `pnpm --filter @agentarena/server test`.

## Dependencies

02, 03, 05, 15, 17, 18, 19 (tests create matches through the API).

## Non-goals

HTML export assembly (31), UI consumption (28–30), compression.

## Design References

DESIGN.md §10.2 (read rows), §13, §11.2 T7.
