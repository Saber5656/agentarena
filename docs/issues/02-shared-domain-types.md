# Title

Shared domain model: ids, match/contestant/task/eval/verdict schemas

## Summary

Implement in `@agentarena/shared` the zod schemas and inferred TypeScript types for identifiers, `MatchRecord`, `ContestantRecord`, `TaskSpec`/`CheckSpec`, `EvalResult`/`CheckResult`, `Verdict`/`ScoreboardRow`, and the status enums with their transition tables, exactly as specified in DESIGN.md §5.

## Context

These types are the single source of truth consumed by core, server, web, and the export viewer (ADR-001, ADR-004). Getting field names exact here prevents drift everywhere else.

## Scope

- `packages/shared/src/ids.ts`: `newMatchId()` (`m_` + ULID via `ulid` package), `newContestantId(adapterId, n)`, and `MATCH_ID_RE`, `CONTESTANT_ID_RE`, `ADAPTER_ID_RE` regexes exactly per DESIGN.md §5.1, plus `isMatchId/isContestantId` guards.
- `packages/shared/src/domain.ts`: zod schemas `MatchRecordSchema`, `ContestantRecordSchema`, `TaskSpecSchema`, `CheckSpecSchema`, `EvalResultSchema`, `CheckResultSchema`, `VerdictSchema`, `ScoreboardRowSchema`; enums `MatchStatus`, `ContestantStatus`, `CheckStatus`; every field name/optionality exactly per DESIGN.md §5.2–§5.5 including `v: 1` literals.
- `packages/shared/src/transitions.ts`: `MATCH_TRANSITIONS: Record<MatchStatus, MatchStatus[]>` and `CONTESTANT_TRANSITIONS` encoding §5.3 (including `interrupted` reachable from every non-terminal match status), `isTerminalMatchStatus()`, `isTerminalContestantStatus()`, `assertTransition(from, to)` throwing typed `InvalidTransitionError`.
- `packages/shared/src/prompt.ts`: `ARENA_PROMPT_TEMPLATE_V1` exact string and `renderPrompt(taskBody: string): string` per DESIGN.md §9.1 (template id `arena-v1`).
- Barrel export from `packages/shared/src/index.ts`.
- Add runtime deps to shared: `zod`, `ulid` (DESIGN.md §18 closed list).

## Detailed Requirements

1. Schemas use `.strict()` (unknown keys rejected) per DESIGN.md §11.3.
2. Timestamps validated as ISO-8601 strings (regex or `z.string().datetime({ offset: true })` with UTC convention documented in JSDoc).
3. `ScoreboardRow.winner` does not exist — no auto-winner field anywhere (DESIGN.md §5.5).
4. Bounded values enforced: `CheckSpec.run` ≤ 4096 chars, checks array ≤ 10, `timeoutSec` within [10, 7200] (DESIGN.md §11.3).
5. JSDoc on every exported type referencing its DESIGN.md section.

## Acceptance Criteria

- [ ] `pnpm --filter @agentarena/shared test` green with unit tests covering: valid fixture objects parse; each enum rejects unknown values; unknown keys rejected; id regexes accept/reject documented examples (`m_01HZX…` ok, `m_lower` not, traversal strings not).
- [ ] `assertTransition('running','finished')` throws; `assertTransition('evaluating','finished')` passes; full transition tables match DESIGN.md §5.3 (test enumerates them).
- [ ] `renderPrompt("body")` output matches the exact template in DESIGN.md §9.1 (snapshot test).
- [ ] No dependency besides `zod` and `ulid` added.

## Validation

Unit tests above; `pnpm typecheck` across workspace (core/server compile against the new exports once they consume them).

## Dependencies

01.

## Non-goals

AgentEvent schema (issue 03), API/WS protocol types (issues 19/20), config schema (issue 04).

## Design References

DESIGN.md §5 (all), §9.1, §11.3; ADR-001, ADR-004.
