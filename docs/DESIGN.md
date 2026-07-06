# agentarena — v1 Design

> 複数エージェントに同一タスクを解かせ観戦・比較する競技場
> A local arena where multiple coding agents solve the same task while you spectate and compare.

- Status: v1 design, approved decisions recorded in `docs/decisions/ADR-001…006`.
- Canonical source of truth: this file + `docs/ISSUE_PLAN.md` + `docs/issues/*.md`. GitHub Issues are derived artifacts.
- Research grounding: `docs/research/cli-agent-interfaces.md` (CLI facts), `docs/research/prior-art.md` (positioning).
- Language: English per ADR-006. 日本語のコンセプト一行は README が正。

## 1. Overview and goals

agentarena runs one coding task against N agent CLIs simultaneously, each in an isolated git worktree of the same repository, streams every agent's activity live to a browser UI, evaluates results with user-defined check commands, records a human verdict, and exports finished matches as self-contained HTML replays.

Product goals (v1):

1. **Spectate**: watch 2–4 real agent CLIs work side by side in real time (transcript, commands, file changes, cost ticker).
2. **Compare**: after the match, compare unified diffs, objective check results (build/test), duration, tokens/cost.
3. **Judge**: record a human vote (winner / tie / no-winner + notes) next to the objective scoreboard.
4. **Replay & share**: re-watch any match; export a match as one `.html` file that works offline.
5. **Trustworthy defaults**: safe-by-default local server, per-CLI strongest sandbox flags, no secrets persisted.

Design values: the event log is the product (everything renders from it); adapters are thin translators; weaker implementation agents must be able to build each issue mechanically.

## 2. Concepts and terminology

| Term | Definition |
|---|---|
| **Match** | One competition: one task + one repo@baseRef + N contestants. Identified by `matchId`. |
| **Contestant** | One agent instance in a match (adapter + optional model), with its own workspace and event log. |
| **Adapter** | Integration for one agent kind (`codex`, `claude-code`, `gemini`, `api-baseline`, `mock`). Spawns/drives the agent and translates its output into `AgentEvent`s. |
| **Task** | Markdown prompt + YAML frontmatter (title, timeout, checks). Snapshotted into the match at creation. |
| **Check** | A shell command run in a contestant's workspace after it finishes (e.g. `pnpm test`); exit 0 = passed. |
| **Workspace** | Detached git worktree per contestant, created from the match base commit. |
| **AgentEvent** | Normalized, versioned event record; the only event currency (ADR-004). |
| **Scoreboard** | Computed objective metrics table across contestants. |
| **Verdict** | Scoreboard + optional human vote. |
| **Replay bundle** | Self-contained JSON of a finished match (match + events + diffs + eval + verdict). |

## 3. Scope

### 3.1 v1 scope

- Task type: coding tasks in a local git repository (owner-authored prompts). One task per match.
- Contestants: 2–4 per match from adapters: `codex`, `claude-code`, `gemini`, `api-baseline`, `mock`.
- Live Web UI (localhost), replay view, single-file HTML export.
- Auto evaluation via task-defined checks; human vote; scoreboard.
- CLI: `serve` (default), `run` (headless), `ls`, `export`, `doctor`, `demo`.
- Platforms: macOS and Linux. Node.js >= 22.

### 3.2 v1 non-goals

- No hosted/multi-user mode, no authentication beyond the local token, no LAN exposure.
- No container isolation (interface reserved; ADR-002), no Windows-native support (WSL untested), no Windows CI.
- No generic (non-coding) task type; no multi-task suites; no cross-match leaderboard/ELO.
- No LLM-as-judge; no automatic winner declaration.
- No agent-to-agent interaction; contestants never see each other.
- No secrets management beyond redaction + env passthrough.

### 3.3 v2 deferred (explicit)

Container execution driver; Windows support; generic prompt tasks; task suites & batch statistics; cross-match leaderboards; LLM-as-judge; additional adapters (aider, opencode, cursor-agent, amp, …); hosted replay sharing; live cost budgets enforced arena-side; MCP-based adapter integrations.

### 3.4 Known unknowns

Tracked in `docs/ISSUE_PLAN.md` §"Known unknowns" (single list, kept current).

## 4. Architecture overview

### 4.1 Monorepo layout (ADR-001)

```
agentarena/
├── package.json                  # private root, pnpm workspaces
├── pnpm-workspace.yaml
├── tsconfig.base.json
├── packages/
│   ├── shared/    @agentarena/shared   # domain types, AgentEvent, zod schemas, API/WS protocol types, diff parser
│   ├── core/      @agentarena/core     # config, storage, process runner, workspaces, adapters, match engine, eval, redaction, export assembly
│   ├── server/    @agentarena/server   # Fastify HTTP + WS, static UI serving, security middleware
│   ├── web/       @agentarena/web      # React SPA (live/replay/results) + replay-standalone build target
│   └── cli/       agentarena           # published package: commander CLI, bundles core+server+web dist
└── docs/
```

All packages ESM-only, TypeScript strict. Only `agentarena` is published (internal packages bundled via tsup/esbuild).

### 4.2 Runtime topology

One Node process (`agentarena serve`) hosts: match engine (in-process), storage writer, Fastify HTTP+WS server, static SPA. Browser connects over `127.0.0.1`. Agent CLIs are child processes (one per contestant), each with `cwd` = its workspace. `agentarena run` uses core directly without the server.

### 4.3 Data flow

```
CLI child stdout(JSONL) ──parser──▶ adapter ──AgentEvent──▶ engine(seq, redaction)
    ├──▶ storage: append events.jsonl, update match.json (atomic)
    └──▶ WS hub ──▶ browser lanes (live)
contestant ends ──▶ diff capture ──▶ eval runner(checks) ──▶ eval.json ──▶ scoreboard ──▶ verdict.json
replay/export read storage only.
```

Ordering rule (ADR-003): events are appended to `events.jsonl` **before** the `match.json` update that summarizes them.

## 5. Domain model and state machines

All schemas live in `@agentarena/shared` as zod schemas + inferred TS types. All persisted documents carry `v: 1`.

### 5.1 Identifiers

| Id | Format | Regex |
|---|---|---|
| `matchId` | `m_` + ULID (26-char Crockford base32, uppercase) | `^m_[0-9A-HJKMNP-TV-Z]{26}$` |
| `contestantId` | `c_<adapterId>_<n>` (n = 1-based) | `^c_[a-z0-9-]+_[0-9]{1,2}$` |
| `adapterId` | fixed set v1 | `^(codex|claude-code|gemini|api-baseline|mock)$` |

### 5.2 MatchRecord (`match.json`)

```ts
interface MatchRecord {
  v: 1;
  id: string;                       // matchId
  title: string;                    // from task frontmatter or first prompt line
  status: MatchStatus;
  createdAt: string;                // ISO-8601 UTC, ms precision (all timestamps)
  startedAt?: string;
  finishedAt?: string;
  repo: { path: string; baseRef: string; baseSha: string };
  task: TaskSpec;                   // snapshot; prompt body stored as task.md
  promptTemplate: 'arena-v1';
  options: { timeoutSec: number; checkTimeoutSec: number; keepWorkspaces: boolean; maxParallel: number };
  contestants: ContestantRecord[];
  error?: string;                   // infrastructure failure reason (status=failed)
}

interface TaskSpec {
  title: string;
  checks: CheckSpec[];              // may be empty
}
interface CheckSpec { name: string; run: string; timeoutSec?: number }

interface ContestantRecord {
  id: string;
  adapterId: string;
  displayName: string;              // e.g. "Codex CLI", "Claude Code"
  model?: string;
  status: ContestantStatus;
  startedAt?: string;
  finishedAt?: string;
  durationMs?: number;
  exitCode?: number | null;         // null = killed by signal
  cliVersion?: string;              // captured by probe at match start
  lastSeq: number;                  // highest persisted event seq
  metrics?: MetricsSnapshot;        // last metrics event payload
  diffStats?: { filesChanged: number; insertions: number; deletions: number; truncated: boolean };
  evalSummary?: { passed: number; total: number };
  workspaceState: 'present' | 'removed';
  error?: string;
}
```

### 5.3 State machines

Match status transitions (anything else is a bug; transitions are validated centrally):

```
created ─▶ preparing ─▶ running ─▶ evaluating ─▶ finished
   │            │           │            │
   └──────┬─────┴─────┬─────┴──────┬─────┘
          ▼           ▼            ▼
        failed     canceled    canceled
(any non-terminal ─▶ interrupted: assigned only during crash-recovery scan at startup)
```

- `failed` = infrastructure error (workspace creation failed, storage error). Contestant failures do NOT fail a match.
- `canceled` = user cancel; running contestants are killed, then evaluation is skipped (`skipped` checks), verdict still computable.
- Terminal statuses: `finished`, `failed`, `canceled`, `interrupted`.

Contestant status transitions:

```
pending ─▶ preparing ─▶ running ─▶ completed   (process exit 0)
                          │  ├───▶ failed      (exit != 0, spawn error, adapter fatal error)
                          │  ├───▶ timeout     (wall-clock cap hit; process killed)
                          │  └───▶ canceled    (match canceled)
```

Checks run after a contestant reaches ANY terminal state except `canceled` (a timed-out agent may still have produced a useful diff). Match enters `evaluating` when all contestants are terminal and at least one eval is pending.

### 5.4 Evaluation record (`eval.json` per contestant)

```ts
interface EvalResult {
  v: 1;
  contestantId: string;
  startedAt: string; finishedAt: string;
  results: CheckResult[];
}
interface CheckResult {
  name: string;
  status: 'passed' | 'failed' | 'error' | 'timeout' | 'skipped';
  exitCode: number | null;
  durationMs: number;
  stdoutTail: string;   // last 32 KB, redacted
  stderrTail: string;   // last 32 KB, redacted
  truncated: boolean;
}
```

### 5.5 Verdict (`verdict.json` per match)

```ts
interface Verdict {
  v: 1;
  scoreboard: ScoreboardRow[];      // computed once when match reaches a terminal state
  computedAt: string;
  vote?: { winner: string | 'tie' | 'no-winner'; notes?: string; votedAt: string };  // winner = contestantId
}
interface ScoreboardRow {
  contestantId: string; adapterId: string; model?: string;
  status: ContestantStatus;
  checksPassed: number; checksTotal: number;
  durationMs?: number;
  inputTokens?: number; outputTokens?: number; costUsd?: number;
  filesChanged?: number; insertions?: number; deletions?: number;
  exitCode?: number | null;
}
```

No automatic winner: the UI presents the scoreboard and the vote separately.

## 6. Event model (ADR-004)

### 6.1 Envelope

```ts
interface AgentEvent {
  v: 1;
  seq: number;        // per-contestant, monotonic, starts at 1; assigned by the engine
  ts: string;         // ISO-8601 UTC ms; assigned by the engine at emit time
  type: AgentEventType;
  payload: …;         // per-type, below
}
```

Persisted as one JSON object per line in `contestants/<cid>/events.jsonl`. On the wire (WS) it is wrapped with `contestantId` (§10.3). `matchId`/`contestantId` are not duplicated inside the stored event (implied by file path).

### 6.2 Event types and payloads

| type | payload | Notes |
|---|---|---|
| `status` | `{ state: ContestantStatus, detail?: string }` | Emitted on every contestant transition. |
| `message` | `{ text: string, format: 'markdown' }` | Agent's user-facing message. |
| `reasoning` | `{ text: string }` | Model thinking summaries where the CLI exposes them. |
| `tool_call` | `{ callId?: string, name: string, input: Record<string, unknown> }` | Shell commands normalize to `name: "shell", input: { command }`. |
| `tool_result` | `{ callId?: string, name: string, output: string, exitCode?: number, durationMs?: number }` | `output` capped (§6.3). |
| `file_change` | `{ path: string, kind: 'created' | 'modified' | 'deleted' }` | One event per file. Paths relative to workspace root. |
| `metrics` | `MetricsSnapshot = { inputTokens?, cachedInputTokens?, outputTokens?, costUsd?, model? }` | Cumulative snapshot; last one wins. |
| `error` | `{ message: string, source: 'adapter' | 'process' | 'agent', severity: 'warn' | 'fatal' }` | `fatal` accompanies contestant failure. |
| `result` | `{ summary: string, outcome: 'completed' | 'failed' | 'timeout' | 'canceled', exitCode: number | null, durationMs: number }` | Exactly one per contestant, last event. |
| `raw` | `{ stream: 'stdout' | 'stderr', text: string }` | Lossless fallback for unmapped output (ADR-004 rule 3). |

Adapters MAY attach `payload.origin` (original parsed CLI object, ≤ 8 KB) on any event.

### 6.3 Size caps (enforced by the engine before persistence/broadcast)

| Field | Cap | Overflow behavior |
|---|---|---|
| `message.text`, `reasoning.text` | 64 KB | truncate + `payload.truncated: true` |
| `tool_result.output`, `raw.text` | 16 KB | head 8 KB + `…[truncated]…` + tail 8 KB |
| `payload.origin` | 8 KB | drop `origin` entirely |
| events per contestant | 50,000 | further events dropped except `status`/`result`/`error`; a single `error(warn)` records the drop |

## 7. Adapters

### 7.1 Interface (`@agentarena/core`)

```ts
interface AgentAdapter {
  id: string;                       // adapterId
  displayName: string;
  kind: 'cli' | 'api' | 'mock';
  probe(config: AdapterConfig): Promise<ProbeResult>;
  run(ctx: ContestantContext): Promise<RunOutcome>;   // resolves when the contestant is done
}
interface ProbeResult {
  available: boolean;
  version?: string;
  problems: string[];               // human-readable blockers/hints (e.g. "binary not found on PATH")
}
interface ContestantContext {
  matchId: string; contestantId: string;
  workspaceDir: string;             // absolute path to the worktree
  prompt: string;                   // final prompt (template arena-v1 already applied)
  adapterConfig: AdapterConfig;     // from config.json, validated
  model?: string;
  timeoutSignal: AbortSignal;       // fired by the engine on timeout/cancel
  emit(e: AdapterEmittedEvent): void; // AgentEvent minus v/seq/ts (engine assigns those)
  env: NodeJS.ProcessEnv;           // sanitized (§7.2)
}
interface RunOutcome { exitCode: number | null; outcome: 'completed' | 'failed' | 'timeout' | 'canceled' }
```

Rules for every adapter implementation:

1. Never emit nothing for output: unmapped lines → `raw` events.
2. Never call `process.exit`, never write outside `workspaceDir`.
3. Respect `timeoutSignal`: kill children (via process runner) and return `timeout`/`canceled`.
4. Deliver the prompt via stdin where the CLI supports it (avoids ARG_MAX and shell-quoting issues).
5. `probe()` must be side-effect free and complete in < 5 s.

### 7.2 Child-process contract (process runner, issue 06)

- Spawn with `detached: true` → own process group; kill = `SIGTERM` to the group, `SIGKILL` after 10 s grace.
- stdout consumed as UTF-8 lines (line length cap 1 MB); stderr captured to `raw` events (tail-capped).
- Child env = parent env minus `AGENTARENA_*` variables, plus `NO_COLOR=1`, cwd = workspace.
- Wall-clock timeout owned by the engine (default 900 s, per-task override).

### 7.3 Codex adapter (`codex`)

Verified facts: research/cli-agent-interfaces.md §1.

```
codex exec --json -C <workspaceDir> --sandbox workspace-write \
  --ignore-user-config --ignore-rules --ephemeral --color never \
  [-m <model>] -
# prompt written to stdin, then stdin closed
```

Event mapping:

| Codex JSONL | AgentEvent |
|---|---|
| `thread.started` | `status(running)` detail "thread started" |
| `item.completed` item `agent_message` | `message` |
| `item.*` item `reasoning` | `reasoning` |
| `item.started` item `command_execution` | `tool_call(shell)` |
| `item.completed` item `command_execution` | `tool_result(shell)` |
| `item.completed` item `file_change` | one `file_change` per file |
| `item.*` item `mcp_tool_call` / `web_search` / `plan_update` | `tool_call`/`tool_result` (`plan_update` → `message` with origin) |
| `turn.completed.usage` | `metrics` (tokens; costUsd omitted) |
| `turn.failed` | `error(fatal)` |
| unknown lines | `raw` |

Config keys: `command` (default `codex`), `defaultModel?`, `sandbox` (default `workspace-write`; value `danger-full-access` is rejected by config validation), `cleanProfile` (default true → the two `--ignore-*` flags), `extraArgs: string[]`.
Probe: `codex --version` parse; auth not probeable cheaply → `problems` hint only.

### 7.4 Claude Code adapter (`claude-code`)

Verified facts: research/cli-agent-interfaces.md §2.

```
claude -p --output-format stream-json --verbose \
  --permission-mode bypassPermissions --no-session-persistence \
  [--model <model>] [--max-budget-usd <cap>] [--setting-sources <sources>]
# cwd = workspaceDir; prompt via stdin
```

- MUST NOT use `--bare` by default (breaks OAuth; research §2). MUST NOT use `--dangerously-skip-permissions` (redundant with permission-mode).
- `--include-partial-messages` intentionally not used in v1 (message-level granularity suffices; halves event volume).

Event mapping:

| stream-json | AgentEvent |
|---|---|
| `system/init` | `status(running)` + `metrics{model}` |
| `system/api_retry` | `error(warn)` |
| `assistant` content `text` blocks | `message` |
| `assistant` content `thinking` blocks (if present) | `reasoning` |
| `assistant` content `tool_use` blocks | `tool_call` (Bash → name `shell`, input `{command}`) |
| `user` content `tool_result` blocks | `tool_result` |
| `result` | `metrics{costUsd: total_cost_usd, tokens from usage}` then adapter returns |
| unknown lines | `raw` |

`file_change` events are synthesized from `tool_use` blocks of Edit/Write/NotebookEdit (path from input).
Config keys: `command` (default `claude`), `defaultModel?`, `maxBudgetUsd?`, `settingSources?` (default unset = CLI default; documented fairness caveat), `settingsJson?` (passthrough to `--settings`, for users enabling Claude's native sandbox), `extraArgs`.
Probe: `claude --version`.

### 7.5 Gemini adapter (`gemini`)

Docs-derived, partially UNVERIFIED (research §3): parser must be fixture-driven and fall back hard to `raw`.

```
gemini --output-format stream-json --approval-mode yolo [-m <model>] [-s]
# cwd = workspaceDir; prompt via stdin (fallback: -p "<prompt>" if stdin headless proves unreliable)
```

Mapping: stream-json events → best-effort mapping established during implementation from captured fixtures; final `stats` object → `metrics`; `error` object → `error(fatal)`; everything unrecognized → `raw`.
Config keys: `command` (default `gemini`), `defaultModel?`, `sandbox` (default false), `extraArgs`.
Probe: `gemini --version`; must handle "not installed" gracefully (true on the reference machine today).

### 7.6 API baseline adapter (`api-baseline`)

Purpose: "raw model without an agent harness" reference contestant. One-shot (plus one repair round) HTTP call to an OpenAI-compatible chat-completions endpoint.

- Config: `{ baseUrl: string (default "https://api.openai.com/v1"), model: string, apiKeyEnv: string (default "OPENAI_API_KEY"), maxRepairRounds: 1, maxContextFiles: 30, temperature?: number }`. The key is read from env at run time and never persisted (§11.4).
- Context construction (deterministic): file tree from `git ls-files` (cap 400 paths), README head (first 4 KB) if present, full content of up to `maxContextFiles` files ≤ 24 KB each selected by a documented heuristic (paths mentioned in the task prompt first, then smallest source files), total context budget 192 KB.
- Instruction: reply with exactly one fenced ```diff block containing a unified diff against the current workspace.
- Apply: extract the first ```diff fence; `git apply --3way --whitespace=nowarn` in the workspace. On failure → one repair round (send apply error + rejected hunks). Still failing → `error(fatal)` + outcome `failed`.
- Events: `status`, `tool_call(name:"chat.completions", input:{model})`, `message` (model reply), `file_change` per applied file, `metrics` (usage tokens; costUsd omitted — no price table in v1), `result`.

### 7.7 Mock adapter (`mock`)

Replays a fixture file of `AdapterEmittedEvent`s with scripted delays and optional scripted file writes into the workspace (so diffs/checks are real). Drives demos, e2e, and UI development. Fixture format documented in issue 10; bundled fixtures live in the published package.

### 7.8 Fairness rules

1. Identical final prompt to all contestants (template `arena-v1`, §9.1) — recorded in `match.json`.
2. Same base commit, independent identical worktrees.
3. Clean-profile flags applied where the CLI supports them without breaking auth (codex: yes; claude/gemini: best-effort, documented per adapter).
4. The repo's own committed agent-config files (AGENTS.md, CLAUDE.md, .gemini/…) are part of the task environment and intentionally NOT stripped.
5. `cliVersion` recorded per contestant for reproducibility.

## 8. Storage layout (ADR-003)

Root: `~/.agentarena/` (override: `AGENTARENA_HOME` env).

```
~/.agentarena/
├── config.json                        # 0600; schema §15
├── matches/
│   └── m_<ULID>/
│       ├── match.json                 # MatchRecord, atomic writes
│       ├── task.md                    # snapshotted task source (frontmatter + body)
│       ├── verdict.json               # Verdict (created at terminal state)
│       └── contestants/
│           └── c_<adapter>_<n>/
│               ├── events.jsonl       # append-only AgentEvents
│               ├── diff.patch         # git diff --binary vs baseSha (cap 5 MB + truncated marker file)
│               └── eval.json          # EvalResult
└── workspaces/
    └── m_<ULID>/c_<adapter>_<n>/      # git worktrees; removed per keepWorkspaces policy / match deletion
```

Rules: atomic whole-document writes (tmp + rename, same dir); JSONL readers skip a torn final line; directory names are the ids (validated before any path join); deleting a match = `git worktree remove` for live worktrees (best-effort) + recursive delete of both dirs.

## 9. Task, prompt, evaluation, verdict

### 9.1 Task file format

```markdown
---
title: Fix the flaky retry test          # required
timeoutSec: 900                          # optional, default from config
checks:                                  # optional, ordered
  - name: typecheck
    run: pnpm tsc --noEmit
  - name: tests
    run: pnpm vitest run
    timeoutSec: 600
---
(The prompt body in Markdown. Delivered to every contestant verbatim.)
```

Prompt template `arena-v1` (exact, versioned constant in `@agentarena/shared`):

```
{taskBody}

---
Arena rules:
- Work only inside the current working directory (a dedicated git worktree).
- Do not push, do not create branches, do not rewrite git history. Commits are allowed but optional.
- When done, print a short summary of what you changed.
```

### 9.2 Evaluation runner

- Trigger: contestant reaches terminal state ≠ `canceled`. Checks are also recorded as `skipped` when the match was canceled first.
- Execution: sequential per contestant (checks in declared order), parallel across contestants (cap `options.maxParallel`).
- Each check: `bash -c <run>` with cwd = workspace, env = sanitized parent env + `AGENTARENA=1`, `CI=1`, `NO_COLOR=1`; timeout `check.timeoutSec ?? options.checkTimeoutSec` (default 600 s); kill = process-group SIGTERM→SIGKILL.
- `passed` = exit 0. `failed` = non-zero. `error` = spawn failure. `timeout` = cap hit.
- Checks are user-authored code and run unsandboxed by design (same trust domain as the user's own shell); the UI shows the exact commands before match start (§11.2 T4).

### 9.3 Diff capture (after contestant terminal, before checks)

In the workspace: `git add -A -N` (make untracked files visible to diff), then `git diff --binary <baseSha>` → `diff.patch` (5 MB cap + `truncated` flag), `git diff --numstat <baseSha>` → `diffStats`. Committed and uncommitted changes are both captured because the diff base is the fixed `baseSha`.

### 9.4 Scoreboard & vote

Scoreboard computed once when the match reaches a terminal state (rows per §5.5, sourced from ContestantRecord + eval.json). Vote: `PUT /api/matches/:id/vote`, overwritable, allowed only for terminal matches. No automatic winner.

## 10. Server: HTTP API and WS protocol

Fastify 5, all routes under `/api`, JSON bodies zod-validated, uniform error envelope `{ error: { code, message } }` with codes `unauthorized | forbidden | not_found | invalid_request | conflict | payload_too_large | internal`.

### 10.1 Security middleware order (ADR-005)

1. Host-allowlist check → 403 `forbidden`.
2. Token check for `/api/*` (`Authorization: Bearer` or `?token=` for WS/download URLs) → 401 `unauthorized`.
3. Origin check for WS upgrades and non-GET `/api/*` → 403.
4. Body size limit 1 MB (task creation), zod validation → 400.

### 10.2 REST endpoints

| Method & path | Body → Response | Notes |
|---|---|---|
| `GET /api/health` | → `{ ok: true, version, dataDir }` | |
| `GET /api/adapters` | → `{ adapters: (ProbeResult & {id, displayName, kind, defaultModel?})[] }` | probes cached 60 s |
| `POST /api/repo/inspect` | `{ path }` → `{ path, isGitRepo, currentRef, headSha, dirty }` | for the wizard; path must be absolute |
| `POST /api/matches` | `{ repoPath, baseRef?, taskMarkdown, contestants: [{adapterId, model?}], options? }` → `MatchRecord` (201) | validates, creates, and starts; 409 `conflict` if > `maxConcurrentMatches` (default 1) running |
| `GET /api/matches?limit=50&offset=0` | → `{ matches: MatchSummary[], total }` | newest first |
| `GET /api/matches/:id` | → `{ match: MatchRecord, verdict?: Verdict }` | |
| `POST /api/matches/:id/cancel` | → `{ match }` | idempotent |
| `DELETE /api/matches/:id` | → 204 | 409 while running unless `?force=1` (cancels first) |
| `GET /api/matches/:id/contestants/:cid/events?after=0&limit=1000` | → `{ events: AgentEvent[], nextAfter, done }` | replay/backfill paging |
| `GET /api/matches/:id/contestants/:cid/diff` | → `text/plain` patch | 404 if absent |
| `GET /api/matches/:id/eval` | → `{ evals: Record<contestantId, EvalResult> }` | |
| `PUT /api/matches/:id/vote` | `{ winner, notes? }` → `{ verdict }` | terminal matches only (409 otherwise) |
| `GET /api/matches/:id/replay` | → `ReplayBundle` JSON | terminal matches only |
| `GET /api/matches/:id/export.html` | → `text/html` (attachment) | terminal matches only |

### 10.3 WebSocket protocol

Endpoint: `GET /api/matches/:id/ws?token=…` (upgrade). Messages are JSON text frames.

```
client → server:  { "type": "subscribe", "since": { "<contestantId>": <seq>, … } }   // once, immediately after open
server → client:  { "type": "hello", "match": MatchRecord }
                  { "type": "event", "contestantId": "…", "event": AgentEvent }      // backfill (seq > since), in order, then live
                  { "type": "live" }                                                 // backfill complete marker
                  { "type": "match", "match": MatchRecord }                          // on every match/contestant state change (incl. eval, verdict)
                  { "type": "error", "code": "…", "message": "…" }                   // then close
```

Server pings every 30 s; drops connections with > 5 000 queued frames (slow consumer) — the client reconnects and backfills from its last seqs. Multiple concurrent subscribers per match are supported.

## 11. Security model

### 11.1 Trust boundaries and assets

- Assets: the user's repository contents, CLI auth material (keychains/OAuth/env keys), event logs & diffs (may embed code and accidentally-echoed secrets), exported replay HTML (leaves the machine).
- Trust domains: (1) the user + their shell + task files + check commands = trusted; (2) agent processes = **semi-trusted** (arbitrary code execution driven by an LLM); (3) the browser page = trusted once token-authenticated; (4) any other local process/web page = untrusted; (5) replay HTML viewers = untrusted environment (recipient's browser).

### 11.2 Threats and mitigations

| # | Threat | Mitigation |
|---|---|---|
| T1 | Malicious web page hits `127.0.0.1` API (CSRF / DNS rebinding) | ADR-005: token + Host allowlist + Origin check + no cookies + no CORS |
| T2 | Agent escapes its workspace / damages host | ADR-002: worktree isolation + strongest native CLI sandbox flags; dangerous CLI flags rejected by config validation; documented per-adapter isolation table; residual risk documented (Claude default) |
| T3 | Secrets leak into event logs / diffs / exports | Redaction filter at the engine emit boundary + eval tails + diff capture (issue 09); export command prints a content warning; best-effort limitation documented |
| T4 | Malicious task file / check command (if user pastes third-party tasks) | Checks/tasks are code: UI shows full commands pre-run; docs state "treat task files like shell scripts"; no task auto-fetch from URLs in v1 |
| T5 | Dependency / supply chain | Minimal pinned runtime deps, committed lockfile, no postinstall scripts, npm provenance publish, Dependabot, CI on PRs only from the repo |
| T6 | XSS via agent-controlled content (transcripts, diffs, file paths) in app or exported HTML | All rendering through React text nodes; markdown via react-markdown + rehype-sanitize (default schema, raw HTML disabled); ANSI stripped; export embeds JSON with `<` escaped (`<`) + CSP meta `default-src 'none'; script-src 'unsafe-inline'; style-src 'unsafe-inline'; img-src data:; font-src data:` |
| T7 | Path traversal via API params | Id-regex validation (§5.1) before any path join; resolved paths asserted under data-dir roots; `repoPath` must be absolute, existing, and a git repo |
| T8 | Resource exhaustion (event floods, giant diffs/outputs) | Caps §6.3, diff cap §9.3, WS slow-consumer drop §10.3, body limits §10.1, per-contestant event cap |
| T9 | Token theft from disk | config 0600; token never logged; docs: data dir is per-user private |

### 11.3 Input validation table (every external boundary)

| Boundary | Input | Validation |
|---|---|---|
| REST | all JSON bodies | zod schemas, strict (unknown keys rejected) |
| REST | `matchId`, `contestantId` | §5.1 regexes |
| REST | `repoPath` | absolute path, exists, `git rev-parse --git-dir` succeeds, not inside `AGENTARENA_HOME` |
| REST | `taskMarkdown` | ≤ 256 KB; frontmatter parsed with a YAML-safe loader (no custom types); checks ≤ 10; `run` ≤ 4 KB |
| WS | first frame | must be valid `subscribe` within 10 s or the socket closes |
| CLI child stdout | JSONL lines | try/catch parse per line; failures → `raw` |
| Config file | config.json | zod; unknown adapter keys warn; dangerous values (codex `danger-full-access`) rejected |
| Task frontmatter | `timeoutSec` etc. | bounded ranges (10 s ≤ timeout ≤ 7200 s) |
| Export/import | replay bundle | zod-validated on assembly; viewer treats all strings as text |

### 11.4 Secret handling rules

1. No secrets in any persisted file: API keys only via env (`apiKeyEnv` names the variable; the value is read at spawn time).
2. The redaction filter (issue 09) masks values of parent-env variables matching `/(KEY|TOKEN|SECRET|PASSWORD|CREDENTIAL)/i` (length ≥ 8) and a built-in pattern list (`sk-…`, `ghp_…`, `github_pat_…`, `AKIA…`, `xox…`, `eyJ…` JWT-shape) in every event string field, eval output tails, and diff text before persistence/broadcast.
3. The launch token (ADR-005) is not a secret-class credential but is treated like one: 0600 file, never in logs.
4. Setup instructions tell users to configure keys manually; agentarena never writes key material.

## 12. Web UI

React 19 + Vite + Tailwind CSS v4 + zustand + react-router (browser router; server falls back to `index.html` for non-`/api` GETs). Token bootstrap per ADR-005 (§ launch URL). WS client with auto-reconnect + `since`-based resume.

Routes & surfaces:

| Route | Surface | Content |
|---|---|---|
| `/` | Home | match list (status chips, title, adapters, date), delete, "New match", link to replay |
| `/new` | Wizard | repo path (+ `repo/inspect` feedback), task editor (title/timeout/checks form + prompt textarea), adapter picker with live probe status + model override, review step showing exact check commands (§11.2 T4), start |
| `/m/:id` | Live + results | lane grid (one lane per contestant: header = name/model/status/elapsed/token-cost ticker; body = virtualized event feed with per-type renderers; footer = follow toggle). When terminal: results tabs — Scoreboard (table + vote panel), Diffs (per-contestant file-collapsible diff viewer), Checks (matrix with output tails) |
| `/m/:id/replay` | Replay | same lane components driven by a virtual clock: timeline scrubber (by ts), play/pause, speed ×1/×2/×4/×8, jump-to-end |

Event renderers (issue 27) are pure components `AgentEvent → JSX`, shared by live, replay, and the standalone export viewer. Rendering safety per §11.2 T6.

## 13. Replay bundle and HTML export

```ts
interface ReplayBundle {
  v: 1;
  exportedAt: string;
  appVersion: string;
  match: MatchRecord;
  taskMarkdown: string;
  verdict?: Verdict;
  contestants: { record: ContestantRecord; events: AgentEvent[]; diff?: string; eval?: EvalResult }[];
}
```

Export pipeline (issue 31): at **package build time**, a dedicated Vite config builds `replay-standalone` (entry reusing lane/renderer/replay components, data source = embedded JSON) into one JS string + one CSS string, stored as assets inside the published package. At **export time**, the server/CLI assembles `template.html`: title, CSP meta (§11.2 T6), inline CSS, `<script type="application/json" id="agentarena-replay">` + escaped JSON, inline JS. Output is fully offline (no network requests; enforced by CSP). Size guard: warn > 20 MB. Export requires a terminal match.

## 14. CLI (`agentarena`)

commander-based; global flags `--data-dir`, `--port`. Commands:

| Command | Behavior | Exit codes |
|---|---|---|
| `agentarena` / `agentarena serve [--port] [--no-open]` | start server, print `http://127.0.0.1:<port>/?token=…`, open browser | 0 on clean shutdown (SIGINT) |
| `agentarena run --repo <path> --task <file> --agents <id[:model],…> [--json] [--no-eval]` | headless match via core (no server): stream one status line per contestant transition; final scoreboard table (or JSON) | 0 finished; 2 match failed/interrupted; 130 SIGINT (cancels match) |
| `agentarena ls [--json]` | list matches | 0 |
| `agentarena export <matchId> [-o out.html]` | write export HTML (default `./agentarena-<matchId>.html`), print privacy warning | 0 / 1 not found or not terminal |
| `agentarena doctor` | table: node/git versions, data dir writable, port free, per-adapter probe (version, problems) | 0 all pass; 1 any blocker |
| `agentarena demo [--no-open]` | create temp fixture repo + match with 3 mock contestants (bundled fixtures, ×4 speed), serve + open | 0 |

`run` never prompts; it is CI-usable with only the mock adapter (no keys needed).

## 15. Configuration (`~/.agentarena/config.json`)

```jsonc
{
  "v": 1,
  "token": "<64 hex chars, generated on first run>",
  "port": 7788,
  "defaults": {
    "contestantTimeoutSec": 900,
    "checkTimeoutSec": 600,
    "maxParallel": 4,
    "keepWorkspaces": true,
    "maxConcurrentMatches": 1
  },
  "adapters": {
    "codex":        { "command": "codex",  "defaultModel": null, "sandbox": "workspace-write", "cleanProfile": true, "extraArgs": [] },
    "claude-code":  { "command": "claude", "defaultModel": null, "maxBudgetUsd": null, "settingSources": null, "settingsJson": null, "extraArgs": [] },
    "gemini":       { "command": "gemini", "defaultModel": null, "sandbox": false, "extraArgs": [] },
    "api-baseline": { "baseUrl": "https://api.openai.com/v1", "model": "", "apiKeyEnv": "OPENAI_API_KEY", "maxRepairRounds": 1, "maxContextFiles": 30 },
    "mock":         { "speed": 1 }
  },
  "redaction": { "extraPatterns": [] }
}
```

zod-validated on load; missing file → created with defaults + fresh token; unknown keys → warning, not error (forward compat); dangerous values rejected (§11.3). `extraArgs` is an escape hatch executed verbatim — documented as user-owned risk.

## 16. Failure modes and edge cases (normative)

| # | Case | Required behavior |
|---|---|---|
| F1 | Adapter binary missing / probe fails at match creation | contestant enters `failed` immediately with `error(fatal)` "not available"; match proceeds with the rest (min 1 runnable contestant required, else 400) |
| F2 | CLI not authenticated (detected only at runtime as an error/exit) | normal contestant failure path; stderr surfaced via `raw`; doctor gives hints |
| F3 | CLI JSON format drift | unparsed lines → `raw` (ADR-004); match completes; parser fixtures updated in a patch release |
| F4 | Agent hangs | wall-clock timeout → group SIGTERM → SIGKILL → status `timeout`; diff+eval still run |
| F5 | Agent floods output | caps §6.3; event-count cap with drop notice |
| F6 | Target repo dirty | allowed; wizard shows `dirty: true` hint "uncommitted changes are not visible to contestants" (worktrees are created from baseSha) |
| F7 | Two matches on the same repo / concurrent worktree ops | per-repo mutex around `git worktree add/remove`; `maxConcurrentMatches` default 1 |
| F8 | Port in use | try next port up to +20, print chosen; `--port` exact-fail |
| F9 | Server crash / SIGKILL mid-match | on next start: non-terminal matches → `interrupted`; orphaned workspaces detected via `git worktree list` + directory scan, removed on match deletion |
| F10 | Torn JSONL tail after crash | readers skip the invalid final line |
| F11 | `git apply` failure (baseline adapter) | one repair round then `failed` (§7.6) |
| F12 | Worktree removal fails (locked files) | fallback `rm -rf` + `git worktree prune`; failure logged, match deletion still succeeds |
| F13 | Match deleted while WS subscribers attached | `{type:"error", code:"not_found"}` then close |
| F14 | Export of a match with truncated diffs/events | export succeeds; truncation markers visible in the viewer |
| F15 | Browser tab reconnects after sleep | WS resume via `since` map; no duplicate events (seq-deduped client-side) |

## 17. Testing strategy

| Layer | Tooling | What |
|---|---|---|
| Unit | vitest | zod schemas (valid/invalid), diff parser, redaction, storage (tmp dirs, torn-line), process runner (kill-tree, timeout), per-CLI parsers against **recorded fixtures**, prompt/context builders |
| Integration (node) | vitest | full match through core with mock adapters on a temp git repo: state transitions, events on disk, diff capture, eval, scoreboard; API+WS through Fastify `inject`/real socket |
| UI | vitest + @testing-library/react | event renderers (XSS strings render inert), scoreboard, wizard validation |
| E2E | Playwright (chromium) | `demo`-seeded server: lanes stream, results tabs, vote, replay scrubber, export downloads and opens as `file://` |
| CI | GitHub Actions | lint + typecheck + unit/integration on ubuntu & macos, Node 22; Playwright on ubuntu; no secrets in CI (mock only) |

Fixture policy: every CLI adapter issue includes capturing at least one real session fixture (JSONL) with secrets scrubbed, committed under `packages/core/src/adapters/<id>/fixtures/`.

## 18. Toolchain and dependency policy

- Node >= 22 (`engines`), pnpm >= 9, TypeScript strict, ESM only, eslint + prettier, vitest, tsup for the publishable bundle.
- Runtime deps (closed list for v1; additions require an ADR note): `fastify`, `@fastify/websocket`, `@fastify/static`, `zod`, `ulid`, `commander`, `yaml`, `open`, `strip-ansi`; web-only: `react`, `react-dom`, `react-router`, `zustand`, `react-markdown`, `remark-gfm`, `rehype-sanitize`, `@tanstack/react-virtual`, `tailwindcss`.
- No postinstall scripts anywhere; lockfile committed; Dependabot enabled; GitHub Actions pinned by commit SHA.

## 19. Traceability

Every section above maps to implementation issues in `docs/ISSUE_PLAN.md` (coverage table there). Behavior described only in prose here and not covered by an issue is a planning bug — report it against ISSUE_PLAN.md.
