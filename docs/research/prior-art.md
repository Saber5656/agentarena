# Research: Prior art and positioning

- Date: 2026-07-07
- Purpose: position agentarena and justify the judging/spectating design in DESIGN.md §2 and §9.
- Nature: comparative context from well-known public projects; product details below are summarized from general knowledge and were not individually re-verified. Nothing here is load-bearing for implementation correctness.

## Comparable systems

| System | What it does | What agentarena takes from it | What agentarena does differently |
|---|---|---|---|
| SWE-bench (+ Verified) | Offline benchmark: agents fix real GitHub issues, scored by hidden tests | Objective scoring via tests; workspace-per-attempt isolation | agentarena is interactive and local: your repo, your task, watch it live; no fixed dataset |
| Aider polyglot / exercism-style benchmarks | Scripted harness runs an agent across many small tasks, pass/fail | Deterministic check commands as the objective metric | Single rich match with live spectating instead of batch statistics |
| LMArena (Chatbot Arena) | Crowd-sourced blind A/B voting on model responses; ELO ladder | Human vote as a first-class verdict alongside metrics | Local, single-user, agents-not-models, code-diff-centric comparison |
| Terminal-Bench and similar CLI-agent benchmarks | Evaluate agent CLIs on terminal tasks in containers | The idea that the *agent product* (CLI), not the raw model, is the unit under test | v1 uses worktree+native CLI sandboxes instead of mandatory containers (ADR-002) |
| CI matrix pattern (same job, N configs) | Fan out one job across variants and compare results | Match = fan-out of one task across N adapters | Adds live event streaming, replay, and human verdicts |

## Positioning statement

agentarena is a **local, spectator-first comparison harness for coding agents**: give the same task in the same repository to several real agent CLIs, watch them work side by side in a browser, then compare diffs, objective check results, cost/latency metrics, and record a human verdict — and export the whole match as a self-contained shareable HTML replay.

Not a benchmark suite (no dataset, no leaderboard aggregation in v1), not a router/orchestrator (agents do not cooperate), not a hosted service (localhost only in v1).

## Design consequences adopted

1. Both objective metrics and a human vote are recorded per match; neither alone decides a "winner" automatically (DESIGN.md §9).
2. The event log is the primary artifact (enables live view, replay, export, and future scoring) — append-only JSONL per contestant (ADR-003, ADR-004).
3. Shareable replays are a distribution mechanism for an OSS project (README-friendly demo without API keys via the mock adapter demo mode).
