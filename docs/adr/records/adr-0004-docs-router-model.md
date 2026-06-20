# ADR-0004: Documentation Router Model

## Status

Accepted

## Context

Agent-facing root docs had grown into long manuals. Long root prompts waste context and become stale. The repository needs concise routing and task-specific docs.

## Decision

Keep `AGENTS.md` and `CLAUDE.md` short. Use `docs/INDEX.md` as the primary map, and put detailed Why/contract material in focused docs. Treat historical plans/reports as archives with current routing headers.

## Consequences

- Agents should reach relevant docs within three hops.
- New docs need index entries.
- Archive Markdown remains useful but is not canonical current state.

## Anchors

- `AGENTS.md`
- `CLAUDE.md`
- `docs/INDEX.md`
- `docs/00-docs-list.md`
