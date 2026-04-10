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
~/.claude/projects/<project-path>/memory/
```

The directory name is your project's absolute path with slashes replaced by dashes. For example:

| Project location | Memory directory |
|-----------------|-----------------|
| `/Users/alex/myapp` | `~/.claude/projects/-Users-alex-myapp/memory/` |
| `/home/alex/work/api` | `~/.claude/projects/-home-alex-work-api/memory/` |

To find yours:

```bash
# Option 1: Compute it from your project path
PROJECT_DIR=$(pwd)
MEMORY_DIR="$HOME/.claude/projects/$(echo "$PROJECT_DIR" | tr '/' '-')/memory"
echo "$MEMORY_DIR"

# Option 2: List all project directories and look for yours
ls ~/.claude/projects/

# Option 3: Let Claude Code create it automatically
# Just start a Claude Code session in your project — it creates the directory on first use
```

The `MEMORY.md` file inside this directory is the index — Claude loads it at the start of every session. It's the table of contents for everything the AI knows about your project between sessions.

If you're using Claude Code with auto-memory enabled, this directory probably already exists. Open it. You'll likely find a flat list of memories with auto-generated names. That's the starting point.

---

## Create Your First Memory Directory

If the directory doesn't exist:

```bash
# Compute the path and create it
PROJECT_DIR=$(pwd)
MEMORY_DIR="$HOME/.claude/projects/$(echo "$PROJECT_DIR" | tr '/' '-')/memory"
mkdir -p "$MEMORY_DIR"
echo "Created: $MEMORY_DIR"
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

### Verify auto-memory is enabled

GuildHouse depends on Claude Code's auto-memory feature. Check that it's on:

```bash
# Look for the memory setting in your config
cat ~/.claude/settings.json 2>/dev/null | grep -i memory

# Or just check: does your memory directory exist and have files?
ls ~/.claude/projects/$(echo "$(pwd)" | tr '/' '-')/memory/ 2>/dev/null
```

If auto-memory is off, enable it in Claude Code settings or by running `/memory on` in a session.

### Start a new session

Start a new Claude Code session in your project. Verification checklist:

1. **Does it load?** — Claude should have access to your MEMORY.md content. You can verify by asking "what do you know about this project?"

2. **Does it reference memories naturally?** — When working in a relevant area, does Claude use the stored context without being prompted?

3. **Does it avoid hallucinating?** — Ask about something NOT in memory. Claude should say it doesn't know, not fabricate an answer from memory-adjacent content.

If it doesn't use the memory:
- Check that MEMORY.md is properly formatted
- Verify file paths in the index match actual filenames (case-sensitive)
- Ensure YAML frontmatter is valid (no tabs, proper `---` delimiters)
- Confirm the memory directory is in the right location under `~/.claude/projects/`
- Confirm auto-memory is enabled (see above)

---

## Adding Semantic Search (Recommended)

File-based memory is fast and free but can only find exact keyword matches. Semantic search finds meaning — "how did the architecture evolve?" matches documents about "system design decisions" even if they never use the word "architecture."

### Install QMD

[QMD](https://github.com/tobilu/qmd) is a local vector search engine over markdown files. It runs entirely on your machine — no API keys needed. Install it globally:

```bash
npm install -g @tobilu/qmd
```

Requires Node.js 18+. Verify it installed: `qmd --help`

### Configure a collection

Add your memory directory as a QMD collection by editing `~/.config/qmd/index.yml`:

```bash
# Create config directory if it doesn't exist
mkdir -p ~/.config/qmd

# Compute your memory path
MEMORY_DIR="$HOME/.claude/projects/$(echo "$(pwd)" | tr '/' '-')/memory"
echo "Your memory dir: $MEMORY_DIR"
```

Then add it to `~/.config/qmd/index.yml` (create the file if it doesn't exist):

```yaml
collections:
  my-project:
    path: /Users/you/path/from/above/memory
    pattern: "**/*.md"
```

Replace the `path` with the actual directory printed by the command above. You can add multiple collections (one per project, or a shared knowledge directory).

### Index and embed

```bash
# Index your markdown files
qmd update

# Generate vector embeddings (takes a few seconds)
qmd embed
```

### Verify it works

```bash
# Keyword search (fast, no embeddings needed)
qmd search "decisions"

# Semantic search (requires embeddings — run qmd embed first)
qmd query "what decisions were made recently"
```

If results come back, semantic search is working. Re-run `qmd update && qmd embed` whenever you add new memory files, or automate it with [session hooks](automation.md).

### Using semantic search from Claude Code

QMD runs as an MCP server that Claude Code can query directly. Add it to your project's `.mcp.json` file (in your project root):

```json
{
  "mcpServers": {
    "qmd": {
      "command": "qmd",
      "args": ["mcp"]
    }
  }
}
```

Or add it globally in `~/.claude/settings.json` under the `"mcpServers"` key (same format) so it's available in every project.

Once connected, Claude can search your memories by meaning using `qmd query` — this is the "Semantic Search" tier in the routing table.

### Other vector stores

QMD is what we use, but any embedding pipeline works. The routing table describes query shapes, not specific tools. If you prefer Chroma, Pinecone, or Obsidian + Smart Connections, substitute them for the semantic search tier.

---

## Adding the Routing Table to Your Project

The routing table is what makes GuildHouse more than organized files. It tells your AI assistant which retrieval system to use for each query shape.

### Where it goes

The routing table needs to be in a file your AI assistant reads at session start. Where you put it depends on how many projects you have:

**Single project — put it in your project's `CLAUDE.md`:**

```markdown
## Retrieval Routing

Route retrieval by query shape:

| Need | Route | Cost |
|------|-------|------|
| **Exact fact** (date, status, price) | KG with predicate filter | 15-67 tok |
| **Broad entity** ("What is X?") | KG + semantic search in parallel | ~1,875 tok |
| **Narrative** (history, context) | Semantic search only (lex+vec+hyde) | ~750 tok |
| **Relationship** (who connects to what) | Semantic search first → KG backfill | ~1,200 tok |
| **Recent activity** (what happened lately) | File memory (grep dates) → semantic search | ~1,100 tok |

**Fast-path:** KG exact fact with relevance ≥ 0.95 → stop. Semantic top_score ≥ 0.90 with empty KG → use semantic result directly.
```

**Multiple projects — put the routing table in a global rules file, keep project-specific context in each CLAUDE.md:**

Claude Code loads global rules from `~/.claude/rules/*.md` into every session. Create a file like `~/.claude/rules/guildhouse-routing.md` with the routing table, then each project's CLAUDE.md only needs:

```markdown
## GuildHouse (Retrieval Routing)

Routing table is in ~/.claude/rules/guildhouse-routing.md (loaded globally). This project:
- **QMD collections:** my-project (primary), shared-knowledge (cross-reference)
- **KG domain:** [entities relevant to this project]
- **Dominant shapes:** [which query shapes come up most here]
```

This way you update the routing logic in one place and it applies everywhere. Each project just declares what to route *to*.

**Other AI assistants:** The concept is the same — put the routing table wherever your assistant reads project instructions. Cursor uses `.cursorrules`, Windsurf uses `.windsurfrules`, Copilot uses `.github/copilot-instructions.md`. The routing table is plain markdown that works in any of these.

### How it works

There's no separate classification model. The routing table in your project instructions *is* the classifier — your AI assistant reads the query, pattern-matches it against the five shapes, and picks the route. See [The Routing Table](../reference/routing-table.md) for the full rules and decision logic.

### If you don't have a knowledge graph

Skip the KG column. The routing table still works with just two tiers:

| Need | Route (no KG) |
|------|---------------|
| **Exact fact** | Semantic search with specific query |
| **Broad entity** | Semantic search (lex+vec) |
| **Narrative** | Semantic search (lex+vec+hyde) |
| **Relationship** | Semantic search → scan for entity names |
| **Recent activity** | File memory (grep) → semantic search |

You lose temporal precision and exact-fact speed, but the routing pattern still works.

---

## Next Steps

Once your single-project memory is working well:

- **[Scaling Across Projects](multi-project.md)** — extend the pattern across your full workspace
- **[Automation & Hooks](automation.md)** — automate context loading and session capture
- **[AI Implementation Playbook](ai-implementation.md)** — hand this to your AI assistant to read
