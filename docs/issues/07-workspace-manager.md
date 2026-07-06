# Title

Workspace manager: worktree lifecycle, per-repo mutex, diff capture

## Summary

Implement git worktree isolation in `@agentarena/core`: repo preconditions, base-ref resolution, per-contestant worktree create/remove serialized per repo, diff + numstat capture against the base SHA, and orphan cleanup, per DESIGN.md §9.3 and ADR-002.

## Context

One detached worktree per contestant under `workspaces/<matchId>/<cid>/` guarantees the user's checkout is untouched and diffs are computable against a fixed `baseSha` even if the agent commits (DESIGN.md §7, §9.3, §16 F6/F7/F12).

## Scope

`packages/core/src/workspace/`:

- `git.ts`: thin `runGit(repoOrDir, args, {timeoutMs=60_000})` over the process runner (issue 06); typed `GitError` including stderr.
- `inspect.ts`: `inspectRepo(path)` → `{ path: resolvedRealpath, isGitRepo, currentRef, headSha, dirty }` using `git rev-parse --git-dir`, `rev-parse --abbrev-ref HEAD`, `rev-parse HEAD`, `status --porcelain` (dirty = non-empty). Non-repo → `isGitRepo: false` (no throw). Powers `POST /api/repo/inspect`.
- `mutex.ts`: `withRepoLock(repoRealpath, fn)` — in-process async mutex keyed by realpath (DESIGN.md F7).
- `manager.ts`:
  - `resolveBase(repoPath, baseRef?)` → `{ baseRef, baseSha }` via `git rev-parse --verify <ref>^{commit}` (default ref: `HEAD`).
  - `createWorkspace({repoPath, baseSha, matchId, contestantId, dataDir})` → creates parent dirs, `git -C <repo> worktree add --detach <wsPath> <baseSha>` under the repo lock; returns absolute `wsPath`.
  - `captureDiff(wsPath, baseSha)` → in the worktree: `git add -A -N`, then `git diff --binary <baseSha>` (capped at 5 MB → `{patch, truncated}`), then `git diff --numstat <baseSha>` parsed to `{filesChanged, insertions, deletions}` (binary `-` entries count as 0/0 but count the file). Cap applies to patch text only; numstat always computed.
  - `removeWorkspace(repoPath, wsPath)` → under repo lock: `git worktree remove --force <wsPath>`; on failure: `rm -rf` the dir + `git worktree prune` (DESIGN.md F12); never throws on cleanup failure — returns `{ok, warning?}`.
  - `removeMatchWorkspaces(repoPath, dataDir, matchId)` — iterate contestant dirs.
  - `cleanOrphans(dataDir, knownMatchIds)` — directories under `workspaces/` whose match no longer exists → best-effort remove (called at startup after issue 05's recovery scan).

## Detailed Requirements

1. All paths absolute; `wsPath` built via issue 04 path builders (id validation included).
2. `captureDiff` runs `git` with `cwd = wsPath` and never touches the origin repo index.
3. Numstat parser handles renames (`{old => new}` paths) and binary markers.
4. Diff cap measured in bytes on the patch text; truncation appends `\n# [agentarena] diff truncated at 5 MB\n` and sets the flag.
5. Concurrency test-proof: parallel `createWorkspace` calls for 4 contestants on one repo must all succeed (serialized internally).

## Acceptance Criteria

- [ ] Tests build a scratch repo (init, commit files) in tmp: `inspectRepo` reports clean/dirty correctly before/after touching a file.
- [ ] `createWorkspace` yields a worktree whose `git rev-parse HEAD` equals baseSha; user repo `git status` stays clean.
- [ ] Modify + add untracked + delete files in the worktree → `captureDiff` patch contains all three; numstat matches; committed-then-modified scenario (agent commits) also fully captured vs baseSha.
- [ ] 4 parallel `createWorkspace` on one repo: all succeed (mutex prevents `worktree add` races).
- [ ] `removeWorkspace` after planting an un-removable state (e.g. delete `.git` file inside worktree first) still returns `{ok: true|false, warning}` without throwing and the directory is gone.
- [ ] `resolveBase` with a bad ref → typed error naming the ref.
- [ ] Patch > 5 MB (generate big file) → `truncated: true`, marker line present.

## Validation

`pnpm --filter @agentarena/core test` (creates real git repos in tmp; requires `git` on PATH — CI has it).

## Dependencies

01, 04, 06.

## Non-goals

Container driver (v2, ADR-002), storage of diff.patch (issue 05 API, called by engine in issue 15), repo-path API validation (issue 19).

## Design References

DESIGN.md §7 intro, §9.3, §16 F6/F7/F9/F12; ADR-002.
