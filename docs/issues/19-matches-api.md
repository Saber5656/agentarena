# Title

Matches REST API: create/list/get/cancel/delete, repo inspect, adapters probe

## Summary

Implement the match lifecycle REST routes and supporting endpoints on the issue 18 server, delegating all behavior to the engine/store/registry, per the DESIGN.md §10.2 table.

## Context

These routes are the wizard's and CLI's control surface. All request bodies are zod-validated strict; ids re-validated at the route boundary (defense in depth, §11.3).

## Scope

`packages/server/src/routes/`:

- `adapters.ts`: `GET /api/adapters` → `{adapters: [{id, displayName, kind, defaultModel?, available, version?, problems}]}` from `registry.probeAll()` (60 s cache inside registry; issue 08).
- `repo.ts`: `POST /api/repo/inspect` body `{path: string}` → issue 07 `inspectRepo` result; validation: absolute path required; non-existent → 400 `invalid_request` with message; NOT a git repo → 200 with `isGitRepo: false` (wizard renders the hint). Reject paths inside `AGENTARENA_HOME` (400, §11.3).
- `matches.ts`:
  - `POST /api/matches` body `{repoPath, baseRef?, taskMarkdown, contestants: [{adapterId, model?}] (2–4), options?: {timeoutSec?, checkTimeoutSec?, keepWorkspaces?, maxParallel?}}` → `engine.createAndStart`; 201 + MatchRecord. Error mapping: task/frontmatter validation → 400; all-contestants-unavailable → 400 with per-adapter problems; concurrent-match limit → 409 `conflict`; taskMarkdown > 256 KB → 413 `payload_too_large`.
  - `GET /api/matches?limit&offset` → `{matches, total}` (store.listMatches; limit default 50, max 200).
  - `GET /api/matches/:id` → `{match, verdict?}` (verdict when file exists); 404 on unknown/invalid id (invalid-format ids are 404, not 500).
  - `POST /api/matches/:id/cancel` → `engine.cancel`, 200 `{match}` (idempotent: canceling a terminal match returns it unchanged).
  - `DELETE /api/matches/:id` → engine.deleteMatch; running without `?force=1` → 409; with force → cancel then delete; 204.
- Route-level schemas in `packages/server/src/schemas.ts` importing shared domain schemas (no re-definition of MatchRecord etc.).

## Detailed Requirements

1. Handlers contain no business logic beyond translation (engine/store do the work); target ≤ 40 lines each.
2. Typed core errors (`conflict`, not-found, validation) map to the envelope via the issue 18 error registry — extend it here for engine error types.
3. `POST /api/matches` returns only after the match reaches `running` (or failed validation) — workspace provisioning happens within the request (bounded by repo size; acceptable v1; document).
4. All responses JSON except 204; envelope on every error.
5. OpenAPI is NOT generated (v1 non-goal); the DESIGN table is the contract.

## Acceptance Criteria

- [ ] Inject tests with mock-adapter engine on a scratch repo: POST creates and returns 201 with a `running|finished` MatchRecord; GET list contains it; GET by id round-trips; cancel → `canceled`; DELETE running → 409; DELETE `?force=1` → 204 and dir gone.
- [ ] POST with 1 contestant → 400; with 5 → 400; unknown adapterId → 400 naming it; both-unavailable adapters → 400 listing probe problems.
- [ ] `POST /api/repo/inspect` with relative path → 400; with non-repo dir → `{isGitRepo: false}`; with the scratch repo → correct `headSha`/`dirty`.
- [ ] `GET /api/matches/m_%2e%2e%2f` (traversal-shaped id) → 404 without touching fs outside data dir (spy/assert on path builders).
- [ ] `GET /api/adapters` returns all five with probe fields; second call within TTL doesn't re-probe (spy).
- [ ] Oversized taskMarkdown → 413 envelope.

## Validation

`pnpm --filter @agentarena/server test` (inject-based; one real end-to-end mock match).

## Dependencies

02, 04, 05, 07, 08, 15, 18.

## Non-goals

Event/diff/eval/replay reads (22), vote (21), WS (20), UI.

## Design References

DESIGN.md §10.2, §11.3, §16 F1/F6/F7; ADR-005 (no new auth semantics here).
