# Post-Setup Checklist

You've built the memory system. Now make sure it actually works. This checklist covers validation, testing, and ongoing maintenance.

---

## Immediate Cleanup (Day 1)

- [ ] **Deduplicate** — Search for memories that say the same thing in different words. Keep the better one, delete the other. Common duplicates: preferences stated as both feedback and user memories, project status captured in both a project memory and a session memory.

- [ ] **Remove CLAUDE.md duplicates** — If a memory just restates what's in CLAUDE.md, delete it. CLAUDE.md is the source of truth for project instructions. Memory is for what emerges during work, not what's known upfront.

- [ ] **Verify frontmatter** — Every memory file needs valid YAML frontmatter with at least: `name`, `description`, `type`. Check for common YAML errors:
  - Missing `---` delimiters (need one above and one below)
  - Tabs instead of spaces
  - Unquoted strings with colons (wrap in quotes: `description: "PostgreSQL: connection pooling"`)
  - Missing required fields

- [ ] **Check MEMORY.md format** — Each index entry should follow `- [Title](file.md) — hook` format and stay under 150 characters. Verify every linked filename matches an actual file. Broken links mean invisible memories.

- [ ] **Index length** — MEMORY.md should be under 200 lines. If it's longer, consolidate related memories or move low-priority items to an "On-Demand Reference" section at the bottom. Claude loads the full index, so every line costs context.

---

## Testing (Day 1-2)

- [ ] **Fresh session test** — Start a new Claude Code session in the project. Does MEMORY.md load? You should see evidence of memory awareness in how Claude approaches tasks — referencing known preferences, acknowledging project state, or avoiding previously corrected mistakes.

- [ ] **Memory recall test** — Ask about something stored in memory. For example, if you have a feedback memory about test philosophy, ask Claude to write a test. Does it follow the stored preference naturally, without being told "check your memory"?

- [ ] **Correction test** — Correct Claude on something during the session. Does it create or update a feedback memory? If auto-memory is enabled, a new memory file should appear. If it doesn't, check that auto-memory is configured correctly.

- [ ] **Relevance test** — Ask about something NOT in memory. Claude should respond based on available context (code, docs, conversation) — not hallucinate that it "remembers" something from a previous session that was never stored.

---

## Week 1 Review

- [ ] **Are memories being used?** — Over the first week, track whether Claude references stored memories during normal work. If memories are consistently ignored:
  - Descriptions may be too vague — rewrite them to be more specific about when they're relevant
  - The memory might not match the work you're actually doing — realign content to current tasks
  - The index might be too long — Claude may be skimming past relevant entries

- [ ] **Are new memories being created?** — After corrections and key decisions, new feedback and session memories should appear. Healthy growth rate: 1-3 new memories per week. If it's 0, the system isn't learning from sessions. If it's 10+, quality is likely suffering — most sessions don't produce 10 novel insights.

- [ ] **Is the structure holding?** — Are new memories being placed in the right categories? Is MEMORY.md staying organized, or has it become a chronological dump? Course-correct early — structure debt compounds fast.

---

## Ongoing Maintenance (Monthly)

- [ ] **Review expired memories** — Check dates on project and session memories:
  - Project memories older than 14 days: Is the status still accurate? Refresh or remove.
  - Session memories older than 90 days: Is the decision still relevant? If it's become standard practice, it's in the code now — the memory can go.
  - Feedback older than 180 days: Still applicable? Preferences evolve.

- [ ] **Promote patterns** — If 3+ projects have the same feedback memory (or you keep giving the same correction across projects), move it to `~/.claude/rules/` as a global rule. The rule applies everywhere; the duplicated memories can be deleted.

- [ ] **Audit for staleness** — Read through MEMORY.md start to finish. For every entry, ask: "Is this still true? Would Claude make a worse decision without it?" Delete what doesn't pass both tests.

- [ ] **Check for orphans** — Files in `memory/` that aren't linked from MEMORY.md. These are invisible to the session-start index. Either link them (if still valuable) or delete them (if not).

- [ ] **Measure index size** — Run `wc -l` on MEMORY.md. If it's crept past 200 lines, consolidate. Combine related memories into single files. Move reference material to an on-demand section.

- [ ] **Run staleness check** — If using TTL frontmatter (`last_verified` + `ttl_days`), scan for overdue files. A simple script that compares dates can flag these automatically at session start. See [Advanced Patterns](../reference/advanced-patterns.md#staleness-detection) for the 42-line implementation.

- [ ] **Check for overlapping memories** — Two files that say the same thing in different words waste context. Look for memories with >50% sentence overlap and merge them. See [Memory Maintenance](../reference/advanced-patterns.md#memory-maintenance).

- [ ] **Review orphaned files** — Files in `memory/` not linked from MEMORY.md are invisible to the session index. Either add them to the index or delete them.

---

## Signs It's Working

**The AI references past decisions without being asked.** You discuss a feature, and Claude mentions the architectural constraint you documented three weeks ago. You didn't prompt it — the memory surfaced naturally because the description matched the context.

**Corrections stick.** You said "no semicolons" once. It never happened again. The feedback memory persisted and influenced every subsequent session.

**New sessions start with useful context.** Instead of a blank-slate greeting, Claude's first actions show awareness of project state, your preferences, and recent decisions. The ramp-up time from "new session" to "productive work" shrinks from minutes to seconds.

**The AI's behavior matches your preferences across sessions.** Tone, verbosity, tool choices, testing philosophy — all consistent, even in sessions weeks apart.

---

## Signs Something's Wrong

**The AI ignores memories entirely.**
- Check MEMORY.md formatting — broken markdown means broken loading
- Verify the memory directory is at the correct path under `~/.claude/projects/`
- Ensure file links in the index match actual filenames (case-sensitive)

**Memories are all stale (>30 days without updates).**
- Schedule a 30-minute refresh session
- Focus on project memories first — they age fastest
- Delete anything that's no longer true rather than trying to update everything

**MEMORY.md is 300+ lines.**
- Consolidate aggressively — combine related memories into single files
- Move low-priority items to an on-demand section
- Ask: "Which 20 of these 50 memories actually influence daily work?" Keep those, archive the rest.

**The AI quotes memories verbatim instead of using them naturally.**
- Descriptions may be too prescriptive — they should describe relevance, not dictate behavior
- Example of too prescriptive: "When writing tests, always say: 'I'm using integration tests because mocks cause drift'"
- Better: "Integration tests preferred over mocks — mock drift caused production bug in March"

**Memory count is growing but quality isn't improving.**
- You're saving too much. Apply the golden rule: save what would be lost between sessions and can't be derived from the codebase.
- Review the last 10 memories created. How many actually influenced a decision? Delete the ones that didn't.

---

## The Maintenance Mindset

Memory systems decay by default. Without active maintenance, they fill with stale project states, outdated decisions, and corrections that no longer apply. The monthly review isn't optional — it's what keeps the system valuable.

Think of it like tending a garden: regular pruning produces better growth than letting everything grow wild and doing a massive cleanup once a year.
