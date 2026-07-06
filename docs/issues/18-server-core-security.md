# Title

Fastify server core: loopback bind, token auth, host/origin validation, static serving, error envelope

## Summary

Implement the HTTP server skeleton in `@agentarena/server` with the full ADR-005 security middleware stack (loopback bind, bearer/query token, Host allowlist, Origin checks), security headers, the uniform error envelope, port fallback, and static SPA serving with history fallback. No business routes yet.

## Context

Every API and WS route (issues 19–22, 31) mounts on this foundation. The middleware ORDER is normative (DESIGN.md §10.1); getting it wrong reintroduces T1.

## Scope

`packages/server/src/`:

- `app.ts`: `buildApp({config, dataDir, engine, store, webDistDir}): FastifyInstance` (deps injected; engine/store may be null-stubbed until issue 19).
  - Fastify 5, `bodyLimit: 1 MiB`, no `x-powered-by`, disable request logging of query strings (token leak, ADR-005) — log method+path only.
  - Hook order (onRequest):
    1. Host allowlist: `Host` must be `127.0.0.1[:port] | localhost[:port] | [::1][:port]` → else 403 `forbidden` (before routing).
    2. Token gate for `/api/*`: `Authorization: Bearer <token>` (timing-safe compare) or `?token=` (accepted ONLY for WS upgrades and `GET /api/matches/:id/export.html`) → else 401 `unauthorized`. `/api/health` also requires the token (uniform).
    3. Origin check: for WS upgrades and non-GET `/api/*`: if `Origin` present it must be `http://` + allowed host → else 403; absent Origin allowed.
  - `onSend` security headers on ALL responses: `X-Content-Type-Options: nosniff`, `Referrer-Policy: no-referrer`, `Cache-Control: no-store` for `/api/*`; on HTML: `Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; connect-src 'self' ws://127.0.0.1:* ws://localhost:*; frame-ancestors 'none'` (exact string constant; adjust ws-port at render? No — wildcard port form as written; verify browser acceptance in issue 23 e2e and correct the constant if a browser rejects it — record outcome in code comment).
  - Error envelope: global error + not-found handlers returning `{error: {code, message}}` with codes per DESIGN.md §10 (zod errors → 400 `invalid_request` with first-issue message; thrown typed errors mapped by a small registry).
- `static.ts`: `@fastify/static` serving `webDistDir` at `/`; GET fallback to `index.html` for non-`/api`, non-asset paths (SPA history routing).
- `listen.ts`: `startServer(app, {port})` binding `127.0.0.1` ONLY; if `EADDRINUSE` and port came from config default → try up to +20 (DESIGN.md F8), return the actual port; explicit `--port` → fail fast.
- `printLaunchInfo(port, token)` helper returning `http://127.0.0.1:<port>/?token=<token>` (used by CLI; never logged elsewhere).

## Detailed Requirements

1. Timing-safe token compare (`crypto.timingSafeEqual` on equal-length buffers; length mismatch → immediate fail without compare).
2. The token never appears in any log line or error message (test greps).
3. Health route `GET /api/health` → `{ok: true, version, dataDir}` (version from package.json via build-time constant).
4. All hosts/CSP/allowlist values are exported constants (single source for tests).
5. `buildApp` performs no listening (injectable for tests via `app.inject`).

## Acceptance Criteria

- [ ] Tests via `app.inject`: missing token → 401 envelope; wrong token → 401; correct bearer → 200 health.
- [ ] `Host: evil.com` with valid token → 403 (proves order: host check first — also assert 403 with NO token + bad host).
- [ ] POST `/api/health` (any `/api` non-GET) with `Origin: https://evil.com` + valid token → 403; with `Origin: http://127.0.0.1:<port>` → not 403 (404/405 acceptable).
- [ ] `?token=` on a plain REST GET (not export/WS) → 401 (query token restricted).
- [ ] Static: `GET /` serves index.html; `GET /m/whatever` (no file) serves index.html; `GET /api/nope` → 404 JSON envelope, NOT index.html.
- [ ] Security headers present on `/` (CSP, nosniff) and `/api/health` (no-store).
- [ ] Port fallback test: occupy the port, start with config-default port → binds port+1 and reports it; explicit-port mode → typed failure.
- [ ] Grep test: response bodies/log capture from all tests never contain the token string.

## Validation

`pnpm --filter @agentarena/server test`.

## Dependencies

01, 02, 04 (config/token). Web dist not required (tests use a fixture dir).

## Non-goals

Business routes (19–22), WS handling (20), TLS (loopback-only by design), rate limiting (single trusted user).

## Design References

DESIGN.md §10.1, §10.2 (health), §11.2 T1/T9, §16 F8; ADR-005.
