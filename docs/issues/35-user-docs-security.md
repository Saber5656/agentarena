# Title

User documentation: README rewrite, SECURITY.md, task-file reference, CONTRIBUTING stub

## Summary

Write the public-facing docs for the OSS release: a bilingual-intro README (quickstart, demo, screenshots/GIF placeholder discipline), SECURITY.md with the per-adapter isolation table and reporting policy, a task-file format reference, and a minimal CONTRIBUTING.md — all consistent with DESIGN.md as the source of truth.

## Context

Docs are the release gate for an OSS tool whose core risk is "it runs autonomous agents on your machine" — the security expectations must be readable by end users, not only in DESIGN.md (ADR-002 follow-up; DESIGN.md §11).

## Scope

- `README.md` (replaces the one-liner; keeps the Japanese concept line under the English title per ADR-006):
  - What it is (3 sentences + the JP line), demo GIF placeholder comment (media added post-v1), Quickstart: `npx agentarena demo` → `npx agentarena` → create first real match (repo, task, 2+ CLIs installed & logged in), `run`/CI example with mock adapters, adapter support matrix (the §7 table condensed: codex/claude-code/gemini/api-baseline + isolation level per ADR-002), task file example (from §9.1), FAQ (5 entries: costs? keys? what leaves my machine? [answer: nothing except exports you share] can agents damage my repo? [worktree explanation] Windows? [not v1]).
  - Explicit trust warning box near the top mirroring ADR-002's usage assumption.
- `SECURITY.md`:
  - Reporting channel (GitHub private vulnerability reporting), supported-versions table (0.x latest only).
  - User-facing security model: local-server token model (ADR-005 in user words), per-adapter isolation table with the honest Claude-default note (ADR-002), redaction = best-effort with the pattern list reference (issue 09 constant), export privacy warning, "task files and checks are code — review before running third-party ones" (T4), env/key handling rules (§11.4).
- `docs/task-format.md`: normative task-file reference — frontmatter fields/bounds table, checks semantics (§9.2), prompt template arena-v1 verbatim, 3 worked examples (test-fix, feature, refactor with typecheck+test checks).
- `CONTRIBUTING.md` (stub): dev setup (pnpm install/build/test), package map (one line each), test layers (§17), "design changes update docs/DESIGN.md first" rule, issue-drafts location.
- Cross-check pass: every command/flag/path mentioned in docs exists in the implementation (checklist in PR description enumerating each claim → source file).

## Detailed Requirements

1. English primary; JP one-liner preserved; no other translated sections v1.
2. No screenshots committed in this issue (binary bloat before UI stabilizes); placeholders with TODO comments.
3. SECURITY.md table content generated from the same facts as DESIGN.md §7/§11 (manual sync, with a comment linking DESIGN sections).
4. README quickstart commands must be copy-paste runnable on a fresh machine with Node 22 (verified as part of validation).
5. Keep total README under ~200 lines (link to docs/ for depth).

## Acceptance Criteria

- [ ] All four files exist with the sections above; markdownlint-clean (add a docs lint script if trivial, else manual).
- [ ] Fresh-machine walkthrough evidence in the PR: quickstart executed verbatim (demo + one real or mock match) with output pasted.
- [ ] Per-adapter isolation table lists all 5 adapters with sandbox mechanism and an honest risk note for each.
- [ ] Every CLI command shown in README exists (`--help` cross-check evidence).
- [ ] Task-format doc round-trips: its three examples parse via the engine's task parser (unit test added in core: `docs/task-format.md` examples extracted and validated — literal fixture copies acceptable with a sync comment).

## Validation

PR checklist evidence; the example-parse unit test; reviewer reads SECURITY.md against ADR-002/005 for drift.

## Dependencies

32 (CLI surface final), 34 (install story final); content sources: ADR-002/005/006, DESIGN.md §7/§9/§11/§14.

## Non-goals

Docs website, API reference generation, localization beyond the JP concept line, marketing copy.

## Design References

DESIGN.md §7, §9.1–9.2, §11, §14; ADR-002, ADR-005, ADR-006.
