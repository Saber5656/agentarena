# Title

Scaffold pnpm/TypeScript monorepo with lint, test, and CI skeleton

## Summary

Create the five-package pnpm workspace defined in DESIGN.md §4.1 with strict TypeScript, eslint + prettier, vitest wiring, and a GitHub Actions CI workflow running lint/typecheck/test on ubuntu and macos. No product logic.

## Context

Every other issue lands inside this skeleton. ADR-001 fixes the stack: TypeScript everywhere, ESM only, Node >= 22, pnpm workspaces, only the `agentarena` package published later.

## Scope

- Root: `package.json` (`private: true`, `"type": "module"`, `engines.node ">=22"`, `packageManager` pinned to a pnpm 9+ version), `pnpm-workspace.yaml` (`packages/*`), `tsconfig.base.json` (strict, `module`/`moduleResolution` `NodeNext`, `target` ES2022, `declaration` true), `.gitignore`, `.editorconfig`, eslint flat config + prettier config, `vitest.workspace.ts`.
- Packages `packages/shared`, `packages/core`, `packages/server`, `packages/web`, `packages/cli` each with: `package.json` (names per DESIGN.md §4.1, `private: true` except `packages/cli` named `agentarena` also `private: true` for now), `tsconfig.json` extending base, `src/index.ts` placeholder exporting a const, one passing placeholder vitest test.
- Workspace deps: core→shared, server→core+shared, cli→server+core+shared, web→shared (via `workspace:*`).
- `.github/workflows/ci.yml`: trigger `pull_request` + `push` to `main`; matrix `os: [ubuntu-latest, macos-latest]`, Node 22 via `actions/setup-node` with pnpm cache; steps: install (frozen lockfile), lint, typecheck (`tsc -b` or per-package `tsc --noEmit`), test. All third-party actions pinned by full commit SHA (DESIGN.md §18). Workflow `permissions: contents: read`.
- `.github/dependabot.yml`: weekly updates for npm + github-actions ecosystems.
- Root scripts: `lint`, `typecheck`, `test`, `build` (recursive no-op ok for now).

## Detailed Requirements

1. `pnpm install && pnpm lint && pnpm typecheck && pnpm test` all succeed from a clean clone.
2. ESLint covers `packages/*/src/**/*.ts(x)`; prettier check integrated (`lint` fails on format drift).
3. No runtime dependencies added in this issue (dev deps only). No postinstall scripts (verify lockfile).
4. `pnpm-lock.yaml` committed.
5. Do NOT add a LICENSE file (pending owner decision — ISSUE_PLAN "Known unknowns").

## Acceptance Criteria

- [ ] Fresh clone: `pnpm install --frozen-lockfile && pnpm lint && pnpm typecheck && pnpm test` exit 0 on macOS and Linux.
- [ ] CI workflow runs the same four steps green on both OSes for a PR.
- [ ] All five packages exist with the exact names/paths from DESIGN.md §4.1 and resolve each other via `workspace:*`.
- [ ] Every GitHub Action referenced is pinned to a full commit SHA.
- [ ] Repo contains no LICENSE file and no `.npmrc` with auth.

## Validation

Run the four root scripts locally on macOS; open a draft PR and confirm CI matrix is green; `grep -R "postinstall" pnpm-lock.yaml package.json packages/*/package.json` finds nothing.

## Dependencies

None (first issue).

## Non-goals

Product code, publishing config (`files`, `bin` — issue 34), Playwright (issue 33), Tailwind/Vite app setup (issue 23).

## Design References

DESIGN.md §4.1, §17, §18; ADR-001.
