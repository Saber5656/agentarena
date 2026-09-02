# Review resolution addendum

- Repository: `Saber5656/agentarena`
- Pull request: #1
- Original PR head before this resolution addendum: `ccd11c8a20e14af3ff7103ff67ba6de20d882806`
- The immutable current PR head is supplied by the parent task's fresh GitHub read immediately before review/reply/resolve; any later head change invalidates that evidence and requires a fresh review.
- Scope: each exact review thread below has a normative design contract and a focused verification gate.
- This is design-level handling only; it does not claim implementation, test, build, CI, or security validation is complete.
- Per task instruction, the PR review bot is not re-triggered after these responses/resolutions.

## 1. Thread `PRRT_kwDOTNj-ic6Ot8SW` — Cover `payload.origin` in the redaction policy.

**Normative resolution**: Treat `payload.origin` as untrusted event data: redact it with the same secret scrubber before persistence, replay, or export, and drop/replace the field when it cannot be safely scrubbed.

**Focused verification gate**: Persist and replay payloads containing bearer tokens, API keys, and multiline secrets in `origin`; assert the stored event, replay, and export contain only the redacted form.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 2. Thread `PRRT_kwDOTNj-ic6Ot8SY` — Narrow the `?token=` exception to the explicit export/WS cases.

**Normative resolution**: Query-token authentication is allowed only for the documented WebSocket upgrade and `GET /api/matches/:id/export.html` routes. All other endpoints reject query tokens and require the normal authorization path.

**Focused verification gate**: Exercise both allowed routes and representative future/download-like routes with query tokens; assert only the two explicit cases authenticate.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 3. Thread `PRRT_kwDOTNj-ic6Ot8Sa` — Use a boundary-aware containment check for path builders.

**Normative resolution**: All path builders canonicalize the candidate and root, then use a separator-aware `relative`/ancestor check; a sibling prefix is not containment and traversal or symlink escape fails closed.

**Focused verification gate**: Test root, descendant, sibling-prefix, `..`, symlink, and nonexistent-target cases; assert only canonical descendants are accepted.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 4. Thread `PRRT_kwDOTNj-ic6Ot8Se` — Include the redaction-pattern fields in this schema.

**Normative resolution**: The config schema includes the DESIGN §15 redaction-pattern fields with types, defaults, validation, and precedence, so the loader and redactor consume one complete contract.

**Focused verification gate**: Generate/validate the schema and load configs with defaults, valid patterns, invalid patterns, and unknown keys; assert the redactor receives the documented values.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 5. Thread `PRRT_kwDOTNj-ic6Ot8Sj` — Make workspace cleanup ownership explicit.

**Normative resolution**: The workspace manager owns removal of `workspaces/<id>`. `deleteMatch()` first verifies the match is terminal and delegates cleanup to that manager, with ordering and idempotency documented; the engine does not independently delete the directory.

**Focused verification gate**: Trace successful, failed, interrupted, and repeated deletion paths; assert the manager performs one safe cleanup and no caller races or double-deletes.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 6. Thread `PRRT_kwDOTNj-ic6Ot8Sl` — Include `git worktree list` in orphan detection.

**Normative resolution**: Startup orphan recovery reconciles the workspace index with `git worktree list --porcelain` before deleting anything; registered worktrees are retained or explicitly pruned through Git's supported operation.

**Focused verification gate**: Create indexed-only, Git-registered-only, matching, and stale worktree cases; assert no live Git worktree is deleted by a directory-only scan.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 7. Thread `PRRT_kwDOTNj-ic6Ot8Sp` — Keep the bundled fixture metadata parseable.

**Normative resolution**: Fixture coordination metadata is valid JSON data in the fixture schema (or the file is explicitly changed to JSONC with a parser); `.json` fixtures never rely on header comments for required metadata.

**Focused verification gate**: Parse every bundled fixture with the production parser and validate its metadata schema; assert malformed metadata fails before fixture execution.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 8. Thread `PRRT_kwDOTNj-ic6Ot8S2` — Keep `file_change.path` workspace-relative.

**Normative resolution**: The canonical `file_change` event contains a validated workspace-relative path only. Out-of-workspace edits are rejected or represented by a separate safe event without serializing an absolute host path.

**Focused verification gate**: Apply changes inside, outside, through traversal, and through symlinks; assert event schemas contain no absolute path or `..` escape.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 9. Thread `PRRT_kwDOTNj-ic6Ot8S5` — Don't derive file changes from `git status` on a dirty workspace.

**Normative resolution**: File-change events are derived from the applied patch/touched-path manifest or from a pre-run baseline diff, excluding unrelated pre-existing changes. `git status` may be diagnostic only, never the authoritative model-change source.

**Focused verification gate**: Start with unrelated dirty files, apply a patch touching a disjoint set, and assert only applied paths appear; test created/deleted/renamed paths.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 10. Thread `PRRT_kwDOTNj-ic6Ot8S8` — Redact the synthesized result summary.

**Normative resolution**: The result summary passes through the same redactor and single-line/length normalizer as raw events before storage, replay, or export; no raw last-message shortcut bypasses redaction.

**Focused verification gate**: Return secrets and control characters in the last message and assert summary, event, replay, and export are all redacted and bounded.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 11. Thread `PRRT_kwDOTNj-ic6Ot8S-` — Group raw events on the full feed, not per virtualized slice.

**Normative resolution**: `groupEvents` consumes the complete ordered event feed first, producing stable groups; UI virtualization is applied only to the grouped rows and never changes grouping or counts.

**Focused verification gate**: Render the same feed with different viewport/window sizes and scroll positions; assert group boundaries, collapsed counts, and order remain identical.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 12. Thread `PRRT_kwDOTNj-ic6Ot8TA` — Add an explicit fallback kind for parse failures.

**Normative resolution**: The change kind union includes a distinct `unparsed`/`parse_failed` discriminator with preserved bounded raw metadata. Malformed patches are never coerced into `binary` and the export viewer renders the fallback distinctly.

**Focused verification gate**: Feed valid text, binary, malformed, and truncated patches; assert schema validation and UI/export classification distinguish parse failure from binary.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 13. Thread `PRRT_kwDOTNj-ic6Ot8TD` — Align `run` with DESIGN §14.

**Normative resolution**: This issue adopts the canonical DESIGN §14 run surface and contestant cardinality exactly; unsupported `--keep-workspaces` and divergent 1-agent/2–4-agent rules are removed or explicitly reconciled in the canonical design before implementation.

**Focused verification gate**: Compare CLI help, parser, issue, and DESIGN §14; run 0, 1, 2–4, and >4-agent cases and assert one consistent contract.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 14. Thread `PRRT_kwDOTNj-ic6Ot8TF` — Don't freeze the npm package name before the owner gate.

**Normative resolution**: The package name remains a placeholder until the owner availability/approval gate passes. Packaging and release templates consume the approved name from one configuration source and cannot publish the provisional `agentarena` name automatically.

**Focused verification gate**: Run the flow before and after the owner gate, including an unavailable name; assert no package or publish metadata is emitted with an unapproved name.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 15. Thread `PRRT_kwDOTNj-ic6Ot8TM` — Document the concrete export hardening.

**Normative resolution**: `SECURITY.md` names the T6 controls: sanitized rendering, escaped embedded JSON (including `<` as `\u003c`), and strict CSP, plus the route/token boundary. “Export privacy warning” alone is not the security contract.

**Focused verification gate**: Cross-check SECURITY.md against the export renderer and CSP tests; assert each named control has a corresponding acceptance case.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 16. Thread `PRRT_kwDOTNj-ic6Ot814` — Don't promise no data leaves the machine.

**Normative resolution**: The FAQ distinguishes local UI/export storage from provider transmission: real matches may send prompts and selected repository content to configured API/CLI providers. It states the provider/privacy configuration boundary and makes no absolute no-upload promise.

**Focused verification gate**: Review FAQ copy against every adapter path and run a provider-transmission disclosure check; assert private-repository users see the accurate scope.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 17. Thread `PRRT_kwDOTNj-ic6Ot816` — Escape '<' to a real Unicode escape in export JSON.

**Normative resolution**: Before embedding transcript/export JSON in an HTML script element, escape `<` as `\u003c` (and apply the documented JSON/HTML-safe escaping) so `</script>` cannot terminate the data block; keep CSP strict.

**Focused verification gate**: Export content containing `</script>`, `<`, `&`, quotes, and control characters; assert the HTML parser cannot create executable markup and the viewer round-trips data.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 18. Thread `PRRT_kwDOTNj-ic6Ot817` — Reject sandbox-bypass flags in Codex extraArgs.

**Normative resolution**: The Codex argument validator rejects dangerous sandbox/approval bypass flags in both structured fields and `extraArgs` before process launch; only the fixed safe flags can be emitted.

**Focused verification gate**: Try exact and alias bypass flags in structured config and extraArgs, including reordered/duplicated forms; assert rejection and no child launch.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 19. Thread `PRRT_kwDOTNj-ic6Ot818` — Reap detached agent processes after crashes.

**Normative resolution**: The engine persists child PID/process-group identity and recovery state before launch. Crash recovery uses the platform reaper to signal, poll, and verify detached children before releasing/deleting workspaces; unsupported cleanup fails closed or is explicitly surfaced.

**Focused verification gate**: Kill the server during an active match with detached children, restart recovery, and assert processes are reaped before workspace cleanup and no stale spend/edit remains.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 20. Thread `PRRT_kwDOTNj-ic6Ot81-` — Redact captured diffs before writing diff.patch.

**Normative resolution**: The captured patch is passed through `redactor.redactText` before `store.putDiff` and before replay/export persistence; event redaction does not substitute for diff redaction.

**Focused verification gate**: Write token-shaped and multiline secret content into a changed file, capture the patch, and assert stored diff, replay, and export contain no secret material.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 21. Thread `PRRT_kwDOTNj-ic6Ot82A` — Include the query token when triggering downloads.

**Normative resolution**: The results-page download action uses the documented export route with a correctly encoded `?token=<getToken()>`, or uses a bearer-authenticated Blob fetch; navigation without authentication is not an accepted path.

**Focused verification gate**: Invoke download with valid, missing, expired, and special-character tokens; assert valid requests authenticate and tokens are not leaked to unrelated origins/logs.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 22. Thread `PRRT_kwDOTNj-ic6Ot82C` — Keep the `run --no-eval` CLI option in scope.

**Normative resolution**: The canonical run command either implements and tests `--no-eval` with the documented evaluation-skip semantics or removes it from DESIGN §14 and every dependent issue before implementation; this issue does not silently omit it.

**Focused verification gate**: Compare help/parser/docs and run with/without `--no-eval`; assert evaluation is skipped only when explicitly requested and default behavior is unchanged.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 23. Thread `PRRT_kwDOTNj-ic6Ot82E` — Implement the keepWorkspaces cleanup path.

**Normative resolution**: The match finish path consumes `keepWorkspaces`: when false it invokes the workspace manager's idempotent removal and updates `workspaceState`; when true it records retained state and never leaves ambiguous cleanup ownership.

**Focused verification gate**: Complete matches successfully, with errors, and after interruption under both values; assert workspace/index state and cleanup calls match the option.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 24. Thread `PRRT_kwDOTNj-ic6Ot82G` — Buffer match updates during WS backfill.

**Normative resolution**: The WebSocket session buffers every engine notification, including `match`, while historical backfill is sent, then emits a coherent backfill/live boundary and a current terminal snapshot so no terminal state is lost.

**Focused verification gate**: Delay backfill while match completion/evaluation notifications occur; assert the client receives ordered history plus a current terminal state and does not remain falsely running.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 25. Thread `PRRT_kwDOTNj-ic6Ot82I` — Create token files with restrictive mode up front.

**Normative resolution**: Token directories and temporary config files are created with restrictive modes at `mkdir`/exclusive `open` time, then atomically renamed; post-write `chmod` is defense in depth, not the first protection.

**Focused verification gate**: Use permissive umask and concurrent first-run creation, inspect every creation window, and assert no token/data path is ever group/world-readable.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.

## 26. Thread `PRRT_kwDOTNj-ic6Ot82K` — Align missing-command checks with bash semantics.

**Normative resolution**: The acceptance contract distinguishes shell execution (`bash -c` missing command => failed/127) from direct spawn failure (`error`). Tests and status mapping use the same execution mode and no longer expect an impossible `[... error]` result for `/nonexistent-bin` under bash.

**Focused verification gate**: Run both direct-exec and bash-shell missing commands; assert exact status, exit code, and error classification for each mode.

**Completion boundary**: this section is a contract for later implementation and full-validation gates, not evidence that those gates have passed.