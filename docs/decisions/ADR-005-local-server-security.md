# ADR-005: Local server security model (loopback bind + bearer token + host/origin validation)

- Status: Accepted (2026-07-07)
- Deciders: Fable (design); posture confirmed by product owner (local single-user tool)

## Context

The server exposes match data (source code diffs, transcripts) and mutating endpoints (create match = **spawn processes that edit files and spend API credits**). Even loopback-bound local servers are attackable from the browser: malicious web pages can attempt CSRF against `http://127.0.0.1:<port>` and DNS-rebinding attacks bypass same-origin assumptions. Other local OS users on shared machines can also reach loopback ports.

## Decision

1. **Bind**: `127.0.0.1` only, hard-coded in v1 (no `--host` flag; remote/LAN exposure is explicitly out of scope and the flag's absence is deliberate).
2. **Token auth**: a 256-bit random token generated on first run, stored in `~/.agentarena/config.json` (file mode `0600`). Every `/api/*` request must present it (`Authorization: Bearer <token>`); WebSocket upgrades must present it (`?token=` on the upgrade URL). Static UI assets are served without the token; the SPA receives the token via `?token=` on the launch URL (printed/opened by the CLI), stores it in `sessionStorage`, and immediately strips it from the address bar via `history.replaceState`.
3. **Host allowlist** (DNS-rebinding defense): every request's `Host` header must be `127.0.0.1[:port]`, `localhost[:port]`, or `[::1][:port]`; otherwise 403 before any routing.
4. **Origin check**: for WS upgrades and all non-GET `/api/*` requests, if an `Origin` header is present it must match the allowed hosts; absent `Origin` (curl, CLI) is allowed because the bearer token is still required.
5. **No CORS headers ever** (no cross-origin consumers by design); `Content-Security-Policy` on served HTML restricting to `'self'` plus same-host WS; `X-Content-Type-Options: nosniff`.
6. **No cookies** — auth is header/query-token only, which removes ambient-authority CSRF.
7. Path-shaped inputs from the API (`matchId`, `contestantId`, repo paths) are validated against strict formats and resolved paths are asserted to stay inside their expected roots (DESIGN.md §11 input-validation table).

## Consequences

- Positive: drive-by web pages cannot enumerate or mutate (no token, Host/Origin fail); other local users cannot use the API without reading the user's `0600` config; token-in-URL exposure is limited to the launch URL and our own server logs (which we do not write for query strings).
- Negative: opening the UI manually without the token requires copying the printed URL; acceptable for a developer tool.
- Follow-up: exact middleware order and tests are fixed in issue 18 acceptance criteria.

## Alternatives considered

- Cookie session + CSRF tokens: more moving parts, reintroduces ambient authority; rejected.
- No auth ("it's just localhost"): fails both the rebinding and shared-machine threats for an endpoint that can spawn code-editing processes; rejected.
