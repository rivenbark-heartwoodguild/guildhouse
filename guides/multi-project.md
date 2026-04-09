# Scaling Across Multiple Projects

You work across 3-5 repos. Each has its own `memory/` directory. Some knowledge is project-specific, some crosses projects. This guide covers maintaining memory coherence at scale.

---

## The Problem

Project A knows you prefer terse responses. Project B doesn't. Project C has a memory about your deploy process that's identical to Project A's. You're duplicating knowledge, and the copies drift out of sync.

The root issue: without a layering strategy, every project becomes an island. Cross-project knowledge gets duplicated, corrections don't propagate, and you end up re-teaching the same preferences in every repo.

---

## One Memory Directory Per Project

Claude Code already handles this. Each project gets its own memory at:

```
~/.claude/projects/<path-hash>/memory/
```

The key insight: **project memory should contain only what's specific to THAT project.** Don't duplicate cross-project knowledge. If it's true everywhere, it doesn't belong in a single project's memory.

What belongs in project memory:
- Project-specific architecture decisions
- Team members and their roles on THIS project
- Current sprint/milestone status
- Tech stack details unique to this repo
- Bugs, quirks, and workarounds specific to this codebase

What does NOT belong in project memory:
- Your communication preferences (terse, no emojis, etc.)
- General tool preferences (vim keybindings, terminal choice)
- Decision frameworks you use across all work
- Your role and expertise level

---

## Shared Rules for Cross-Project Behavior

`~/.claude/rules/*.md` files apply to ALL projects. This is where cross-cutting knowledge lives:

```
~/.claude/rules/
  coding-style.md        # Terse responses, no boilerplate
  tool-preferences.md    # Prefer pnpm over npm, use rg not grep
  review-standards.md    # Always check for N+1 queries, etc.
  decision-frameworks.md # How you evaluate trade-offs
```

Rules files are behavioral directives — they tell Claude HOW to work, not WHAT the project is. Think of them as personality configuration that persists across every session in every repo.

---

## The Layering Strategy

```
~/.claude/rules/              <-- Global behavior (all projects)
~/.claude/projects/A/memory/  <-- Project A knowledge
~/.claude/projects/B/memory/  <-- Project B knowledge
~/.claude/projects/C/memory/  <-- Project C knowledge
```

When Claude starts a session in Project A, it loads:
1. Global rules (from `~/.claude/rules/`)
2. Project A's MEMORY.md and referenced files
3. Project A's CLAUDE.md (if present in the repo root)

The layering means you write preferences once, and they apply everywhere. Project memory only carries what's unique.

**The test:** If you find yourself copying the same memory into multiple projects, it belongs in `rules/`, not `memory/`.

---

## Deployment Strategy

Don't try to set up all projects at once. Sequential deployment with learning between each:

### Phase 1: Flagship Project

Pick your most active project. This is where you'll iterate on the memory structure.

- Create 5-10 well-structured memories (see [Getting Started](getting-started.md))
- Organize MEMORY.md with semantic categories
- Run for 1-2 weeks
- Note which memories get used, which get ignored
- Note which corrections you make that should apply everywhere

### Phase 2: Extract Global Rules

After Phase 1, you'll have identified cross-project patterns. Move them:

```bash
# Example: you kept correcting "no trailing summaries" in Project A
# That's a global preference, not a project fact
# Move it to ~/.claude/rules/communication-style.md
```

Common rules that emerge:
- Response style (length, tone, formatting)
- Code review expectations
- Testing philosophy
- Tool preferences
- Error handling patterns

### Phase 3: Expand to Project 2

- Copy the MEMORY.md *structure* (section headings), not the content
- Seed with 3-5 project-specific memories
- Global rules already apply — you don't need to re-teach preferences

### Phase 4: Remaining Projects

- Same structure, project-specific content
- Smaller projects may only need 2-3 memories
- That's fine — they benefit from global rules more than their own memory

---

## How We Did It

We deployed across 6 projects over 3 weeks. Here's what we learned:

**The flagship had 52 memory files.** It was the most complex project with the most domain knowledge to preserve. The MEMORY.md index was organized into 8 sections: Feedback, Infrastructure, Active Projects, Deliverables, Tools, Team, Operations, and Reference.

**The second project had 9 files.** Mostly project-specific architecture decisions and a few key session outcomes.

**The smallest projects had 2-3 files each.** A project status memory and one or two specific technical decisions. They benefited more from the 12 global rules files than from their own memories.

**What surprised us:** The act of organizing memories forced clarity about what was actually important. Half the flat-file memories in the flagship turned out to be duplicates or stale. The migration was as much a cleanup as a restructuring.

**What we'd do differently:** Start with global rules earlier. We spent the first week putting preferences into project memory, then had to extract them later. If you identify something as a preference (not a fact), put it in rules from day one.

---

## Tips

- **Start with your most active project** — that's where you have the most feedback data and the most to gain from structured memory.

- **Expect 2-3 iterations** before the MEMORY.md structure feels right. The first pass is always too granular or too coarse. That's normal.

- **Global rules > duplicated memories.** Every time you catch yourself writing the same thing in two projects, stop and move it to rules.

- **Small projects benefit from shared rules more than their own memories.** Don't force every project to have 50 memories — some genuinely only need 3.

- **Review monthly:** Which project memories should become global rules? Which are stale? Which projects have grown enough to need better organization?

- **Name files consistently across projects.** If Project A has `feedback-no-mocking.md`, Project B's equivalent should follow the same `feedback-*.md` pattern. Consistency helps you spot duplicates.

- **Don't over-index on memory count.** Quality matters more than quantity. 5 precise, well-described memories outperform 30 vague ones.
