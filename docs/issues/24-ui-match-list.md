# Title

Home page: match list with status, navigation, and deletion

## Summary

Implement the `/` home page per DESIGN.md §12: newest-first match list (title, status chip, adapter badges, created time, check summary), navigation to live/results/replay, delete with confirmation (force for running), and a "New match" entry point.

## Context

First screen users see; also the recovery surface after restarts (interrupted matches must be visibly distinct).

## Scope

- `src/pages/HomePage.tsx`:
  - Data: `GET /api/matches` (apiFetch + shared schema), poll every 5 s while any listed match is non-terminal (no WS on this page, v1 simplicity).
  - Row: title, match status chip (colors from issue 23 tokens; all 8 statuses render), adapter badges with model suffix (e.g. `claude-code · fable`), relative created time (plain `Intl.RelativeTimeFormat` helper, no dep), evalSummary compact (`2/3 · 1/3 · 0/3` per contestant order) when present.
  - Row click → `/m/:id`; explicit "Replay" link when terminal.
  - Delete: trash action → inline confirm; running match → confirm copy explains force-cancel, calls `DELETE ?force=1`; errors surface as toast (minimal toast utility in `src/lib/toast.ts`, no dep).
  - Empty state: short explainer + "New match" button + `agentarena demo` hint (copy exact: "No matches yet. Create one, or run `agentarena demo` for a demo race.").
  - Header button "New match" → `/new`.
- Pagination: "Load more" appending pages of 50 (server `offset`), only when `total` exceeds loaded.

## Detailed Requirements

1. All rendering through React text nodes (no dangerouslySetInnerHTML anywhere in the app — repo-wide lint rule added here via eslint `react/no-danger`).
2. Poll timer cleaned up on unmount; no overlapping fetches (in-flight guard).
3. Delete updates the list optimistically with rollback on error.
4. Status chip component (`StatusChip`) shared (results header in 26/29 reuse it) — export from `src/components/`.
5. Layout usable at 1280×800 and 768px width (single-column collapse).

## Acceptance Criteria

- [ ] Component tests (@testing-library/react, apiFetch mocked): renders rows for a 3-match fixture covering `running`, `finished`, `interrupted`; chips carry distinct token classes; interrupted visually flagged.
- [ ] Delete flow: confirm → DELETE called with force flag iff running; row removed; API error → row restored + toast text shown.
- [ ] Empty state renders the exact copy above.
- [ ] Poll starts only when a non-terminal match exists in the response (timer spy) and stops after all become terminal.
- [ ] "Load more" hidden when `total ≤ loaded`; appends without duplicates otherwise.
- [ ] eslint `react/no-danger` rule active repo-wide and passing.

## Validation

`pnpm --filter @agentarena/web test`; visual smoke in `vite dev` against a mock server.

## Dependencies

19, 23.

## Non-goals

Live streaming on this page, search/filter (v2), bulk delete.

## Design References

DESIGN.md §10.2 (list/delete rows), §12 (Home row), §16 F9 (interrupted visibility).
