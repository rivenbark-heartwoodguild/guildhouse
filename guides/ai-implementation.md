# AI Implementation Playbook

If you're an AI reading this to help your human set up GuildHouse — this is for you.

Your human wants better memory between sessions. You're going to build it. But before you create a single file, you need to understand what exists, what's missing, and what actually matters to this person.

---

## Pre-Implementation Due Diligence

### 1. What memory systems already exist?

Before creating anything, investigate the current state:

```bash
# Check for existing memory directories
ls ~/.claude/projects/

# Check for global behavior files
ls ~/.claude/rules/

# Check for CLAUDE.md in the project root
ls CLAUDE.md

# Look for existing memory files with frontmatter
find ~/.claude/projects/ -name "*.md" -path "*/memory/*" | head -20
```

If memory files already exist, read them. Understand the current structure before proposing changes. You're building on what's there, not replacing it.

### 2. What information gets lost between sessions?

Ask the human directly:

> "What do you find yourself re-explaining every session?"

If they're not sure, look for signals:
- Check recent git history — are there patterns of repeated work or reverted decisions?
- Look at CLAUDE.md — are there instructions that compensate for lost context? (e.g., "Always use pnpm, not npm" suggests this was corrected multiple times)
- Check if previous sessions left any notes or TODOs

### 3. What questions come up repeatedly?

Common categories of lost knowledge:
- **Project status** — What's in progress, what's blocked, what's done
- **Team and roles** — Who works on what, who to ask about what
- **Architecture decisions** — Why X was chosen over Y
- **Deploy and infrastructure** — How to ship, where things run
- **Style and conventions** — Naming, formatting, code patterns
- **Tool configuration** — Which tools, how they're set up, known quirks

Each of these is a memory candidate.

### 4. How many projects does the human work across?

This determines your strategy:

| Projects | Strategy |
|----------|----------|
| 1 | Focus entirely on `memory/` quality. Deep, well-structured memories. |
| 2-3 | Set up `memory/` for each, start identifying shared rules |
| 4+ | Prioritize `~/.claude/rules/` for shared behavior, lean project memories |

### 5. What's the current session flow?

Understand what already loads at session start:
- Does CLAUDE.md exist? What's in it?
- Are there hooks configured? (Check `~/.claude/settings.json`)
- Is there an existing session start/end process?

Understanding the existing flow prevents you from creating conflicts or duplicating what's already handled.

---

## Implementation Tasks (in order)

### Step 1: Create memory directory + MEMORY.md

If the directory doesn't exist:

```bash
mkdir -p ~/.claude/projects/<path-hash>/memory/
```

If it exists, audit the current state. Read every file. Categorize what you find:
- Still relevant? Keep and restructure.
- Stale? Archive or delete.
- Duplicate of CLAUDE.md? Delete — CLAUDE.md is the source of truth for project instructions.

Create or restructure MEMORY.md as the index:

```markdown
# Project Memory

## Active Feedback
- [Entry](file.md) — hook

## Infrastructure
- [Entry](file.md) — hook

## Active Projects
- [Entry](file.md) — hook
```

### Step 2: Seed 5-10 memories from what you already know

Pull from the current conversation and existing project artifacts:

| Source | Memory Type | Example |
|--------|------------|---------|
| Corrections the human has made | feedback | "Don't use semicolons — this project uses ASI" |
| Current project state | project | "API v3 migration: 8 of 20 endpoints complete" |
| Key recent decisions | session | "Chose SQLite over PostgreSQL for simplicity" |
| Where resources live | reference | "Designs in Figma workspace 'Product v2'" |
| Who the human is | user | "Frontend lead, 5 years React, learning Rust" |

Write each as a markdown file with proper frontmatter:

```yaml
---
name: Descriptive name
description: One-line description Claude uses for relevance matching
type: feedback
---

The actual content. Be specific. Include dates when relevant.
Include the WHY, not just the WHAT.
```

### Step 3: Organize by type, not chronologically

Group memories semantically in MEMORY.md. Good categories:

- **Active Feedback** — Corrections and confirmed preferences
- **Infrastructure** — Deploy, CI/CD, environment, tools
- **Active Projects** — Current work streams and their status
- **Reference** — Where things live, external resources
- **Team** — People, roles, responsibilities

Bad organization: sorting by date, sorting by file creation order, one giant list.

### Step 4: Set TTLs based on change velocity

Different memory types age at different rates. Use these as guidelines for when to review:

| Type | Review Interval | Rationale |
|------|----------------|-----------|
| feedback | 180 days | Corrections are durable — preferences don't change often |
| project | 14 days | Status changes fast — stale project state is worse than no state |
| session | 90 days | Decisions are semi-durable but context fades |
| reference | Evergreen | Resources rarely move — check annually |
| user | Evergreen | People don't change roles often — check quarterly |

You don't need to implement automated expiry. Just be aware: when you encounter a project memory that's 30 days old, verify it before acting on it.

### Step 5: Add frontmatter to every file

Every memory file needs at minimum:

```yaml
---
name: Human-readable name
description: One-line description (this is what Claude uses for relevance)
type: feedback | project | session | reference | user
---
```

The `description` field is critical. It's what determines whether Claude loads this memory for a given task. Make it specific:

- Bad: "Database stuff"
- Good: "PostgreSQL connection pooling config and known timeout issues"

### Step 6: Test

Start a new session and verify:

1. MEMORY.md loads at session start
2. Memories influence behavior without explicit prompting
3. Corrections from feedback memories are reflected in responses
4. Project status is referenced when relevant

If memories aren't being used, the descriptions may be too vague. Rewrite them to be more specific about WHEN the memory is relevant.

---

### Step 7: Set up checkpoints (recommended)

At natural milestones during sessions — significant task completed, topic switch, ~30 turns of conversation — write a checkpoint file capturing the current state:

- Current task
- Key decisions made
- Important state (what would be lost if context compressed)
- What's done
- What remains

Overwrite the same checkpoint file each time — you only need the latest state. This enables recovery after context compression or when resuming in a new session. See [Advanced Patterns](../reference/advanced-patterns.md#checkpoints) for the full format.

### Step 8: Consider hooks (optional)

If the human works daily, hooks that load memory at session start and capture transcripts at session end make the system self-maintaining. Without hooks, memory is passive files. With hooks, it becomes an active learning loop. See [Automation & Hooks](automation.md) for the full pattern.

---

## Things to Avoid

**Don't save code patterns.** The code itself is the source of truth. If Claude needs to know the codebase's patterns, it should read the code. Memory files that duplicate code patterns go stale immediately.

**Don't save git history.** `git log` is authoritative and always current. A memory about "what we did last sprint" is stale the moment the next sprint starts.

**Don't save debugging solutions.** The fix is in the code. The commit message has the context. A memory file about "how we fixed the auth bug" adds nothing that `git log --grep="auth bug"` doesn't provide.

**Don't save ephemeral task state.** "Currently working on X" belongs in the session, not in persistent memory. Use in-session tracking (scratchpad, task lists) for work-in-progress.

**Don't duplicate CLAUDE.md.** If it's in CLAUDE.md, don't also put it in memory. CLAUDE.md is the project instruction manual. Memory is for things that emerge during work, not things that are known upfront.

**Don't create a "memory of everything."** Be selective. The value of memory is not completeness — it's relevance. Every memory that loads is context window space that could be used for the actual task. Save what's surprising, non-obvious, or hard to re-derive.

**Don't save negative judgments about the human.** Memory is for being helpful. "User makes bad architecture choices" is not a memory — it's an opinion that will poison future interactions.

---

## The Golden Rule

**Save what would be lost between sessions and can't be derived from the codebase.**

Everything else is noise. A 10-file memory system where every file is relevant beats a 100-file system where Claude has to wade through stale noise to find what matters.

When in doubt about whether to save something, ask: "If this memory didn't exist, would Claude make a worse decision?" If the answer is no, don't save it.
