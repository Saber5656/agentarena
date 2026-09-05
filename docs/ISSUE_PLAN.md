# agentarena — v1 Issue Plan

- Derived from: `docs/DESIGN.md` (canonical). GitHub Issues are derived from `docs/issues/*.md`; when they disagree, these files win.
- 35 issues, 5 waves. Issue files: `docs/issues/NN-short-title.md`.

## v1 completion statement

When every issue 01–35 is completed and validated, agentarena v1 is complete: a user on macOS or Linux with Node ≥ 22 can `npx agentarena demo` for a keyless demo race; create a match on their own git repository against any 2–4 of Codex CLI, Claude Code, Gemini CLI, and an API-baseline contestant; watch all contestants live in the browser (transcripts, commands, file changes, token/cost tickers); get automatic check results, diff comparison, and a scoreboard; record a human vote; replay the match; and export it as a single offline HTML file — with the security posture of ADR-002/ADR-005 (worktree + native CLI sandboxes, loopback token-gated server) and CI-verified quality gates. The only remaining pre-publish actions are the two owner decisions listed under Known unknowns (npm package name confirmation, LICENSE choice) and the owner-triggered first release tag.

## Recommended execution order and waves

Order within a wave = recommended sequence; issues in the same wave with no mutual dependency may proceed in parallel.

### Wave 0 — Foundations
| # | Issue file | One-liner |
|---|---|---|
| 01 | 01-monorepo-scaffolding.md | pnpm/TS monorepo, lint/test/CI skeleton |
| 02 | 02-shared-domain-types.md | ids, match/task/eval/verdict schemas, transitions |
| 03 | 03-shared-event-schema.md | AgentEvent union + caps |
| 04 | 04-config-loader.md | data dir, config.json, token, path builders |
| 05 | 05-storage-match-store.md | atomic docs, JSONL logs, listing, recovery scan |

### Wave 1 — Execution engine
| # | Issue file | One-liner |
|---|---|---|
| 06 | 06-process-runner.md | group spawn, kill-tree, timeout, line streams |
| 07 | 07-workspace-manager.md | worktrees, repo mutex, diff capture |
| 08 | 08-adapter-interface-registry.md | adapter contract + registry + probes |
| 09 | 09-secret-redaction.md | env/pattern redaction filter |
| 10 | 10-mock-adapter.md | fixture replay adapter + demo fixtures |
| 15 | 15-match-engine.md | orchestration state machine (mock-testable) |
| 16 | 16-evaluation-runner.md | check execution + eval.json |
| 17 | 17-scoreboard-verdict.md | scoreboard math + vote store |
| 11 | 11-codex-adapter.md | codex exec --json adapter (parallel w/ 15–17) |
| 12 | 12-claude-code-adapter.md | claude -p stream-json adapter |
| 13 | 13-gemini-adapter.md | gemini headless adapter (fixture-first) |
| 14 | 14-api-baseline-adapter.md | one-shot diff-via-API adapter |

### Wave 2 — Server
| # | Issue file | One-liner |
|---|---|---|
| 18 | 18-server-core-security.md | Fastify core + ADR-005 middleware |
| 19 | 19-matches-api.md | match lifecycle REST + inspect + adapters |
| 20 | 20-ws-live-stream.md | WS subscribe/backfill/live |
| 21 | 21-vote-api.md | vote route |
| 22 | 22-match-read-api.md | events/diff/eval/replay-bundle reads |

### Wave 3 — Web UI & export
| # | Issue file | One-liner |
|---|---|---|
| 23 | 23-web-scaffolding.md | Vite/React app, token bootstrap, API/WS clients |
| 24 | 24-ui-match-list.md | home list + delete |
| 27 | 27-ui-event-renderers.md | safe per-type event components |
| 26 | 26-ui-live-lanes.md | live lanes, virtualized feeds, cancel |
| 28 | 28-ui-diff-viewer.md | diff parser (shared) + viewer |
| 25 | 25-ui-new-match-wizard.md | new-match wizard |
| 29 | 29-ui-scoreboard-vote.md | results tabs, scoreboard, checks, vote |
| 30 | 30-ui-replay.md | replay clock + scrubber |
| 31 | 31-replay-html-export.md | standalone viewer + HTML assembler + route/button |

### Wave 4 — CLI, E2E, release
| # | Issue file | One-liner |
|---|---|---|
| 32 | 32-cli-entrypoint.md | serve/run/ls/export/doctor/demo |
| 33 | 33-e2e-suite.md | node E2E + Playwright flows |
| 34 | 34-packaging-release.md | bundling, pack-install smoke, release workflow |
| 35 | 35-user-docs-security.md | README, SECURITY.md, task reference |

## Dependency table

Transitive dependencies omitted (e.g. everything depends on 01).

| Issue | Direct dependencies | Blocks (direct) |
|---|---|---|
| 01 | — | all |
| 02 | 01 | 03,04,05,08,15,17,19,22,23 |
| 03 | 01,02 | 05,08,09,10,11,12,13,14,15,20,22,23,27 |
| 04 | 01,02 | 05,07,08,09,10,15,16,18,32 |
| 05 | 01–04 | 15,16,17,20,22,32 |
| 06 | 01 | 07,08,10,11,12,13,14,15,16 |
| 07 | 01,04,06 | 14(tests),15,19,32 |
| 08 | 01–04,06 | 10,11,12,13,14,15,19,25 |
| 09 | 01,03,04 | 15,16 |
| 10 | 01,03,04,06,08 | 15(tests),32,33 |
| 11 | 01–04,06,08 | v1 lineup (no downstream code deps) |
| 12 | 01–04,06,08 | 〃 |
| 13 | 01–04,06,08 | 〃 |
| 14 | 01–04,06,07,08 | 〃 |
| 15 | 02–10 | 16(hook),17(hook),19,20,21,22,32 |
| 16 | 02,04,05,06,09 | 17,29(data) |
| 17 | 02,05,15,16 | 21,22,29 |
| 18 | 01,02,04 | 19,20,21,22,23,31,32 |
| 19 | 02,04,05,07,08,15,18 | 20(tests),21,22,24,25,26 |
| 20 | 03,05,15,18,19 | 21(test),23,26 |
| 21 | 17,18,19,20 | 29 |
| 22 | 02,03,05,15,17,18,19 | 28,29,30,31,32 |
| 23 | 01,02,03,18,20 | 24,25,26,27,28,29,30,31 |
| 24 | 19,23 | 25,26(shared components) |
| 25 | 08,19,23,24 | — |
| 26 | 19,20,23,24,(27 seam) | 29,30 |
| 27 | 03,23 | 26,30,31 |
| 28 | 22,23 | 29,31 |
| 29 | 17,19,21,22,23,24,26,28 | 31(button slot) |
| 30 | 22,23,26,27 | 31 |
| 31 | 18,22,23,27,28,29,30 | 32(export),33 |
| 32 | 04,05,07,08,10,15,16,17,18,22,23,31 | 33,34 |
| 33 | 01–32 | 34 |
| 34 | 01,18,23,31,32,33 | 35, first release |
| 35 | 32,34 | release gate |

## Coverage table (DESIGN.md → issues)

| DESIGN.md section | Covered by issues |
|---|---|
| §1 Overview / §2 Concepts | (context for all; no code) |
| §3 Scope | this plan; §3.3/3.4 lists below |
| §4.1 Monorepo | 01, 34 |
| §4.2–4.3 Topology & data flow | 15, 18, 20, 32 |
| §5.1 Ids | 02 |
| §5.2 MatchRecord | 02, 05, 15 |
| §5.3 State machines | 02 (tables), 15 (enforcement), 26 (rendering) |
| §5.4 EvalResult | 02, 16 |
| §5.5 Verdict/Scoreboard | 02, 17, 29 |
| §6 Event model | 03 (schema/caps), 15 (pipeline), 27 (rendering) |
| §7.1 Adapter interface | 08 |
| §7.2 Child-process contract | 06 |
| §7.3 Codex | 11 |
| §7.4 Claude Code | 12 |
| §7.5 Gemini | 13 |
| §7.6 API baseline | 14 |
| §7.7 Mock | 10 |
| §7.8 Fairness | 11–14 (flags), 15 (prompt/template recording) |
| §8 Storage | 04 (paths), 05 (store) |
| §9.1 Task format & prompt | 02 (template), 15 (parsing), 25 (composer), 35 (reference doc) |
| §9.2 Evaluation | 16 |
| §9.3 Diff capture | 07 |
| §9.4 Scoreboard & vote | 17, 21, 29 |
| §10.1 Middleware | 18 |
| §10.2 REST | 19 (lifecycle), 21 (vote), 22 (reads), 31 (export route) |
| §10.3 WS | 20, 23 (client), 26 (consumer) |
| §11.1–11.2 Threat model T1–T9 | T1:18 · T2:07/11–13(flags)/04(config guard) · T3:09/31/35 · T4:25/35 · T5:01/34 · T6:27/28/31 · T7:04/05/19/22 · T8:03/06/20 · T9:04/18 |
| §11.3 Validation table | 02,03,04,13(platform warn),18,19,20,22,25 |
| §11.4 Secret rules | 09, 14 (env keys), 35 (docs) |
| §12 Web UI | 23–30 |
| §13 Replay/export | 22 (bundle), 30 (viewer logic), 31 (export) |
| §14 CLI | 32 |
| §15 Config | 04 |
| §16 Failure modes F1–F15 | F1:15/19 · F2:11–13 · F3:11–13(ADR-004) · F4:06/15/16 · F5:03/06 · F6:07/25 · F7:07/15/19 · F8:18/32 · F9:05/07/24 · F10:05 · F11:14 · F12:07 · F13:20 · F14:31 · F15:20/23/26 |
| §17 Testing | each issue's Validation + 33 (E2E) |
| §18 Toolchain/deps | 01, 23, 34 |

Gap audit: every normative DESIGN.md behavior maps to ≥ 1 issue. If an implementer finds otherwise, that is a planning bug — file it against this document (DESIGN.md §19).

## Validation strategy (whole product)

1. **Per-issue gates**: every issue defines Acceptance Criteria + Validation; nothing merges red. Unit/integration tests accumulate in-package (DESIGN.md §17 pyramid).
2. **Fixture realism**: CLI adapters (11–13) must commit real captured fixtures and update `docs/research/cli-agent-interfaces.md` — parser truth is recorded reality, not docs memory.
3. **Mock-first integration**: engine (15), server (19–22), UI (23–30), CLI (32) all fully testable with the mock adapter — CI never needs API keys.
4. **Env-gated real-CLI smokes**: `AGENTARENA_TEST_CODEX/CLAUDE/GEMINI/BASELINE=1` tests run manually per adapter issue with evidence pasted in the PR.
5. **E2E gate (33)**: API/WS/storage assertions + Playwright user flows incl. the export opened from `file://` with a zero-network assertion and an XSS canary fixture.
6. **Security acceptance**: ADR-005 middleware order tests (18), traversal-id tests (19/22), XSS suite (27/28/31), redaction canary through the full pipeline (15/33), config foot-gun rejections (04/12).
7. **Release gate (34/35)**: pack-install smoke on a clean machine profile, LICENSE guard, docs walkthrough evidence.

## Deferred v2 items

Container execution driver (ADR-002); Windows support; generic (non-coding) task type; task suites + batch statistics; cross-match leaderboards/ELO; LLM-as-judge; more adapters (aider, opencode, cursor-agent, amp, …); hosted replay sharing; replay-file import into the app; arena-side live cost budget enforcement; side-by-side/intraline diff view; syntax highlighting; MCP-based integrations; saved task templates in UI; vote history/audit.

## Known unknowns (may spawn issues during implementation)

| # | Unknown | Owner action / resolving issue | Status (2026-07-07) |
|---|---|---|---|
| U1 | Codex `--json` per-item payload field names | Issue 11 fixture capture updates research doc | open |
| U2 | Claude Code `result` message exact fields on 2.1.x | Issue 12 fixture capture | open |
| U3 | Claude Code native sandbox settings schema (`--settings` passthrough) | Issue 12 documents; possible follow-up issue "claude sandbox preset" | open |
| U4 | Gemini stream-json event schema; stdin prompt reliability; `--sandbox` on macOS | Issue 13 fixture-first step | open |
| U5 | npm package name `agentarena` availability | Issue 34 blocking sub-task; owner decides fallback name | open — **owner decision** |
| U6 | LICENSE choice (MIT assumed nowhere; release-gated) | Owner decision before first release; guard in issue 34 | open — **owner decision** |
| U7 | CSP `connect-src` wildcard-port WS form acceptance across browsers | Issue 18 constant + issue 23/33 verification note | open |
| U8 | Virtualized feed performance at 50k events (cap adequacy) | Issue 26 perf AC; revisit caps if failing | open |
| U9 | Auth-state probing for codex/claude/gemini (cheap non-mutating check) | Issues 11–13 probe hints; possible follow-up "doctor auth checks" | open |
| U10 | `git apply --3way` behavior coverage for exotic baseline diffs (mode changes, symlinks) | Issue 14 tests; follow-up issue if failure classes emerge | open |

Any newly discovered unknown during implementation must be appended here (single list, ADR-004-style additive discipline).
