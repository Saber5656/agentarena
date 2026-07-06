# ADR-006: Documentation language policy

- Status: Accepted (2026-07-07)
- Deciders: Fable (design), per repository-owner conventions

## Context

The repository is intended to become a public OSS project. GitHub Issues must be written in English (owner's working agreement). `docs/issues/*.md` are issue drafts and therefore English. `docs/DESIGN.md` is cross-referenced by those English issues (section anchors, schema names). The README is currently a one-line Japanese concept statement.

## Decision

1. `docs/DESIGN.md`, `docs/ISSUE_PLAN.md`, `docs/issues/*.md`, `docs/decisions/*.md`, `docs/research/*.md`: **English** — they form one mutually-referencing document graph with the English GitHub Issues.
2. `README.md`: bilingual-friendly — English primary once the project is published; the existing Japanese concept line is preserved until the README rewrite (issue 35).
3. Identifiers, commands, schemas, and file paths are exact and never translated.
4. Conversational work with the repository owner remains in Japanese; that policy lives outside this repo.

## Consequences

- English-only contributors can consume the full design/issue graph; issue drafts can be pasted to GitHub verbatim.
- The Japanese one-liner in README stays authoritative for product intent until issue 35 replaces it with a bilingual intro.
