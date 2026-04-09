# Advanced Patterns — Growing Beyond the Basics

These patterns emerged from running a three-tier memory system in production over several months. They're not required — the basic routing table and memory files work fine on their own. But each pattern addresses a specific failure mode we hit as the system scaled.

---

## Knowledge Gardener

**Problem:** With 70+ memory files and thousands of KG triples, knowledge contradicts itself silently. The KG says a project is "active" but no one's mentioned it in 45 days. Two memory files describe the same decision differently.

**Pattern:** Multi-system triangulation. For each significant entity, query all three tiers and compare:
- Does the KG agree with the memory files?
- Is the KG more recent than the document store, or vice versa?
- Are there entities in one system with zero coverage in the others?

**Classification:**

| Finding | Label | Action |
|---------|-------|--------|
| KG fact contradicts document evidence | `[FIX]` | Draft KG update for human approval |
| Entity's newest mention is >30 days old across all systems | `[DECAY]` | Flag for review — is this still active? |
| Entity found in documents but has no KG triples | `[GAP]` | Draft seed triples from document context |
| Entity found in KG and documents but has no memory file | `[COVERAGE]` | Consider adding to the Workbench |

**Output:** A health report surfaced at session start:
```
Knowledge Health: 71 entities scanned | 0 contradictions | 5 decaying | 45 coverage gaps
```

**How we run it:** Python script (~500 lines) triggered nightly after search index update. Queries the KG for entities with 5+ triples, runs semantic searches for each, greps memory files. Writes report to `memory/knowledge-health.md`.

**The constraint insight:** The comparison engine is the valuable part, not any individual check. Once you can query the same entity across all systems, every subsequent pattern (decision replay, gap detection, freshness checking) becomes trivial.

---

## Checkpoints

**Problem:** Long sessions (50+ turns) accumulate context that will be lost when the conversation compresses or ends. The session transcript captures everything, but it's not structured for recovery.

**Pattern:** At natural milestones, write a structured checkpoint to a known file path:

```markdown
## Checkpoint — 2026-04-09 14:30

**Current task:** Migrating payment service from Stripe v2 to v3
**Key decisions:**
- Using the migration helper, not a full rewrite
- Keeping v2 webhooks active during transition (dual-write)
**Important state:**
- 6 of 12 endpoints migrated
- Tests passing on migrated endpoints
- Found a bug in refund handling — filed as issue #234
**Completed:** endpoints 1-6 (customer, subscription, invoice, charge, refund, payment_intent)
**Remaining:** endpoints 7-12 (transfer, payout, balance, dispute, coupon, promotion_code)
**Context that would be lost:** The refund bug only manifests with partial refunds >$500. Regular refunds work fine.
```

**Key rules:**
- Overwrite the same file each time — you only need the latest state
- Write at: significant task completion, topic switches, ~30 turns, key decisions
- Don't announce it — just write silently and continue working

**Recovery:** Start a new session, read the checkpoint, resume from where you left off. The structured format means the AI can immediately orient itself without re-reading the full transcript.

---

## Speculative Prefetch

**Problem:** SessionStart always loads the same generic context. But most sessions have a specific focus — you're working on the payment service, or reviewing security, or doing a weekly check-in. Generic context wastes tokens.

**Pattern:** Classify the likely session focus from signals available before the first message, then pre-load only the relevant knowledge.

**Signals we use:**

| Signal | Source | What It Predicts |
|--------|--------|-----------------|
| Git status (modified files) | `git status --short` | Which area of the codebase you're working in |
| Recent commits | `git log -5 --oneline` | Momentum direction |
| Day of week | `date +%A` | Recurring meetings (standup Monday, retro Friday) |
| Time of day | `date +%H` | Morning = deep work, afternoon = meetings |
| Open transactions | Memory directory scan | Active multi-session work |
| Health report flags | `knowledge-health.md` | Entities needing attention |

**Budget:** <3 seconds total. This runs inside the 5-second SessionStart timeout, leaving room for other checks.

**Example classifier (bash pseudocode):**
```bash
# If git status shows files in src/payments/
if git status --short | grep -q "src/payments/"; then
  echo "## Pre-loaded: Payment Service Context"
  # Query KG for payment-related entities
  # Show recent payment-related decisions from memory
fi

# If it's Monday
if [ "$(date +%A)" = "Monday" ]; then
  echo "## Pre-loaded: Weekly Standup Context"
  # Show active project statuses
  # Surface any blocked items
fi
```

**The insight:** A prefetch that fires on 2 of 10 sessions but is highly relevant when it does is more valuable than one that fires on every session with mediocre relevance. Optimize for precision, not recall.

---

## Decision Replay

**Problem:** Decisions made 3 months ago were correct at the time. But the landscape shifts — a library gets deprecated, a team member leaves, a customer changes requirements. Old decisions silently become wrong.

**Pattern:** Take a past decision, reconstruct the original context, then re-ask the question with current knowledge.

**Three verdicts:**

| Verdict | Meaning | Action |
|---------|---------|--------|
| **Confirmed** | Same answer with today's knowledge | No action — decision is solid |
| **Drifted** | Decision still defensible, but key assumptions shifted | Review — might want to adjust |
| **Invalidated** | New evidence contradicts the original reasoning | Must revisit — the decision is wrong |

**Process:**
1. **Reconstruct original context** — Find when the decision was made, what was known at the time, what the alternatives were
2. **Gather current evidence** — What's changed? New information, deprecated tools, shifted requirements
3. **Re-ask the question** — Given what we know NOW, would we make the same choice?

**Triggers:**
- Entities appearing on the Gardener's decay report (knowledge going cold means decisions going stale)
- Before doubling down (Phase 2 of a project, scaling a prototype, extending a library choice)
- Monthly review of technology decisions

**Scope:** Focus on decisions that carry forward — architecture choices, library selections, process decisions. Skip one-time operational calls, naming decisions, and aesthetic choices.

**Example output:**
```
## Decision Replay — 2026-04-09

### "Use SQLite for local knowledge graph"
- **Original context (Feb 2026):** Needed lightweight embedded DB, no server setup
- **Current evidence:** KG has grown to 2,700 entities. Query patterns are still simple lookups. No concurrent write pressure.
- **Verdict: Confirmed** — SQLite handles this load easily. Would make the same choice today.

### "Skip unit tests for hook scripts"
- **Original context (Mar 2026):** Hooks were simple 10-line scripts
- **Current evidence:** Scripts now 50-200 lines, with date parsing, API calls, and conditional logic. Two bugs in the last month.
- **Verdict: Drifted** — The scripts have outgrown the "too simple to test" justification. Should add tests.
```

---

## Staleness Detection

**Problem:** Memory files don't expire themselves. A project status from 30 days ago silently pollutes decisions today.

**Pattern:** Add TTL metadata to frontmatter, then check at session start.

```yaml
---
name: Sprint status
type: project
last_verified: 2026-04-01
ttl_days: 14
---
```

**The check (Python — cross-platform):**
```python
#!/usr/bin/env python3
import os, re, sys
from datetime import datetime, date

memory_dir = os.environ.get("MEMORY_DIR", "memory")
today = date.today()
stale = 0

for fname in sorted(os.listdir(memory_dir)):
    if not fname.endswith(".md"):
        continue
    path = os.path.join(memory_dir, fname)
    with open(path) as f:
        head = f.read(1024)
    verified = re.search(r'last_verified:\s*(\d{4}-\d{2}-\d{2})', head)
    ttl = re.search(r'ttl_days:\s*(\d+)', head)
    if not verified or not ttl:
        continue
    days_old = (today - datetime.strptime(verified.group(1), "%Y-%m-%d").date()).days
    if days_old > int(ttl.group(1)):
        stale += 1
        print(f"  Stale: {fname} ({days_old}d old, TTL {ttl.group(1)}d)")

if stale:
    print(f"Warning: {stale} memory file(s) past TTL")
else:
    print("All memory files within TTL")
```

**Recommended TTLs:**

| Type | TTL | Why |
|------|-----|-----|
| feedback | 180 days | Corrections are durable |
| project | 14 days | Status changes fast |
| session | 90 days | Decisions are semi-durable |
| reference | -- (evergreen) | Resources rarely move |

---

## Memory Maintenance

**Problem:** Over time, memory accumulates redundancy: two files describing the same decision in different words, files that drifted from their MEMORY.md entry, files that grew too large to be useful context.

**Pattern:** Three automated checks, run on-demand (not nightly):

**Overlap detection:** Compare sentences across memory files. If two files share >50% of their sentences, they're dedup candidates. Sentence-level comparison (not word-level) catches paraphrasing.

**Orphan detection:** Files in `memory/` not referenced by MEMORY.md are invisible to the session index. Either add them or delete them.

**Size check:** Memory files over 3,000 words are too large for effective context loading. Split them or compress to the essential points.

**A simple maintenance script outputs:**
```
Memory Maintenance Report
=========================
Stale: 3 files past TTL
Overlapping: 2 pairs (>50% sentence match)
  - feedback-testing.md <> feedback-integration-tests.md (67% overlap)
  - project-api.md <> session-api-decisions.md (52% overlap)
Orphaned: 1 file not in MEMORY.md
  - research-caching.md
Oversized: 0 files
```

---

## Transactional Knowledge (Emerging Pattern)

**Problem:** Multi-session work creates speculative knowledge. You research a potential client, draft a proposal, explore an architecture. If the deal falls through or the design gets rejected, that speculative knowledge stays in the system forever, polluting future queries.

**Pattern:** Scope work into named engagements with explicit lifecycle:

```
/engage open project-x
  -> creates a transaction log
  -> all facts tagged as PROVISIONAL
  -> session context scoped to this engagement

  [work happens across multiple sessions]
  [facts, documents, decisions accumulate]

/engage commit project-x
  -> promotes PROVISIONAL facts to permanent
  -> archives the transaction log

/engage rollback project-x
  -> expires PROVISIONAL facts
  -> archives without promoting
  -> speculative knowledge cleaned up
```

**Why "emerging":** This pattern is designed but not fully battle-tested. The concept is sound — databases have had transactions for decades — but applying it to knowledge management adds complexity. Start with the simpler patterns above and add transactions when you notice speculative knowledge causing problems.

**The core question this pattern answers:** "If this project doesn't work out, can I cleanly remove everything we learned about it without manually hunting through 3 different knowledge stores?"

---

## Pattern Dependencies

Not every pattern requires every other pattern. Here's what depends on what:

```
Basics (memory files + MEMORY.md)
  |-- Staleness Detection (adds TTL frontmatter)
  |-- Checkpoints (adds checkpoint writing)
  |-- Memory Maintenance (adds dedup/orphan checking)
  |-- Automation (adds hooks)
        |-- Speculative Prefetch (runs inside SessionStart)
        |-- Session Export Pipeline (runs at SessionEnd)
              |-- Knowledge Gardener (runs on indexed data)
                    |-- Decision Replay (uses Gardener's decay report)
                    |-- Transactional Knowledge (uses Gardener's growth detection)
```

Start at the top. Work down as you feel the pain each pattern solves.
