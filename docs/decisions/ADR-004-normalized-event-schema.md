# ADR-004: Normalized event schema with lossless raw fallback

- Status: Accepted (2026-07-07)
- Deciders: Fable (design)

## Context

Each agent CLI emits a different streaming format (research/cli-agent-interfaces.md §4): Codex JSONL thread/turn/item events, Claude Code stream-json Anthropic-shaped messages, Gemini stream-json (schema unverified). The UI, storage, replay, export, and evaluation must not care which CLI produced an event. CLI formats also drift across versions.

## Decision

1. One normalized `AgentEvent` union (defined once in `@agentarena/shared`, zod-validated) is the only event currency in the system: adapters translate CLI output into it at the process boundary; everything downstream (storage, WS, UI, replay, export) consumes only `AgentEvent`.
2. Event types (v1, closed set): `status`, `message`, `reasoning`, `tool_call`, `tool_result`, `file_change`, `metrics`, `error`, `result`, `raw`. Exact payloads in DESIGN.md §6.
3. **Raw fallback rule**: any CLI output line an adapter cannot confidently map MUST be emitted as `raw` (never dropped, never guessed). This makes version drift degrade the UI gracefully instead of breaking matches.
4. Every event carries `v: 1`, per-contestant monotonic `seq` (assigned by the engine, not adapters), and ISO-8601 `ts`.
5. Adapters may attach the original parsed CLI line under `payload.origin` capped at 8 KB, for debugging and fixture regeneration.
6. Parser fixtures: each CLI adapter ships recorded real-output fixtures; parser unit tests run against fixtures, and the mock adapter replays normalized fixtures for demos/e2e.

## Consequences

- Positive: UI renderers, replay, and export are written once; new adapters are pure translators; CLI upgrades break at worst down to `raw` rendering.
- Negative: normalization loses CLI-specific richness (e.g. Codex `plan_update` granularity) — mitigated by `payload.origin` and by extending the union in minor versions (additive only).
- Constraint: the union is append-only; renaming/removing a type or required field is a breaking change requiring `v: 2` and a replay-compat plan.

## Alternatives considered

- Raw passthrough per CLI with per-CLI UI renderers: three UIs to maintain, replay/export tripled; rejected.
- Adopting one CLI's format as canonical: couples the whole product to one vendor's drift; rejected.
