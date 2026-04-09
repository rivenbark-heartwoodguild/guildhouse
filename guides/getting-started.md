# Getting Started — Single Project

GuildHouse is a hybrid retrieval architecture for AI memory. This guide covers the foundation: the `memory/` directory pattern inside Claude Code's auto-memory system.

---

## What GuildHouse Is

A structured set of markdown files with YAML frontmatter that Claude reads at session start. Not a database, not a plugin — just organized files that follow a schema.

The core idea: Claude Code already has a memory system. It stores markdown files and loads them at session start. GuildHouse gives that system structure so the AI can find what it needs, when it needs it, without loading everything into context.

---

## Where Memory Lives

Claude Code stores project memory at:

```
~/.claude/projects/<project-path-hash>/memory/
```

Each project gets its own directory. The `MEMORY.md` file is the index — Claude loads it at the start of every session. It's the table of contents for everything the AI knows about your project between sessions.

If you're using Claude Code with auto-memory enabled, this directory already exists. Open it. You'll probably find a flat list of memories with auto-generated names. That's the starting point.

---

## Create Your First Memory Directory

If the directory doesn't exist yet:

```bash
# Find your project's memory path
ls ~/.claude/projects/

# Look for the directory matching your project
# The path hash is derived from your project's absolute path
```

If it already exists (likely), the goal isn't creation — it's transformation. You're turning a flat list of memories into a semantically organized system that Claude can navigate efficiently.

---

## The MEMORY.md Index

This is your table of contents. Organize by semantic category, not chronologically:

```markdown
# Project Memory

## Active Feedback
- [No mocking databases](feedback-real-db-tests.md) — Integration tests hit real DB, not mocks
- [Terse responses](feedback-terse-responses.md) — No trailing summaries after completing tasks

## Infrastructure
- [Deploy config](reference-deploy-config.md) — CI/CD pipeline runs on GitHub Actions, deploys to Fly.io
- [Environment setup](reference-env-setup.md) — Node 20, pnpm, PostgreSQL 15

## Active Projects
- [Auth rewrite](project-auth-rewrite.md) — Session token storage migration, legal compliance deadline June 15
- [API v3](project-api-v3.md) — Breaking changes documented in CHANGELOG, 12 endpoints remaining
```

Rules for the index:

- Each line under 150 characters
- Keep total under 200 lines
- Use `[Title](file.md) — one-line hook` format consistently
- Group by semantic category: Feedback, Infrastructure, Projects, Sessions, Reference
- The hook after the dash should tell Claude *when* this memory is relevant, not just *what* it contains

---

## The Memory File Schema

Every memory file uses YAML frontmatter:

```yaml
---
name: Human-readable name
description: One-line description (used for relevance matching)
type: feedback | project | session | reference | user
---
```

### Types explained

**feedback** — Corrections and confirmations. "Don't do X because Y." "Yes, keep doing Z." These are the highest-value memories because they prevent repeated mistakes.

```yaml
---
name: No mocking databases
description: Integration tests must hit real database, not mocks
type: feedback
---

Patrick corrected test approach on 2026-03-15. Unit tests can mock
external services, but database tests must use a real PostgreSQL instance.
Reason: mock drift caused a production bug in the user table migration.
```

**project** — Current state of ongoing work. Team, status, decisions, blockers. These change frequently and need regular updates.

```yaml
---
name: Auth rewrite status
description: Session token migration project - current state and decisions
type: project
---

## Status: In Progress (Sprint 3 of estimated 4)

### Key Decisions
- Using Redis for session storage (decided 2026-03-01)
- JWT refresh tokens with 7-day expiry
- Migration is backward-compatible — old sessions valid for 30 days

### Remaining
- Rate limiting middleware
- Admin session revocation endpoint
- Load testing under concurrent auth
```

**session** — Outcomes from specific work sessions. Key decisions, insights, things that worked or didn't.

```yaml
---
name: Database indexing session
description: Query performance investigation - added composite indexes
type: session
---

Session 2026-03-20: Investigated slow dashboard queries.
- Root cause: missing composite index on (tenant_id, created_at)
- Added 3 indexes, query time dropped from 2.3s to 45ms
- Decision: all tenant-scoped queries must include tenant_id in index
```

**reference** — Pointers to where information lives. These rarely change and prevent the AI from searching in the wrong places.

```yaml
---
name: Bug tracking location
description: All bugs tracked in Linear, project ENG
type: reference
---

Bugs: Linear project "ENG", view "Active Bugs"
Feature requests: Linear project "ENG", view "Requests"
Docs: /docs directory, deployed to docs.example.com
API spec: /openapi/spec.yaml (OpenAPI 3.1)
```

**user** — Who the human is. Role, expertise, preferences. Helps the AI calibrate explanations and suggestions.

```yaml
---
name: Developer profile
description: Senior backend engineer, 8 years experience, prefers Go
type: user
---

Role: Senior backend engineer
Experience: 8 years, primarily Go and PostgreSQL
Preferences: Terse responses, no boilerplate explanations
Expertise: Distributed systems, database optimization
Less familiar with: Frontend frameworks, CSS layout
```

---

## Write Your First 5 Memories

Start with one of each type:

1. **1 feedback memory** — Something your AI keeps getting wrong. A correction you've made more than once. This is the highest-impact starting point.

2. **1 project memory** — Current state of your main project. What's in progress, what's decided, what's blocked.

3. **1 user memory** — Your role and expertise level. This helps the AI calibrate whether to explain concepts or skip to implementation.

4. **1 reference memory** — Where your important resources live. Bug tracker, docs, API specs, deployment dashboard.

5. **1 session memory** — A recent key decision and its rationale. The kind of thing you'd have to re-explain next session.

Name the files with type prefixes for easy scanning:

```
memory/
  MEMORY.md
  feedback-real-db-tests.md
  project-auth-rewrite.md
  user-developer-profile.md
  reference-resource-locations.md
  session-2026-03-20-indexing.md
```

---

## Test It

Start a new Claude Code session. Verification checklist:

1. **Does it load?** — Claude should have access to your MEMORY.md content. You can verify by asking "what do you know about this project?"

2. **Does it reference memories naturally?** — When working in a relevant area, does Claude use the stored context without being prompted?

3. **Does it avoid hallucinating?** — Ask about something NOT in memory. Claude should say it doesn't know, not fabricate an answer from memory-adjacent content.

If it doesn't use the memory:
- Check that MEMORY.md is properly formatted
- Verify file paths in the index match actual filenames
- Ensure YAML frontmatter is valid (no tabs, proper `---` delimiters)
- Confirm the memory directory is in the right location under `~/.claude/projects/`

---

## Next Steps

Once your single-project memory is working well, see [Scaling Across Multiple Projects](multi-project.md) to extend the pattern across your full workspace.
