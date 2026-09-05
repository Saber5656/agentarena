# Title

Diff viewer: shared unified-diff parser and per-contestant diff UI

## Summary

Implement a dependency-free unified-diff parser in `@agentarena/shared` and the diff viewing components (file list, collapsible per-file hunks, add/del coloring, truncation notice) used in the results tabs and the export viewer.

## Context

Diff comparison is the core "compare" deliverable (product goal 2). The parser lives in shared because the standalone export viewer (31) embeds the same components (DESIGN.md §12/§13).

## Scope

- `packages/shared/src/diff/parseUnifiedDiff.ts`: `parseUnifiedDiff(patch: string): ParsedDiff` →
  `{ files: [{ oldPath, newPath, kind: 'created'|'modified'|'deleted'|'renamed'|'binary', hunks: [{header, lines: [{kind: 'context'|'add'|'del', text}]}], additions, deletions }], truncated: boolean }`.
  - Handles `git diff --binary` output shape: `diff --git` headers, `new file mode`/`deleted file mode`, `rename from/to`, `Binary files … differ` and `GIT binary patch` sections (mark `binary`, skip payload), `\ No newline at end of file`, the agentarena truncation marker line (DESIGN.md §9.3) → `truncated: true`.
  - Total, never throws; unparseable segments folded into a synthetic file entry `kind: 'binary'`? NO — synthetic entry `{oldPath: '(unparsed)', …}` with raw lines as context (documented fallback; keeps viewer honest).
- `src/components/diff/` in web:
  - `DiffViewer.tsx`: props `{patch: string}` → parse + render: summary bar (`N files, +A −D`, truncation warning banner when flagged), file list sidebar (or stacked on narrow), per-file `DiffFile` collapsible (default: expanded ≤ 10 files, else collapsed) with hunk rendering — line numbers both sides, add/del/context row styling, mono, horizontal scroll; binary files → "binary changed" row; renames labeled.
  - Per-file cap: files > 2 000 lines render first 500 lines + "show all" button (client-side perf guard).
  - `DiffTabs.tsx`: given contestants, lazy-fetch `GET …/diff` per tab on first open (apiFetch text mode), empty diff → "No changes" state; fetch error → inline retry.
- Wire into MatchPage results as the "Diffs" tab content (tab shell may be a stub until issue 29 provides the full results layout — export the components; integration finalized in 29).

## Detailed Requirements

1. Parser pure, zero deps, O(n) over lines; fuzz-resistant (property test: random byte strings never throw, always yield a ParsedDiff).
2. All rendered strings are text nodes (paths are agent-influenced! T6).
3. Line numbers computed from hunk headers (`@@ -a,b +c,d @@`), correct across multiple hunks.
4. `X-Agentarena-Truncated` header from the diff route also surfaces the banner (belt and braces with in-patch marker).
5. Dark-theme friendly add/del colors from issue 23 tokens.

## Acceptance Criteria

- [ ] Parser unit tests: fixture patches for create/modify/delete/rename/binary/no-newline/multi-hunk — structure and counts exact; truncation marker detected; property fuzz test passes 1 000 random inputs.
- [ ] Line-number test: crafted 2-hunk file → numbering matches expected pairs.
- [ ] Component tests: 3-file fixture renders summary `3 files`, colored rows, collapsed state logic (11-file fixture collapsed by default); binary row; "No changes" empty state; XSS path fixture (`<img…>` in filename) inert.
- [ ] 2 000+ line file fixture renders capped with "show all" expanding fully.
- [ ] Tab lazy-fetch: switching tabs fetches each contestant's diff exactly once (fetch spy).

## Validation

`pnpm --filter @agentarena/shared test` + `pnpm --filter @agentarena/web test`.

## Dependencies

22 (diff route), 23; coordinates with 29 (tab shell).

## Non-goals

Side-by-side split view (v2), intraline word diff (v2), syntax highlighting, diff editing.

## Design References

DESIGN.md §9.3, §10.2 (diff row), §11.2 T6, §12 (Diffs tab), §13.
