# Title

Self-contained HTML replay export: standalone viewer build, assembler, route, UI button

## Summary

Implement the single-file HTML export per DESIGN.md §13: a build-time standalone viewer bundle (reusing lanes/renderers/diff/replay-clock with the embedded bundle as data source), an XSS-safe assembler in core, the `GET /api/matches/:id/export.html` route, and the export button in the results view.

## Context

Shareable replays are a headline feature (product decision: export in v1). The exported file runs in untrusted contexts — CSP + sanitization are mandatory (T6); it must work offline from `file://` with zero network requests.

## Scope

- Standalone viewer build (in `packages/web`):
  - Entry `src/replay-standalone/main.tsx`: reads `<script type="application/json" id="agentarena-replay">`, zod-parses `ReplayBundle` (parse failure → friendly error page), renders a trimmed app: header (title, exportedAt, appVersion), replay controls + lanes (issue 30 components), results tabs scoreboard/diffs/checks (issues 28/29 components, fed from the bundle, vote read-only display — no vote editing), no router, no token code, no network imports.
  - `vite.export.config.ts`: single-chunk build (`cssCodeSplit: false`, inlined assets, `format: 'iife'`), output `dist-export/viewer.js` + `viewer.css`; wired into the package build script; a build-check script asserts the JS contains no `fetch(`/`XMLHttpRequest`/`WebSocket(` token usages against external URLs (static grep guard, documented as heuristic).
- Assembler in `packages/core/src/replay/exportHtml.ts`:
  - `renderExportHtml({bundle, viewerJs, viewerCss}): string` — template with: `<!doctype html>`, `<meta charset>`, `<title>agentarena replay — {escapedTitle}</title>`, CSP meta exactly `default-src 'none'; script-src 'unsafe-inline'; style-src 'unsafe-inline'; img-src data:; font-src data:`, inline `<style>`, the JSON script tag with `JSON.stringify(bundle)` post-processed replacing `<` → `<`, ` `/` ` escaped, then inline `<script>` viewer.
  - Asset loading: viewer JS/CSS read from files shipped inside the published package (path resolution util; missing assets → typed error telling the user the build is incomplete).
  - Size guard: total > 20 MB → include result anyway; the callers (route/CLI/UI) surface the warning (return `{html, bytes, warnLarge}`).
- Server route (in `packages/server`): `GET /api/matches/:id/export.html` (token via query allowed per issue 18) → 409 non-terminal; headers `Content-Type: text/html; charset=utf-8`, `Content-Disposition: attachment; filename="agentarena-<matchId>.html"`.
- UI: Export button in `ResultsView` slot (issue 29) → navigates to the export URL (browser download), pre-download modal with the privacy warning copy: "The file contains the full transcripts, diffs, and command output of this match. Review before sharing." (exact copy).

## Detailed Requirements

1. Escaping proof obligations: `</script>` inside any transcript cannot terminate the JSON script tag (covered by `<` escaping); template title/vars HTML-escaped separately.
2. The exported page renders from `file://` (relative-nothing: no absolute references at all).
3. Viewer must not require sessionStorage/token code paths (guard with build-time env flag `__STANDALONE__`).
4. Deterministic output for a given bundle+assets (no timestamps injected at assembly beyond bundle.exportedAt).
5. `exportedAt`/`appVersion` set at bundle assembly (issue 22 function gains these params — coordinate; single writer).

## Acceptance Criteria

- [ ] Assembler unit tests: bundle containing `</script>`, `<img onerror>`, U+2028 in messages → output HTML parses (jsdom), script-tag JSON re-parses to deep-equal bundle, no unescaped `</script>` sequence in the JSON segment.
- [ ] CSP meta exact-string test; `Content-Disposition` filename test on the route; 409 for running match.
- [ ] Build check: `dist-export/viewer.js` exists after `pnpm build`, static no-network grep passes.
- [ ] E2E-ish test (node + playwright in issue 33 covers full open; here: jsdom smoke): exported HTML for a finished mock match boots the viewer enough to render the match title and lane headers (jsdom with scripts enabled or a minimal DOM assertion on the embedded JSON + error-free parse path; document chosen approach).
- [ ] UI: button visible only on terminal matches; modal shows exact privacy copy before triggering download.
- [ ] Export of a match with truncated diff/events renders truncation markers (fixture).

## Validation

Unit/route tests; full browser validation lands in issue 33 (Playwright opens the exported file from `file://` and scrubs the replay).

## Dependencies

22 (bundle assembly), 27, 28, 29, 30 (components), 18 (query-token rule), 23 (build infra).

## Non-goals

Hosted sharing, import of external replay files into the app (v2), PDF/video export.

## Design References

DESIGN.md §13, §11.2 T6, §10.2 (export row), §14 (`export` CLI uses the same assembler — issue 32).
