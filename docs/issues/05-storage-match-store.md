# Title

Match store: atomic JSON documents, append-only JSONL event logs, listing, deletion, recovery scan

## Summary

Implement the file-based persistence layer in `@agentarena/core` over the layout in DESIGN.md §8: atomic `match.json`/`verdict.json`/`eval.json` writes, `events.jsonl` append + ranged read, match listing/deletion, and the startup crash-recovery scan that marks non-terminal matches `interrupted`.

## Context

ADR-003 fixes file-based storage with strict ordering (events appended before the match.json that summarizes them) and torn-tail tolerance. This module is the only code that touches `matches/`.

## Scope

`packages/core/src/storage/`:

- `documents.ts`: `writeJsonAtomic(file, obj)` (tmp + rename, same dir, fsync file), `readJson(file, schema)` returning zod-validated data or typed `StorageError`.
- `matchStore.ts`:
  - `createMatch(record: MatchRecord)` — creates dirs, writes `match.json` + `task.md` (taskMarkdown passed alongside).
  - `updateMatch(id, mutate: (m) => m)` — read-modify-write, serialized per match via an in-process promise queue (no cross-process locking needed; single server process owns the dir, ADR-003).
  - `getMatch(id)`, `getVerdict(id)`, `putVerdict(id, v)`, `getEval(id, cid)`, `putEval(id, cid, e)`, `getDiff(id, cid)` / `putDiff(id, cid, patchText, truncated)` (writes `diff.patch` and, when truncated, `diff.patch.truncated` marker file), `getTaskMarkdown(id)`.
  - `listMatches({limit, offset})` — scan `matches/*/match.json`, newest-first by `createdAt`, returns `MatchSummary` (id, title, status, createdAt, adapterIds) + total.
  - `deleteMatch(id)` — recursive delete of match dir and workspace dir (workspace git cleanup is issue 07's `removeWorkspaces`, invoked by the engine before this).
- `eventLog.ts`:
  - `EventLogWriter(matchId, cid)` — `append(e: AgentEvent)` serializing one line + `\n`; an internal write queue guarantees line atomicity and order; `flush()`; `close()`.
  - `readEvents(id, cid, {afterSeq, limit})` — streaming line reader that: skips a torn final line (DESIGN.md F10), skips lines failing `AgentEventSchema` (counted, surfaced as `{skipped}`), returns `{events, nextAfter, done}`.
- `recovery.ts`: `recoverInterrupted(dataDir)` — for every match with non-terminal status: set status `interrupted`, set every non-terminal contestant to `failed` with `error: "interrupted by restart"`; returns the affected ids (DESIGN.md F9; orphaned-worktree cleanup itself is issue 07).

## Detailed Requirements

1. All ids validated via shared regexes before path construction (paths built only through issue 04 builders).
2. `readEvents` must not load the whole file for a tail read: implement forward streaming with early stop at `limit` (full-file scan acceptable v1, but memory O(limit), not O(file)).
3. Append path: at least `fs.appendFile` per line batch; keep an open FileHandle per active writer, closed on `close()`.
4. `listMatches` tolerates junk dirs (non-matching names, unreadable match.json → skipped with warning list in the result).
5. Everything async; no sync fs.

## Acceptance Criteria

- [ ] Unit tests on tmp dirs: create → get round-trip equals input (deep); update serializes 20 concurrent `updateMatch` calls without lost writes (counter field test).
- [ ] Append 10 000 events then `readEvents(afterSeq: 9990, limit: 5)` returns seqs 9991–9995, `done: false`; `afterSeq: 9995` → 5 events, `done: true`.
- [ ] File with torn last line (write partial JSON manually) → `readEvents` returns all whole lines, `skipped` count 0 for torn tail (tail skip is silent per ADR-003), no throw.
- [ ] A line with invalid schema mid-file → skipped and counted.
- [ ] `recoverInterrupted` on a fixture dir with statuses (`running`, `finished`, `created`) marks exactly the non-terminal ones `interrupted` and their contestants `failed`.
- [ ] `deleteMatch` removes both `matches/<id>` and `workspaces/<id>` trees; deleting a nonexistent id → typed not-found error.
- [ ] `listMatches` ignores a planted `matches/junk/` dir and still returns valid matches.

## Validation

`pnpm --filter @agentarena/core test`; run the 10k-event test under `--reporter=verbose` to confirm memory sanity (no assertion, reviewer judgment).

## Dependencies

01, 02, 03, 04.

## Non-goals

Redaction (issue 09 — store receives already-redacted events), WS streaming (issue 20), workspace git operations (issue 07).

## Design References

DESIGN.md §5.2, §8, §16 F9/F10; ADR-003.
