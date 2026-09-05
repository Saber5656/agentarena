# Title

Scoreboard computation and verdict persistence (votes)

## Summary

Implement scoreboard aggregation from contestant records + eval results into `verdict.json`, and the vote write path with its terminal-state guard, per DESIGN.md §5.5 and §9.4.

## Context

The scoreboard is the objective comparison surface consumed by the UI (issue 29), the CLI `run` output (32), and exports (31). The vote is the human half; no automatic winner exists anywhere (product decision).

## Scope

`packages/core/src/verdict/`:

- `scoreboard.ts`: `computeScoreboard(match: MatchRecord, evals: Record<cid, EvalResult>): ScoreboardRow[]` — one row per contestant in match order, fields exactly per DESIGN.md §5.5: `checksPassed` (count of `passed`), `checksTotal` (declared checks count), duration/tokens/cost from `ContestantRecord.durationMs`/`metrics`, diff numbers from `diffStats`, `status`, `exitCode`. Missing sources → fields omitted (schema-optional), never fabricated.
- `verdictStore.ts` (thin over issue 05 store):
  - `initVerdict(matchId, scoreboard)` — called by the engine at terminal state: writes `{v:1, scoreboard, computedAt}` preserving any existing `vote` (recovery-idempotent).
  - `setVote(matchId, {winner, notes?})` — guards: match exists and terminal (else typed `conflict` error); `winner` is `'tie' | 'no-winner'` or a contestantId present in the match; `notes` ≤ 4 KB; writes `votedAt` now; overwritable (§9.4).
  - `getVerdict(matchId)`.
- Engine hook-up (small change in issue 15's finish path if not already merged: engine calls `computeScoreboard` + `initVerdict` — coordinate; the engine issue declares the call, this issue provides the implementation).

## Detailed Requirements

1. Pure computation, no IO in `scoreboard.ts`.
2. Canceled/failed contestants still get rows (status visible), with whatever metrics exist.
3. `initVerdict` is idempotent and safe to re-run (interrupted-recovery may re-finish bookkeeping).
4. Vote validation errors are typed for clean 400/409 mapping (issue 21).
5. No ranking/sorting logic beyond match order (UI sorts client-side if desired).

## Acceptance Criteria

- [ ] Unit tests: 3-contestant fixture (one completed with 2/3 checks, one timeout with diff, one failed no metrics) → rows carry exactly the available fields; no NaN/undefined leaks (schema-validated output).
- [ ] `setVote` with a foreign contestantId → validation error; with `'tie'` → persisted; second vote overwrites (`votedAt` updates).
- [ ] `setVote` on a `running` match fixture → `conflict` error.
- [ ] `initVerdict` twice (second with same scoreboard) preserves an existing vote.
- [ ] verdict.json validates against `VerdictSchema`.

## Validation

`pnpm --filter @agentarena/core test`.

## Dependencies

02, 05, 15 (finish-path integration point), 16 (EvalResult shape).

## Non-goals

Vote UI (29), vote API route (21), multi-vote/multi-user support, ELO or cross-match aggregation (v2).

## Design References

DESIGN.md §5.5, §9.4; docs/research/prior-art.md (human-vote rationale).
