# ADR-001: TypeScript monorepo, single published npm package

- Status: Accepted (2026-07-07, confirmed by product owner)
- Deciders: product owner (Q&A session), Fable (design)

## Context

agentarena is a local tool that (a) spawns and supervises agent CLI subprocesses, (b) serves a localhost web UI with live WebSocket streaming, (c) ships a CLI. The implementation will be executed by lower-capability agents from granular issues, so a single language with shared types minimizes cross-boundary drift. Candidates: TypeScript, Go+TS, Rust+TS, Python+TS.

## Decision

1. One pnpm-workspaces monorepo, TypeScript everywhere (Node.js >= 22, ESM only).
2. Internal packages: `@agentarena/shared` (types/schemas), `@agentarena/core` (engine), `@agentarena/server` (HTTP/WS), `@agentarena/web` (React UI), and the publishable `agentarena` CLI package.
3. Only the `agentarena` package is published to npm; internal packages are `private: true` and bundled into it at build time (tsup/esbuild). This avoids needing an npm org/scope for v1.
4. Event and API types are defined once in `@agentarena/shared` with zod schemas; server and web import the same definitions.

## Consequences

- Positive: one toolchain (pnpm, vitest, eslint, prettier); UI and engine share the `AgentEvent` type verbatim; distribution is `npx agentarena` / `npm i -g agentarena`, the same channel as the target CLIs.
- Negative: no single-binary distribution; Node >= 22 is a hard runtime requirement (declared in `engines`).
- Neutral: the performance-critical work (agent execution) happens in external processes, so Go/Rust performance advantages would not materialize.

## Alternatives considered

- Go backend + TS UI: single binary, but duplicated event schema across languages (codegen layer) raises issue complexity for implementation agents.
- Rust backend: highest implementation cost, no bottleneck to justify it.
- Python backend: weaker distribution story (pipx/uv) and weaker typing across the WS boundary.
