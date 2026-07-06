# Title

npm packaging and release pipeline with provenance

## Summary

Make the `agentarena` package publishable and repeatable: bundle internal packages + web/export assets into the CLI package, verify a clean `npx`-style install works, and add a tag-triggered GitHub Actions release workflow publishing with npm provenance, per DESIGN.md §18 and ADR-001.

## Context

Distribution decision (ADR-001): single published package, internal workspace packages bundled. Supply-chain posture (DESIGN.md §11.2 T5): pinned actions, provenance, no postinstall.

## Scope

- `packages/cli` packaging:
  - tsup (or esbuild) config bundling cli+server+core+shared into `dist/` (ESM, node22 target, shebang on `dist/main.js`); web `dist/` and `dist-export/` viewer assets copied into the package under `assets/web/` and `assets/export/`; asset path resolution (issues 18/31) switched to package-relative lookup that works from an installed location (`import.meta.url` based).
  - `package.json`: `name: agentarena` (see availability check below), `version: 0.1.0`, `bin`, `files: ["dist", "assets", "README.md"]`, `engines.node: ">=22"`, `publishConfig.access: public`, remove `private`.
  - Root `pnpm build` orchestrates: shared→core→server→web (app + export viewer)→cli in order.
- **Name availability check** (blocking sub-task): verify `agentarena` is free on npm at implementation time (`npm view agentarena`); if taken, escalate to the owner with candidates (e.g. `agent-arena-cli`) — do NOT publish under an improvised name (ISSUE_PLAN known-unknown; owner decision gate).
- Install verification: CI job packs (`pnpm pack`), installs the tarball into a scratch dir with plain npm, runs `agentarena --version`, `agentarena doctor`, and `agentarena demo --no-open` smoke (headless boot + match finishes) — proves no workspace-only resolution leaks.
- `.github/workflows/release.yml`: trigger on tag `v*`; jobs: full CI (reuse), pack-install smoke, then publish with `npm publish --provenance` (`id-token: write` permission, `NPM_TOKEN` secret documented as manually configured by the owner — the workflow must fail with a clear message when absent); GitHub Release created with generated notes; all actions SHA-pinned.
- `CHANGELOG.md` seeded (`0.1.0 — initial release` placeholder, Keep-a-Changelog format).
- **LICENSE gate**: release workflow fails early if `LICENSE` file is absent (owner decision pending — ISSUE_PLAN known-unknowns; this guard makes accidental unlicensed publish impossible).

## Detailed Requirements

1. The published tarball contains no source maps pointing at missing sources, no tests, no fixtures except mock demo fixtures (needed at runtime by `demo`), no `.env`-like files (assert via pack-list snapshot test).
2. Tarball size budget: warn > 15 MB (CI check with the number in the log).
3. `agentarena --version` reads the real package version (build-time injection, no fs read of package.json at runtime from a bundled context — or robust resolution; pick one, test it installed).
4. Secrets: publishing auth only via CI secret; nothing in-repo; provenance requires the workflow to run from the canonical repo (documented).
5. No token/keys required for pack-install smoke (mock-only).

## Acceptance Criteria

- [ ] CI pack-install job: tarball installs in a clean dir; `--version` correct; `doctor` runs; `demo --no-open` reaches a finished match; asset resolution works from the installed path.
- [ ] Pack-list snapshot matches the allowlist (`files` behavior verified).
- [ ] Release workflow dry-run (workflow_dispatch with `dry_run: true` input skipping publish) is green end-to-end; publish step reached and skipped.
- [ ] Missing LICENSE → release workflow fails at the gate with the explanatory message.
- [ ] All workflow actions SHA-pinned (grep check in CI itself).
- [ ] `npm view agentarena` availability outcome recorded in the PR + ISSUE_PLAN known-unknown updated (owner pinged if taken).

## Validation

CI jobs above; a maintainer-run `workflow_dispatch` dry run before the first real tag.

## Dependencies

01, 18, 23, 31, 32, 33 (CI green precondition).

## Non-goals

Actual first publish (owner-triggered via tag after LICENSE + name decisions), homebrew/binary distribution, auto-update mechanisms, docs site.

## Design References

DESIGN.md §11.2 T5, §14, §18; ADR-001; ISSUE_PLAN known-unknowns (npm name, LICENSE).
