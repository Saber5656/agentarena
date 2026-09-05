# Title

WebSocket live streaming: subscribe, seq-based backfill, live tail, slow-consumer policy

## Summary

Implement `GET /api/matches/:id/ws` per the DESIGN.md §10.3 protocol: token-gated upgrade, `subscribe` handshake with per-contestant `since` seqs, ordered backfill from storage, seamless switch to live engine events, `match` state frames, pings, and the slow-consumer drop policy.

## Context

This is the pipe that makes spectating live. Correct backfill→live stitching without gaps or duplicates (per-contestant `seq`) is the core difficulty; the client relies on it for reconnect/resume (DESIGN.md F15).

## Scope

`packages/server/src/routes/ws.ts` using `@fastify/websocket`:

- Upgrade path: token via `?token=` (issue 18 gate), Origin check (issue 18 hook), match must exist (else close after `{type:"error", code:"not_found"}`).
- Handshake: first client frame within 10 s must parse as `{type:"subscribe", since?: Record<contestantId, number>}` (zod; unknown contestantIds ignored; missing `since` = all zeros) else `{type:"error", code:"invalid_request"}` + close (§11.3).
- Flow per connection after subscribe:
  1. Send `{type:"hello", match}` (current record).
  2. Register a live-event buffer (queue) on the engine subscription BEFORE reading backfill (no gap window): buffer live events while backfilling.
  3. Backfill: for each contestant, `readEvents(afterSeq: since[cid])` in pages, send `{type:"event", contestantId, event}` in seq order per contestant.
  4. Drain the live buffer, de-duplicating by `(contestantId, seq)` ≤ already-sent seqs.
  5. Send `{type:"live"}`; from then on forward engine notifications directly: `event` frames and `{type:"match", match}` frames.
- Terminal matches: pure-replay connections are valid — backfill then `live` then (nothing further); connection stays open until client closes.
- Ping/pong every 30 s; no pong for 2 intervals → close.
- Slow consumer: if the socket's buffered/queued frame count exceeds 5 000 → close with `{type:"error", code:"conflict", message:"slow consumer"}` best-effort (client reconnects with `since`).
- Unsubscribe/cleanup on close: engine listener removed (no leaks; test with listener-count assertion).
- Shared wire-frame types added to `@agentarena/shared` (`WsServerFrame`, `WsClientFrame` zod schemas) — used by the web client (issue 23).

## Detailed Requirements

1. Per-contestant ordering guaranteed end-to-end; cross-contestant ordering is NOT guaranteed (documented in the shared types JSDoc).
2. Backfill paging (limit 1000) must not block the event loop pathologically (await between pages).
3. Multiple concurrent sockets per match supported (each with own buffer).
4. All frames JSON text; binary frames from client → protocol error close.
5. No frame ever contains the token.

## Acceptance Criteria

- [ ] Test (real ws client against a listening test server, mock-adapter match): connect during `running` with `since: {}` → hello, complete ordered backfill (seq 1..n contiguous per contestant), `live` marker, then live events continue; final frames include a `match` frame with terminal status.
- [ ] Reconnect test: disconnect mid-match, reconnect with `since` = last seqs → zero duplicate and zero missing seqs (client-side assertion over the union).
- [ ] Race test: events emitted continuously while backfilling (slow fixture) → stitched stream has no gaps/dupes (the buffer-before-backfill logic).
- [ ] No first frame for 10 s → server closes with invalid_request error frame.
- [ ] Unknown match id → error frame not_found + close.
- [ ] Replay-only: terminal match connect → full backfill + live marker, socket stays open.
- [ ] After close, engine listener count returns to baseline.
- [ ] Bad token / bad Origin on upgrade → 401/403 (no upgrade).

## Validation

`pnpm --filter @agentarena/server test` (uses real sockets on an ephemeral port; mock adapters).

## Dependencies

03, 05, 15 (subscribe API), 18, 19 (match creation used by tests).

## Non-goals

Client implementation (23), compression (permessage-deflate off v1), multiplexing several matches per socket.

## Design References

DESIGN.md §10.3, §11.3 (WS row), §16 F13/F15; ADR-005.
