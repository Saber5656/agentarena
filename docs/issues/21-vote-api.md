# Title

Vote API: record and update the human verdict

## Summary

Implement `PUT /api/matches/:id/vote` per DESIGN.md §10.2/§9.4 on top of the issue 17 verdict store, including the terminal-state guard and a `match` WS notification so open result views refresh.

## Context

The human vote completes the "auto evaluation + human vote" judging model. Small, isolated route — kept separate so the verdict semantics (17) and the transport (this) stay independently testable.

## Scope

- `packages/server/src/routes/vote.ts`:
  - `PUT /api/matches/:id/vote` body `{winner: string, notes?: string}` (zod strict; winner = contestantId-format string or literal `tie` / `no-winner`; notes ≤ 4096 chars) → `verdictStore.setVote` → 200 `{verdict}`.
  - Error mapping: unknown match → 404; non-terminal match → 409 `conflict`; foreign contestantId / bad winner value → 400 `invalid_request`.
  - After a successful vote, trigger the engine's match-notification path (a `{type:"match", match}` frame reaches WS subscribers so the results view updates without polling; the verdict itself is re-fetched by the client via `GET /api/matches/:id`).
- Extend `GET /api/matches/:id` (issue 19) test coverage to assert `verdict.vote` round-trips after a PUT (no code change expected; regression net).

## Detailed Requirements

1. Route handler ≤ 30 lines; all logic in issue 17's `setVote`.
2. Idempotent overwrite semantics (second PUT replaces; `votedAt` refreshes).
3. Notes stored verbatim (rendering safety is the UI's job per §11.2 T6); length-capped by schema.
4. No vote deletion endpoint in v1 (overwrite with `no-winner` covers it; documented in code comment).

## Acceptance Criteria

- [ ] Inject tests on a finished mock match: PUT `{winner: <cid>}` → 200, verdict contains vote; GET match returns it; second PUT `{winner: "tie", notes: "close call"}` overwrites.
- [ ] PUT on running match → 409 envelope; unknown match → 404; winner `"c_evil_9"` (not in match) → 400; notes 5000 chars → 400.
- [ ] WS subscriber receives a `match` frame after a successful vote (socket-based test).

## Validation

`pnpm --filter @agentarena/server test`.

## Dependencies

17, 18, 19, 20 (for the notification assertion).

## Non-goals

Vote UI (29), multi-user attribution, vote history/audit trail (v2).

## Design References

DESIGN.md §5.5, §9.4, §10.2 (vote row), §10.3 (`match` frames).
