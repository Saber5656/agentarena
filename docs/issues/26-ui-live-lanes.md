# Title

Live match view: contestant lanes, virtualized event feeds, follow mode, cancel

## Summary

Implement the live half of `/m/:id` per DESIGN.md §12: WS-driven lane grid (one lane per contestant with status header, elapsed timer, token/cost ticker, virtualized event feed, follow toggle), connection status surface, and match cancel — switching to the results tabs (issue 29) when the match is terminal.

## Context

This is the spectating heart of the product. It must stay smooth at thousands of events per lane (virtualization) and survive reconnects (issue 23 WS client).

## Scope

- `src/pages/MatchPage.tsx`: loads `GET /api/matches/:id` (404 → not-found page), opens the match socket (issue 23), feeds the zustand match store; renders `<LiveView>` while non-terminal, `<ResultsView>` (issue 29 stub until it lands) when terminal — live→terminal transition animates a brief "match finished" banner then reveals results tabs; a "Live" tab remains available to inspect feeds after finish.
- `src/components/live/LaneGrid.tsx`: responsive columns (2 lanes → 2 cols; 3–4 → 2×2 on ≤1440px, 4 cols on wide; min lane width 320px).
- `src/components/live/Lane.tsx`:
  - Header: adapter display name + model, `StatusChip`, elapsed (ticks 1 s from `startedAt` while running, frozen at `durationMs` when terminal), metrics ticker (input/output tokens compact `12.4k`, `costUsd` as `$0.42` when present).
  - Feed: `@tanstack/react-virtual` list over the store's event array rendering via issue 27's `<EventRenderer>` (until 27 lands, a plain fallback renderer showing `type` + JSON snippet — replaced by 27; wire the import seam now: `renderEvent(event)` indirection module).
  - Follow mode: on by default — auto-scroll to bottom on new events; any manual upward scroll disables it; "Follow" pill re-enables; per-lane state.
  - Footer: last `tool_call` command one-liner while running ("current activity").
- Match header: title, match StatusChip, global elapsed, Cancel button (confirm dialog → `POST /cancel`; hidden when terminal), connection badge from WS status (`connecting/backfilling/live/reconnecting`).
- Not-found and interrupted states rendered distinctly (interrupted matches open directly in results with an interrupted banner).

## Detailed Requirements

1. 60fps-ish smoothness target: event append must not re-render other lanes (store selectors per contestant; React.memo lanes).
2. Virtualizer handles variable-height items (measure-based) — required for markdown messages.
3. Elapsed timers derive from record timestamps (no drift accumulation; recompute from `Date.now() - startedAt` each tick).
4. Cancel is idempotent client-side (button disables in flight; 409/terminal responses treated as success-refresh).
5. All statuses in DESIGN.md §5.3 render correctly in the header chips (enum-exhaustive switch with compile-time exhaustiveness).

## Acceptance Criteria

- [ ] Component tests: store seeded with 3 contestants × mixed events → 3 lanes render headers, tickers (token formatting cases: 999→`999`, 12400→`12.4k`), footers show last command.
- [ ] Follow-mode test: appending events scrolls to bottom (virtualizer scroll spy); simulated user scroll-up stops auto-scroll; pill resumes it.
- [ ] Live→terminal store transition swaps to results (stub) with the Live tab still accessible and feeds intact.
- [ ] Cancel: confirm → POST called; button absent for terminal fixture.
- [ ] 10 000-event lane fixture mounts and scrolls without rendering all rows (assert rendered row count ≪ total via virtualizer).
- [ ] Reconnect status surfaces the `reconnecting` badge (WS client status stub).

## Validation

`pnpm --filter @agentarena/web test`; manual demo-match spectating in dev (3 mock lanes) checking smooth scroll and follow behavior.

## Dependencies

19, 20, 23; 27 (renderer seam — can merge before/after via the indirection module); 24 (StatusChip).

## Non-goals

Event renderers themselves (27), diffs/results tabs (28/29), replay (30), lane reordering/hiding (v2).

## Design References

DESIGN.md §5.3, §10.3, §12 (live lanes), §16 F15.
