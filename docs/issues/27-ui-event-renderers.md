# Title

Event renderers: safe per-type components for every AgentEvent

## Summary

Implement the pure `AgentEvent → JSX` renderer set shared by live lanes, replay, and the standalone export viewer, per DESIGN.md §12 with the §11.2 T6 safety rules: sanitized markdown for messages, collapsible reasoning/tool output, file-change chips, metrics/status/error/raw treatments.

## Context

Everything users read in a lane flows through these components, and the same code is embedded into exported HTML that runs on other people's machines — agent-controlled strings must be inert (XSS-critical, T6).

## Scope

`src/components/events/` in `@agentarena/web` (exported via an internal index consumed by 26/30/31):

- `EventRenderer.tsx`: exhaustive switch over `AgentEvent['type']` delegating to:
  - `MessageEvent`: markdown via `react-markdown` + `remark-gfm` + `rehype-sanitize` (default schema; raw HTML disabled), links rendered with `rel="noopener noreferrer" target="_blank"`; code blocks monospace with overflow-x scroll; `truncated` flag renders a "truncated" pill.
  - `ReasoningEvent`: collapsed by default (first line visible, expand toggle), dimmed styling.
  - `ToolCallEvent`: `shell` → `$ <command>` mono block; other tools → tool name + key/value input summary (values stringified, 200-char clip).
  - `ToolResultEvent`: collapsed beyond 8 lines with expand; ANSI stripped via `strip-ansi` before render; exitCode badge when present.
  - `FileChangeEvent`: chip `± path` with kind-colored icon (created/modified/deleted).
  - `MetricsEvent`: subtle inline stat line (rendered in-feed only when it changes model; the ticker in the lane header is fed from the store instead — renderer returns null for pure-token snapshots, documented).
  - `ErrorEvent`: severity-styled banner (warn amber, fatal red) with source tag.
  - `ResultEvent`: outcome banner + summary text + duration.
  - `StatusEvent`: thin divider line "→ running" style.
  - `RawEvent`: mono dim text, stderr tinted, collapsed in groups of consecutive raws (a `RawGroup` wrapper component groups adjacency at the feed level — export a helper `groupEvents(events)` used by feeds).
- Shared bits: `Collapsible`, `MonoBlock` (overflow-x, max-height), `TruncatedPill`, timestamp tooltip (event `ts` on hover).
- Storybook-less gallery route `/dev/renderers` (dev-only, excluded from production build via `import.meta.env.DEV` guard) rendering one of each type from fixtures for visual QA.

## Detailed Requirements

1. Components are pure (props in → JSX out), no store access, no network — embeddability requirement for the export viewer (31).
2. `react/no-danger` lint holds; sanitizer config forbids raw HTML pass-through; no `javascript:` URLs survive (sanitize schema default behavior verified by test).
3. Long unbroken strings (10 000-char token) must not break layout (`overflow-wrap: anywhere` on text containers, scroll on mono blocks).
4. Every payload field displayed somewhere or deliberately omitted with a code comment.
5. `groupEvents` is O(n) and stable (used per virtualized window in 26 — must accept partial arrays).

## Acceptance Criteria

- [ ] Component tests per type from a fixture covering all 10 types (compile-time exhaustiveness + runtime snapshot per type).
- [ ] XSS suite: message text `<img src=x onerror=alert(1)>`, link `[x](javascript:alert(1))`, raw text with `<script>`, file path `"><svg onload=…` — rendered DOM contains no executable vectors (assert no `script`/`onerror`/`javascript:` in container.innerHTML).
- [ ] ANSI fixture renders stripped; exitCode badge shows for tool results carrying it.
- [ ] Reasoning and long tool results start collapsed and expand on click.
- [ ] Consecutive raw events group into one collapsible block (fixture of 20 raws → 1 group).
- [ ] Dev gallery route renders all types in dev build and is absent from `vite build` output (grep dist).

## Validation

`pnpm --filter @agentarena/web test`; manual gallery review at `/dev/renderers`.

## Dependencies

03, 23. Feeds into 26/30/31.

## Non-goals

Syntax highlighting (v2 — keep deps lean), diff rendering (28), i18n.

## Design References

DESIGN.md §6.2, §11.2 T6, §12, §13 (embedding constraint), §18 (dep list).
