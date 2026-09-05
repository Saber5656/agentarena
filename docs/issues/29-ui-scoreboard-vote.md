# Title

Results view: tabs shell, scoreboard table, checks matrix, vote panel

## Summary

Implement the terminal-match results surface per DESIGN.md §12: the tab shell (Scoreboard / Diffs / Checks / Live), the scoreboard table from `verdict.scoreboard`, the per-check results matrix with output tails, and the vote panel writing to the vote API.

## Context

This is where comparison and judgment happen (product goals 2–3). The scoreboard shows objective metrics; the vote records the human verdict; nothing declares a winner automatically.

## Scope

`src/components/results/` wired into MatchPage (issue 26 transition):

- `ResultsView.tsx`: tabs Scoreboard | Diffs (issue 28 `DiffTabs`) | Checks | Live (issue 26 feeds); export button placeholder slot (issue 31 fills it); data: match from store, `GET /api/matches/:id` refresh on mount (verdict), `GET /api/matches/:id/eval` for the checks tab (lazy).
- `Scoreboard.tsx`: table — rows per contestant (match order): name+model, StatusChip, checks `passed/total` with a green-full highlight, duration (`1m 42s` formatting), tokens in/out (compact), cost (`$x.xx` or `—`), files/`+`/`−`, exit code. Column headers with unit tooltips. Vote-winner row subtly badged when a vote exists. Interrupted/canceled banner above when applicable (from match status).
- `ChecksMatrix.tsx`: grid contestants × checks; cell = check status icon + duration; cell click opens a drawer with `stdoutTail`/`stderrTail` (mono, collapsed, `truncated` pill), exit code; `skipped` styling for canceled matches; empty-checks state ("No checks defined for this task").
- `VotePanel.tsx`: radio row per contestant + `Tie` + `No winner`, notes textarea (4 096 cap with counter), Save → `PUT /api/matches/:id/vote`; existing vote pre-selected; saved state feedback; 409 (rare race) → toast; votedAt shown.

## Detailed Requirements

1. Scoreboard renders exclusively from `ScoreboardRow` fields — missing optionals render `—` (never 0-fabrication).
2. All text agent-influenced fields (summary strings, tails) remain text nodes (T6).
3. Tabs preserve state when switching (feeds/diffs not refetched; keep-mounted with CSS hiding or state retention).
4. Vote panel disabled (with explanatory note) when match is non-terminal — unreachable normally, guard anyway.
5. Duration/token/cost formatters live in `src/lib/format.ts` with unit tests (shared with lane headers — refactor issue 26's local ones here).

## Acceptance Criteria

- [ ] Component tests: 3-row verdict fixture (one full-pass, one partial with missing cost, one failed without metrics) renders exact cell contents incl. `—` for absent fields.
- [ ] Checks matrix: fixture with passed/failed/timeout/error/skipped renders distinct cells; drawer shows tails with truncated pill; empty-checks state.
- [ ] Vote flow: select + notes → PUT body exact; response verdict updates the panel and the scoreboard badge; pre-existing vote pre-selected; overwrite works.
- [ ] Notes over 4 096 blocked client-side with counter.
- [ ] Tab switching does not refetch eval (fetch spy: once).
- [ ] Formatter tests: 61 000 ms → `1m 1s`; 999 → `999`; 12 400 → `12.4k`; 0.4218 → `$0.42`.

## Validation

`pnpm --filter @agentarena/web test`; manual: finish a demo match, vote, reload, vote persists.

## Dependencies

17, 19, 21, 22, 23, 24 (chips/toast), 26 (page shell), 28 (Diffs tab).

## Non-goals

Export button behavior (31), replay (30), cross-match comparisons, auto-winner logic (never).

## Design References

DESIGN.md §5.5, §9.4, §10.2 (eval/vote rows), §12 (results tabs).
