# ADR-002: Contestant isolation = git worktree + each CLI's native sandbox (containers deferred)

- Status: Accepted (2026-07-07, confirmed by product owner)
- Deciders: product owner (Q&A session), Fable (design)

## Context

Matches run agents fully autonomously (no approval prompts). Agents execute arbitrary shell commands. Isolation options: (a) git worktree isolation + each CLI's own sandbox flags, (b) mandatory containers, (c) both drivers in v1. Container mandatory would require solving CLI auth handoff (OAuth tokens, keychains) into containers and slow down v1 substantially.

## Decision

1. v1 execution driver: one **git worktree per contestant** (detached, created from the match base ref, located under `~/.agentarena/workspaces/<matchId>/<contestantId>/`, i.e. outside the user's repo).
2. Each adapter passes the **strongest non-interactive sandbox its CLI natively offers**:
   - Codex: `--sandbox workspace-write` (never `danger-full-access`, never `--dangerously-bypass-approvals-and-sandbox`).
   - Gemini: `--approval-mode yolo`, optional `--sandbox` passthrough when the user enables it.
   - Claude Code: `--permission-mode bypassPermissions` — **no OS-level sandbox by default**; users may pass native sandbox settings through adapter config (`--settings` JSON) once verified.
3. The execution driver is an interface (`ExecutionDriver`) with exactly one v1 implementation (`worktree`). A `container` driver is a declared v2 item; no container code in v1.
4. The trust model is documented user-facing (SECURITY.md + README): *run matches only on repositories you trust, with task prompts you wrote; isolation strength differs per adapter* (comparison table maintained in docs).

## Consequences

- Positive: v1 stays light and fast on macOS; users keep their existing CLI logins; worktrees guarantee the user's checkout is never touched and diffs are cleanly capturable.
- Negative: host protection is delegated to per-CLI sandboxes and is **not uniform** — Claude Code's default posture is the weakest link and is explicitly documented as such; defense is not layered.
- Follow-ups: per-adapter isolation table is a required section in SECURITY.md (issue 35); container driver is the first listed v2 item.

## Alternatives considered

- Mandatory Docker: uniform strong isolation, but auth handoff design (OAuth/keychain into containers), image maintenance, and Apple Silicon overhead push v1 out substantially.
- Both drivers in v1: doubles the test matrix; rejected for scope.
