# Title

Adapter interface, contestant context, and adapter registry

## Summary

Define the `AgentAdapter` contract, `ContestantContext`, `ProbeResult`, `RunOutcome`, and a registry resolving adapterIds to implementations with their validated config slices, per DESIGN.md §7.1.

## Context

Issues 10–14 implement five adapters against this contract; the engine (15) and the adapters API (19) consume the registry. The contract is the seam that keeps CLI churn out of the engine (ADR-004).

## Scope

`packages/core/src/adapters/`:

- `types.ts`: interfaces exactly per DESIGN.md §7.1 (`AgentAdapter`, `ProbeResult`, `ContestantContext`, `RunOutcome`, `AdapterEmittedEvent` imported from shared). `AdapterConfig` = the per-adapter zod-validated slice from config.json (issue 04 schema types).
- `registry.ts`:
  - `createRegistry(config): AdapterRegistry` with `get(adapterId)`, `list(): {id, displayName, kind, defaultModel?}[]`, `probeAll(): Promise<Record<adapterId, ProbeResult>>` (parallel, 5 s per-probe timeout wrapper, cached with 60 s TTL per DESIGN.md §10.2).
  - Registry is constructed with adapter factory functions so issues 10–14 can register incrementally; unknown adapterId → typed `UnknownAdapterError`.
- `probeCli.ts`: shared helper `probeCliVersion({command, versionArgs: ['--version'], parse: (stdout) => string})` using the process runner with 5 s timeout — returns `{available, version?, problems[]}`; ENOENT → `available: false`, problem "not found on PATH: <command> — install hint <docsUrl>".
- Placeholder registration for all five ids returning `available: false, problems: ["not implemented"]` until their issues land (keeps API endpoint functional early).

## Detailed Requirements

1. `probe()` must be side-effect free and never throw — errors fold into `problems`.
2. `ContestantContext.emit` accepts `AdapterEmittedEvent` (no `v/seq/ts`) — typed so adapters cannot set engine-owned fields.
3. `RunOutcome.outcome` values match the contestant terminal statuses mapping in DESIGN.md §5.3/§7.1.
4. Registry never instantiates adapter state at construction (lazy factories) so a broken optional adapter cannot break startup.
5. Probe cache: TTL 60 s, `probeAll({fresh: true})` bypass for `doctor`.

## Acceptance Criteria

- [ ] Typecheck: a sample adapter implementing the interface compiles; setting `seq` on an emitted event is a type error.
- [ ] Unit tests: registry lists exactly the five v1 ids with display names ("Codex CLI", "Claude Code", "Gemini CLI", "API Baseline", "Mock"); `get("nope")` throws `UnknownAdapterError`.
- [ ] `probeCliVersion` against `node` (exists) yields `available: true` with a version; against `definitely-not-a-binary-xyz` yields `available: false` with an ENOENT problem, in < 6 s.
- [ ] A probe that hangs (test double sleeping 30 s) is cut off by the 5 s wrapper with problem "probe timed out".
- [ ] Second `probeAll()` within TTL does not re-invoke probes (spy count).

## Validation

`pnpm --filter @agentarena/core test`.

## Dependencies

01, 02, 03, 04, 06.

## Non-goals

Any real adapter behavior (issues 10–14), engine wiring (15), HTTP exposure (19).

## Design References

DESIGN.md §7.1, §7.2, §10.2 (`GET /api/adapters`), §16 F1.
