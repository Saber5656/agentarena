# Title

New-match wizard: repo, task, contestants, review-and-start

## Summary

Implement `/new` per DESIGN.md §12: a four-step wizard (repo → task → contestants → review) with live repo inspection, task frontmatter form + prompt editor, probe-aware adapter selection with model overrides, an explicit review step showing exact check commands, and submission to `POST /api/matches` with error surfacing.

## Context

This is the only mutating UI surface before a match exists. The review step is a security requirement (DESIGN.md §11.2 T4: user sees exactly what commands will run).

## Scope

`src/pages/NewMatchPage.tsx` + `src/components/wizard/`:

- Step 1 Repo: absolute-path text input; on blur/submit `POST /api/repo/inspect`; render `isGitRepo`, `currentRef`, `headSha` (short), `dirty` warning with the F6 explainer ("uncommitted changes won't be visible to contestants"); optional base ref input (default HEAD) — validated only server-side at create.
- Step 2 Task: fields title (required), timeoutSec (number, bounded 10–7200, default from a `GET /api/health`-adjacent defaults? No — hardcode placeholder 900 and let server default apply when blank), checks editor (name + run + optional timeout; add/remove/reorder up to 10), prompt body textarea (markdown, min 10 chars) with char counter (256 KB limit).
  - The wizard composes `taskMarkdown` (YAML frontmatter + body) client-side with a tested serializer (`src/lib/taskMarkdown.ts`: compose + parse round-trip helper; YAML via plain string building with proper quoting — no yaml dep in web; strings escaped conservatively, arrays block-style).
- Step 3 Contestants: cards from `GET /api/adapters` — unavailable adapters disabled with their `problems` listed; select 2–4; per-selected optional model text input (placeholder = defaultModel); duplicate adapter selection allowed (e.g. codex vs codex with different models) — n suffix handled server-side by contestantId generation.
- Step 4 Review: read-only summary — repo path + base SHA, full prompt preview, table of contestants, and the exact check commands in `<code>` blocks with the warning copy "These commands will run on your machine in each contestant's workspace."; Start button → POST; on 201 navigate to `/m/:id`; on error map envelope (400 field messages inline where attributable, 409 concurrent-match banner with link to running match id when server provides it in message).
- Wizard state in local component state (no store); browser-refresh loses it (accepted v1; note in code).

## Detailed Requirements

1. Step validation gates Next (repo must be git repo; ≥ 2 contestants; title + prompt present).
2. `taskMarkdown` serializer output parses back identically (round-trip property test with quoting edge cases: colons, quotes, newlines in names are rejected by input validation instead of escaped — checks name regex `^[A-Za-z0-9 _.-]{1,40}$`).
3. No free-text ever interpreted as HTML; probe `problems` rendered as text list.
4. Keyboard navigable (labels, focus order); submit disabled while in flight.
5. Adapter cards show `version` when available.

## Acceptance Criteria

- [ ] Component tests: full happy path with mocked API (inspect → compose → adapters → review → POST body matches the exact expected `CreateMatchInput` JSON including composed taskMarkdown).
- [ ] Serializer round-trip tests incl. prompt bodies containing `---` lines and backticks; illegal check name blocked with message.
- [ ] Unavailable adapter cannot be selected; selecting 1 contestant blocks Next; 4 selected blocks adding a 5th.
- [ ] Dirty repo shows the F6 warning text; non-repo path shows blocking error.
- [ ] 409 from POST renders the concurrent-match banner; 400 renders the server message.
- [ ] Review step renders every check command verbatim inside code elements.

## Validation

`pnpm --filter @agentarena/web test`; manual run creating a real mock-adapter match end-to-end in dev.

## Dependencies

19, 23 (apiFetch/layout), 24 (StatusChip/toast utilities), 08 (probe payload shape).

## Non-goals

Task file import/export from disk (CLI covers files; v2 for UI), saved task templates, editing a created match.

## Design References

DESIGN.md §9.1, §10.2 (create/inspect rows), §11.2 T4, §12 (Wizard), §16 F6.
