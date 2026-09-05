# ADR-003: File-based storage (JSON + JSONL), no database in v1

- Status: Accepted (2026-07-07)
- Deciders: Fable (design), derived from product answers (local single-user tool)

## Context

Persistent data: match metadata, per-contestant append-only event logs (hundreds to tens of thousands of events), diffs, evaluation results, verdicts. Consumers: live WS tailing, replay reads, HTML export, match list. Single local user, no concurrent writers across processes (one server process owns the data dir). Candidates: SQLite (better-sqlite3 native module or node:sqlite), or plain files.

## Decision

1. Storage root `~/.agentarena/` (overridable via `AGENTARENA_HOME`), layout fixed in DESIGN.md §8.
2. Event logs are **append-only JSONL**, one file per contestant. Metadata documents (`match.json`, `eval.json`, `verdict.json`, `config.json`) are whole-document JSON written atomically (write temp file in same directory + `rename`).
3. Match listing = directory scan of `matches/` reading only `match.json` files (no index file to corrupt).
4. All documents carry a schema version field `v: 1`; readers reject/flag unknown major versions.
5. No SQL database in v1. Revisit only if cross-match queries (leaderboards) land in v2.

## Consequences

- Positive: zero native dependencies (better-sqlite3 prebuild issues avoided for `npx` distribution); JSONL maps 1:1 to the WS stream and to replay bundles; files are human-inspectable and trivially exportable; crash recovery is a scan for non-terminal `match.json` states.
- Negative: no transactional cross-file consistency — mitigated by ordering rules (events are appended before the `match.json` state that summarizes them) and by treating `events.jsonl` as the source of truth for per-contestant history; a truncated final line (crash mid-append) is tolerated and skipped by readers.
- Negative: cross-match aggregation is O(n) directory scan — acceptable at local scale (hundreds of matches).

## Alternatives considered

- better-sqlite3: robust queries, but a native module hurts `npx` cold-start portability.
- node:sqlite: still not stable across the supported Node range at design time.
