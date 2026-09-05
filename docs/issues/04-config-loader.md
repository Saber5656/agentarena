# Title

Config loading: data dir resolution, config.json schema, token generation

## Summary

Implement in `@agentarena/core` the data-directory resolution (`~/.agentarena`, `AGENTARENA_HOME` override), the zod-validated `config.json` per DESIGN.md §15 including first-run creation with a fresh 256-bit token at file mode 0600, and typed accessors for adapter/default settings.

## Context

Every subsystem reads configuration through this module. The token is the security anchor of ADR-005; dangerous adapter values must be rejected at load time (DESIGN.md §11.3).

## Scope

- `packages/core/src/config/paths.ts`: `resolveDataDir(env): string` — `AGENTARENA_HOME` (must be absolute) else `~/.agentarena`; `matchDir(dataDir, matchId)`, `contestantDir(...)`, `workspaceDir(...)` path builders that validate ids with the shared regexes **before** joining (DESIGN.md §11.2 T7) and assert the result stays under the expected root (`path.resolve` prefix check).
- `packages/core/src/config/schema.ts`: `ConfigSchema` exactly per DESIGN.md §15 (defaults included via `.default(...)`), plus per-adapter sub-schemas. Validation rules: codex `sandbox` enum only `read-only | workspace-write` (reject `danger-full-access`); `port` 1024–65535; `token` `/^[0-9a-f]{64}$/`.
- `packages/core/src/config/load.ts`: `loadConfig(dataDir): Promise<{config, warnings}>` — creates dir (0700) and default config with `crypto.randomBytes(32).toString('hex')` token on first run; writes atomically (tmp + rename) with mode 0600; unknown keys collected as warnings (strip + warn, not fail — forward compat per §15); dangerous values → typed `ConfigError`.
- `saveConfig(dataDir, config)` for later mutation (port change).

## Detailed Requirements

1. First-run creation must be race-safe enough for one machine: write tmp file then `rename`; if the file appears concurrently, re-read and use it.
2. 0600/0700 modes set explicitly (`fs.chmod` after rename to survive umask differences).
3. Never log the token; the module exposes `redactedForLog(config)` returning a copy with `token: "***"`.
4. All fs operations async (`node:fs/promises`); no sync IO.
5. Unknown top-level keys and unknown adapter keys both produce warnings listing the exact key paths.

## Acceptance Criteria

- [ ] Unit tests (tmp dirs): fresh dir → config created, token matches `/^[0-9a-f]{64}$/`, file mode 0600, dir 0700 (skip mode asserts on win32 — not CI'd anyway).
- [ ] Re-load returns identical config; corrupted JSON → `ConfigError` with actionable message (path + parse error).
- [ ] `sandbox: "danger-full-access"` in codex adapter config → `ConfigError` naming the key.
- [ ] Unknown key `adapters.codex.futureFlag` → loaded config drops it and warning `adapters.codex.futureFlag` is returned.
- [ ] `matchDir(dd, "m_../../etc")` throws before any fs access; valid ULID id passes.
- [ ] `AGENTARENA_HOME=/rel/../path` (non-absolute after normalization rules) rejected.

## Validation

`pnpm --filter @agentarena/core test`; manual: run loader twice against a scratch dir, `ls -l` shows 0600.

## Dependencies

01, 02.

## Non-goals

Reading env API keys (adapter runtime, issues 14), CLI `--data-dir` flag plumbing (issue 32), storage of matches (issue 05).

## Design References

DESIGN.md §8, §11.2 T7/T9, §11.3, §15; ADR-003, ADR-005.
