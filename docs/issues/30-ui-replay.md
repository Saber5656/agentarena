# Title

Replay view: virtual clock, timeline scrubber, playback controls

## Summary

Implement `/m/:id/replay` per DESIGN.md §12: load the full event history, then re-drive the existing lane components through a virtual clock with play/pause, speed multipliers (×1/×2/×4/×8), a seekable timeline scrubber based on event timestamps, and jump-to-end.

## Context

Replay turns matches into rewatchable artifacts and is the exact engine the standalone export viewer (31) reuses — the virtual-clock module must be UI-framework-pure and data-source-agnostic (bundle in, no network).

## Scope

- `src/lib/replayClock.ts` (pure TS, no React): given `contestants: {cid, events: AgentEvent[]}[]`, builds a merged timeline `[{atMs, cid, event}]` where `atMs = ts - min(ts)`;
  API: `create(timeline)`, `play()`, `pause()`, `setSpeed(n)`, `seek(ms)` (emits a reset + all events ≤ ms), `onEmit(cb)`, `onTick(cb)` (progress for the scrubber), `durationMs`, `dispose()`. Implementation: single `setTimeout` chain to the next event at `delay/speed` (no per-frame timers); seek is synchronous re-emission.
- `src/pages/ReplayPage.tsx`:
  - Data: `GET /api/matches/:id/replay` (issue 22 bundle; simplest single fetch; non-terminal → redirect to `/m/:id` with toast).
  - A replay-local event store (same shape as the live store, fed by clock emissions) rendering the SAME `LaneGrid`/`Lane` components (issue 26) — lanes must accept the store via props/context seam (small refactor in 26's components allowed here: extract `useLaneEvents(source)`).
  - Controls bar: play/pause (space key), speed cycle buttons, scrubber (`input[type=range]`, width-proportional, tick marks at contestant terminal events), elapsed/total labels, jump-to-end; contestant status chips reflect replay-time state (statuses derive from already-emitted `status` events).
  - Header link back to results.
- Auto-start paused at t=0 with lanes empty; scoreboard/vote intentionally NOT shown here (results view owns them; header links out).

## Detailed Requirements

1. Seeking backwards resets lane stores then re-emits (idempotent renderers make this cheap); seeking forward emits only the delta.
2. Speed change mid-play re-schedules correctly (no burst emission, no stall) — timeline position preserved.
3. Clock is fully deterministic given the timeline (unit-testable with fake timers).
4. Memory: one in-memory copy of events (bundle) + per-lane arrays; 50 k events must not duplicate further (stores reference the same event objects).
5. Keyboard: space play/pause, ←/→ seek ±5 s (bound when page focused, not in inputs).

## Acceptance Criteria

- [ ] Clock unit tests (fake timers): 3-event timeline at ×1/×4 emits at expected virtual times; pause holds; seek(midpoint) emits exactly the prefix; backward seek re-emits prefix after reset callback; dispose cancels pending timers.
- [ ] Page tests: bundle fixture → lanes populate as the clock plays (advance fake timers); scrubber value tracks tick; jump-to-end renders full feeds and terminal chips.
- [ ] Non-terminal match id → redirected with toast (mocked 409).
- [ ] Space/arrow keys drive the clock (keydown simulation), ignored while textarea focused.
- [ ] Same-component proof: Lane component identity used by live and replay is the same export (import path assertion test or simple grep-level lint rule documented in the test).

## Validation

`pnpm --filter @agentarena/web test`; manual: replay a finished demo match, scrub both directions at ×8.

## Dependencies

22, 23, 26 (lane components + seam refactor), 27.

## Non-goals

Frame-accurate timing fidelity (event-order fidelity is the goal), replay of running matches (live view covers it), export packaging (31).

## Design References

DESIGN.md §12 (Replay row), §13 (viewer reuse constraint), §10.2 (replay route).
