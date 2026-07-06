# Title

Web SPA scaffolding: Vite/React/Tailwind, token bootstrap, API + WS clients, layout shell

## Summary

Scaffold `@agentarena/web` per DESIGN.md §12: Vite + React 19 + Tailwind v4 + react-router + zustand, the ADR-005 token bootstrap (`?token=` → sessionStorage → URL strip), a typed fetch wrapper, a reconnecting WS client honoring the issue 20 protocol, and the app layout shell with route stubs.

## Context

All UI issues (24–30) build on these primitives. The WS client's resume logic (`since` map) is the counterpart of issue 20 and must be exact.

## Scope

- Vite app in `packages/web`: `vite.config.ts` (React plugin, `@tailwindcss/vite`, build outDir `dist`, dev proxy `/api` → `http://127.0.0.1:7788` including ws), `index.html`, Tailwind v4 single-import CSS, base design tokens (font stack, dark-first palette, status colors for the 7 contestant/8 match statuses as CSS variables).
- `src/lib/token.ts`: on boot read `?token=` → `sessionStorage.setItem('agentarena_token')` → `history.replaceState` strip; `getToken()`; missing token → render a full-page instruction screen ("start via `agentarena` CLI and use the printed URL") instead of the app (no API calls attempted).
- `src/lib/api.ts`: `apiFetch<T>(path, {method, body, schema})` — adds `Authorization: Bearer`, JSON handling, parses error envelope into typed `ApiError{code, message, status}`, optional zod parse of responses (schemas from `@agentarena/shared`).
- `src/lib/ws.ts`: `openMatchSocket({matchId, since, onFrame, onStatus})` — connects `ws(s)://<host>/api/matches/:id/ws?token=…`, sends `subscribe`, validates frames with shared `WsServerFrame` schema, exposes connection status (`connecting|backfilling|live|closed`), auto-reconnects with exponential backoff (1s→10s cap) passing the caller-maintained `since` map, de-dupes by seq (belt-and-braces per DESIGN.md F15).
- `src/store/matchStore.ts` (zustand): normalized state `{match, eventsByContestant: Map<cid, AgentEvent[]>, connection}`, `applyFrame(frame)` reducer (append event keeping seq order, replace match, mark live), `sinceMap()` selector.
- `src/App.tsx` + router: routes `/`, `/new`, `/m/:id`, `/m/:id/replay` with placeholder pages; shared layout (top bar: product name, home link, version from health endpoint later).
- Root scripts integration: `pnpm --filter @agentarena/web build` produces `dist/`; server package gains a dev note (README in package) for running vite dev + server together.

## Detailed Requirements

1. React StrictMode on; TypeScript strict; no `any` in the three lib modules.
2. WS client must not lose events across reconnect (store-driven `since`), and must ignore duplicate seqs idempotently.
3. Token never rendered into the DOM and never put back into the URL.
4. All shared types imported from `@agentarena/shared` (no local copies).
5. Bundle guard: no dependency beyond the DESIGN.md §18 web list.

## Acceptance Criteria

- [ ] `pnpm --filter @agentarena/web build` succeeds; `vite dev` renders the shell with route stubs.
- [ ] Unit tests (vitest + jsdom): token bootstrap strips the query param and stores the token; missing token renders the instruction screen and `apiFetch` is not called (spy).
- [ ] `apiFetch` test: 401 envelope → typed ApiError; schema-validated happy path.
- [ ] WS client test against a scripted mock server (`ws` in node test): subscribe sent with provided since; frames dispatched; forced close → reconnect issued with UPDATED since (event received pre-close reflected); duplicate seq dropped.
- [ ] `applyFrame` reducer test: out-of-order duplicate event ignored; match frame replaces match; events array stays seq-sorted.
- [ ] Status-color CSS variables exist for every status enum value (test enumerates shared enums against the generated CSS/tokens file).

## Validation

`pnpm --filter @agentarena/web test` + build; manual `vite dev` smoke against a mock-adapter server.

## Dependencies

01, 02, 03, 18, 20 (protocol frozen). Server routes 19/22 helpful but stubs suffice for tests.

## Non-goals

Real pages (24–30), export viewer build target (31), visual polish beyond tokens.

## Design References

DESIGN.md §10.3, §12, §16 F15, §18; ADR-005 (token flow).
