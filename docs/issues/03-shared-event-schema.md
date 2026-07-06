# Title

Shared AgentEvent schema with size caps and truncation helpers

## Summary

Implement the normalized `AgentEvent` union (envelope + 10 payload types), the `AdapterEmittedEvent` input shape (event minus `v/seq/ts`), and the size-cap/truncation utilities, exactly per DESIGN.md §6 and ADR-004.

## Context

Every subsystem (engine, storage, WS, UI, replay, export) consumes only this schema. Adapters emit `AdapterEmittedEvent`; the engine assigns `v`, `seq`, `ts` and applies caps.

## Scope

- `packages/shared/src/events.ts`:
  - `AgentEventSchema` = discriminated union on `type` over: `status`, `message`, `reasoning`, `tool_call`, `tool_result`, `file_change`, `metrics`, `error`, `result`, `raw` — payload fields exactly per DESIGN.md §6.2 (including `MetricsSnapshot`).
  - Optional `payload.origin: unknown` allowed on every type; optional `payload.truncated: boolean` where §6.3 defines it.
  - `AdapterEmittedEventSchema` (same union without `v`/`seq`/`ts`).
  - Type guards `isAgentEventType(x)`, per-type narrowing helpers.
- `packages/shared/src/eventCaps.ts`:
  - Constants: `MESSAGE_TEXT_CAP = 65536`, `TOOL_OUTPUT_CAP = 16384` (head 8192 + marker + tail 8192), `ORIGIN_CAP = 8192`, `MAX_EVENTS_PER_CONTESTANT = 50000` (DESIGN.md §6.3).
  - `applyEventCaps(e: AdapterEmittedEvent): AdapterEmittedEvent` — pure function implementing every §6.3 rule: truncate `message.text`/`reasoning.text` with `truncated: true`; middle-truncate `tool_result.output`/`raw.text` with `…[truncated]…` marker; drop `origin` when serialized size > cap.
  - `truncateMiddle(s, headLen, tailLen, marker)` helper (UTF-8-safe: never splits a surrogate pair).
- Export from the shared barrel.

## Detailed Requirements

1. Schemas `.strict()`; `seq` positive int; `v` literal 1; `ts` ISO-8601 string.
2. `applyEventCaps` never mutates input; returns the same reference when nothing changes (cheap fast path).
3. Cap measurement on `Buffer.byteLength(s, 'utf8')`, not `.length` (multibyte safety).
4. JSDoc on the union documents ADR-004 rule 3 (unmapped output MUST become `raw`, never dropped).

## Acceptance Criteria

- [ ] Unit tests: every event type round-trips through its schema from a valid fixture; invalid `type` and unknown payload keys rejected.
- [ ] Cap tests: 100 KB message → truncated to cap with `truncated: true`; 100 KB tool output → head+marker+tail exact lengths; multibyte string at the boundary produces valid UTF-8 (no replacement chars from split code points); oversized `origin` dropped while the event survives.
- [ ] Property-ish test: for random strings up to 200 KB, `applyEventCaps` output always validates against `AdapterEmittedEventSchema`.
- [ ] `MAX_EVENTS_PER_CONTESTANT` exported and equals 50000.

## Validation

`pnpm --filter @agentarena/shared test`; typecheck workspace.

## Dependencies

01, 02 (shares zod setup and barrel).

## Non-goals

Seq assignment and redaction (engine, issues 15/09), WS wire envelope (issue 20), persistence (issue 05).

## Design References

DESIGN.md §6 (all); ADR-004.
