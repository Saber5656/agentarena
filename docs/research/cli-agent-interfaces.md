# Research: Non-interactive interfaces of target agent CLIs

- Date verified: 2026-07-07
- Purpose: ground the adapter designs in DESIGN.md §7 and issues 11–14.
- Verification method: local `--help` output on the developer machine plus official documentation. Facts that could **not** be verified are marked **UNVERIFIED** and appear in ISSUE_PLAN.md "Known unknowns".

Locally verified versions:

| CLI | Version verified | Locally installed |
|---|---|---|
| Codex CLI | codex-cli 0.141.0 | yes |
| Claude Code | 2.1.172 | yes |
| Gemini CLI | — (docs only) | **no** |

## 1. Codex CLI (`codex exec`)

Source: `codex exec --help` (0.141.0, local), https://developers.openai.com/codex/noninteractive

### Invocation

```
codex exec [OPTIONS] [PROMPT]
```

- Prompt as positional arg, or from stdin when the arg is `-` or absent with piped stdin.
- Relevant flags (all verified in local `--help`):
  - `--json` — print events to stdout as JSONL.
  - `-o, --output-last-message <FILE>` — write final agent message to a file.
  - `--output-schema <FILE>` — JSON Schema for the final response shape.
  - `-s, --sandbox <read-only|workspace-write|danger-full-access>` — sandbox policy for model-generated shell commands. Default in exec mode is read-only per official docs.
  - `-C, --cd <DIR>` — working root for the agent.
  - `--add-dir <DIR>` — extra writable directories.
  - `--skip-git-repo-check` — allow running outside a git repository (not needed by agentarena: worktrees are git repositories).
  - `-m, --model <MODEL>`; `-c key=value` config overrides.
  - `--ephemeral` — do not persist session files.
  - `--ignore-user-config` — do not load `$CODEX_HOME/config.toml`; **auth still uses `CODEX_HOME`** (help text explicit). This enables a "clean profile" fair run without breaking login.
  - `--ignore-rules` — skip user/project execpolicy `.rules` files.
  - `--color <always|never|auto>`.
  - `--dangerously-bypass-approvals-and-sandbox` — MUST NOT be used by agentarena defaults.

### `--json` event stream (JSONL, one JSON object per line)

Event types (official docs):

- `thread.started`, `turn.started`, `turn.completed`, `turn.failed`
- `item.started` / `item.completed` with item types:
  - `agent_message` (LLM response text)
  - `reasoning`
  - `command_execution` (shell command runs)
  - `file_change`
  - `mcp_tool_call`
  - `web_search`
  - `plan_update`
- `turn.completed` carries usage, e.g.:

```json
{"type":"turn.completed","usage":{"input_tokens":24763,"cached_input_tokens":24448,"output_tokens":122}}
```

**UNVERIFIED**: exact per-item payload field names (e.g. command string field, aggregated diff field of `file_change`). Must be captured as fixtures from a real run during issue 11 implementation.

### Resume / sessions

`codex exec resume --last` / `codex exec resume <SESSION_ID>` exist. Not used in v1 (matches are one-shot); `--ephemeral` is the default agentarena posture.

## 2. Claude Code (`claude -p`)

Source: `claude --help` (2.1.172, local), https://code.claude.com/docs/en/headless

### Invocation

```
claude -p [prompt] [flags]   # prompt via arg or stdin
```

Relevant flags (verified in local `--help`):

- `--output-format <text|json|stream-json>`; stream-json requires `--verbose`; optional `--include-partial-messages` for token deltas.
- `--permission-mode <acceptEdits|auto|bypassPermissions|default|dontAsk|plan>`.
- `--dangerously-skip-permissions` / `--allow-dangerously-skip-permissions`.
- `--allowedTools / --disallowedTools <tools...>` (permission rule syntax, e.g. `"Bash(git diff *)"`).
- `--tools <tools...>` — restrict the built-in tool set.
- `--model <alias-or-full-name>`, `--fallback-model` (print mode only).
- `--max-budget-usd <amount>` (print mode only) — hard cost cap, useful as an arena per-contestant budget.
- `--no-session-persistence` (print mode only).
- `--setting-sources <user,project,local>` — which settings sources to load.
- `--settings <file-or-json>` — extra settings (candidate carrier for native sandbox config).
- `--bare` — minimal mode. **Caveat verified in docs: bare mode never reads OAuth/keychain; auth must be `ANTHROPIC_API_KEY` or `apiKeyHelper`.** Subscription-OAuth users would lose auth under `--bare`, so agentarena MUST NOT default to it.
- In `-p` mode the workspace-trust dialog is skipped (help text explicit).
- Permission behavior in `-p` mode (docs): a tool call not covered by allow rules/permission mode **aborts the run** rather than prompting.

### `stream-json` event stream

- `system` messages with `subtype`: `init` (first event; model, tools, plugins), `api_retry`, `plugin_install`.
- `assistant` / `user` messages: wrap Anthropic-API-shaped `message` objects (content blocks including `text` and `tool_use` / `tool_result`).
- `stream_event` (only with `--include-partial-messages`): raw deltas, e.g. `.event.delta.type == "text_delta"`.
- `result` message: final; includes `total_cost_usd`, usage, session id; `--output-format json` variant returns a single object with `result`, `session_id`, cost fields.

**UNVERIFIED**: full field list of `result` (e.g. `num_turns`, `duration_ms`, `is_error`) on 2.1.x; capture fixtures during issue 12.

### Sandboxing note

Claude Code has a native sandbox feature configurable via settings; the exact settings schema to enable it from `--settings` JSON is **UNVERIFIED**. v1 default posture is `--permission-mode bypassPermissions` inside a dedicated git worktree, with the risk documented (see ADR-002) and a config passthrough for users who enable Claude's sandbox.

## 3. Gemini CLI (headless)

Source: https://geminicli.com/docs/cli/headless/ , https://google-gemini.github.io/gemini-cli/docs/cli/headless.html , PR google-gemini/gemini-cli#10883. **Not installed locally; all facts documentation-derived.**

### Invocation

- Headless mode triggers with `-p/--prompt "<text>"` or non-TTY stdin.
- `--output-format json` — single JSON object: `response`, `stats` (per-model `api`/`tokens`, `tools` totals, `files.totalLinesAdded/Removed`), `error` ({`type`, `message`, `code?`}).
- `--output-format stream-json` — real-time JSONL event stream (added in PR #10883). **UNVERIFIED: exact event schema.** Adapter must treat unknown lines as raw passthrough and rely on the final `stats` object when present.
- `--approval-mode <default|auto_edit|yolo>` — `yolo` auto-approves all tool calls (preferred over legacy `-y/--yolo`).
- `-m/--model`, `--include-directories`.
- `-s/--sandbox` exists per configuration docs (container- or seatbelt-based). **UNVERIFIED on macOS**; treat as optional config passthrough.

### Adapter consequences

- Prefer `--output-format stream-json`; degrade gracefully: unparseable stdout lines become `raw` events; synthesize `metrics` from final stats when available.
- `probe()` must handle "binary not installed" (locally true today) and report install guidance.

## 4. Cross-CLI comparison relevant to the adapter interface

| Concern | Codex | Claude Code | Gemini CLI |
|---|---|---|---|
| Non-interactive entry | `codex exec` | `claude -p` | `gemini -p` / non-TTY |
| Streaming JSON | `--json` (JSONL) | `--output-format stream-json --verbose` | `--output-format stream-json` |
| Full-auto in sandboxed dir | `--sandbox workspace-write` | `--permission-mode bypassPermissions` (no OS sandbox by default) | `--approval-mode yolo` (+ optional `--sandbox`) |
| Clean profile (fairness) | `--ignore-user-config --ignore-rules` (auth kept) | `--setting-sources` subset; `--bare` breaks OAuth → not default | UNVERIFIED; none documented |
| Cost/usage reporting | `turn.completed.usage` tokens | `result.total_cost_usd` + usage | `stats.models.*.tokens` |
| Cost cap | not documented | `--max-budget-usd` | not documented |
| Session persistence opt-out | `--ephemeral` | `--no-session-persistence` | not documented |
| Working dir selection | `-C <dir>` | process `cwd` | process `cwd` |

Design consequence: the normalized event model (DESIGN.md §6) needs at minimum `message`, `reasoning`, `tool_call`, `tool_result`, `file_change`, `metrics`, `error`, `result`, `status`, and a lossless `raw` fallback. All three CLIs map onto it; per-CLI parsers live in their adapters.
