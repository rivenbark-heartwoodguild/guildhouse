# Example Project — CLAUDE.md with GuildHouse Routing

This is an example of a `CLAUDE.md` file with the GuildHouse routing table integrated. Place your `CLAUDE.md` in your project's root directory — Claude Code reads it at the start of every session.

This example is for a fictional project called "Relay" (a notification platform). Replace the project-specific content with your own.

---

## Project Context

Relay is a real-time notification platform. WebSocket-based push notifications replacing polling.

**Stack:** Node.js, PostgreSQL, Redis, deployed on Fly.io

**Team:** 4 engineers (Dana, Marcus, Priya, Tomás)

---

## Retrieval Routing

Before significant actions (creating documents, making architecture decisions, choosing tools), route retrieval by query shape:

| Need | Route | Cost |
|------|-------|------|
| **Exact fact** (date, status, price) | KG with predicate filter + `current_only=true` | 15-67 tok |
| **Broad entity** ("What is X?") | KG (`limit=20`) + semantic search (`lex+vec`) in parallel | ~1,875 tok |
| **Narrative** (history, context) | Semantic search only (`lex+vec+hyde`) | ~750 tok |
| **Relationship** (who connects to what) | Semantic search first → extract entity names → KG backfill (`limit=5` per entity, max 2) | ~1,200 tok |
| **Recent activity** (what happened lately) | File memory (grep dates) first → feed into semantic search | ~1,100 tok |

**Fast-path:** KG exact fact with relevance ≥ 0.95 → stop, don't query other systems. Semantic search top_score ≥ 0.90 with empty KG → use semantic result directly.

**If you don't have a KG yet:** Skip the KG steps. Use semantic search for exact facts and broad entities too. Add a KG when you keep asking "when did X change?" and can't find the answer.

---

## Conventions

- Run `npm test` before committing
- Integration tests use real PostgreSQL (testcontainers), never mocked databases
- API versioning: URL-based (v1, v2), 6-month deprecation window
- All tenant-scoped queries must include tenant_id in the index
