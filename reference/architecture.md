# Architecture — The Three Houses

A three-tier memory architecture for AI systems, with a routing layer that classifies queries and directs them to the right tier (or combination of tiers).

---

## The Three Houses

### MemPalace (Structured / Knowledge Graph Tier)

Cold, precise. A filing cabinet with typed facts.

Every entry is a triple: entity, relationship, value — optionally with temporal validity (valid_from, valid_to). This is the only tier that can answer "what was true on March 3rd?" without scanning documents.

**Strengths:**
- Minimal token cost (15-67 tokens for a single fact)
- Temporal validity — knows what changed and when
- Typed relationships — can traverse "depends_on" or "owned_by" edges
- Exact answers, no interpretation needed

**Weaknesses:**
- Cannot handle narrative or context
- Requires explicit data entry (someone must create the triples)
- Sparse by nature — only knows what's been explicitly recorded
- Fails completely on "why" questions

**Best for:** Status checks, date lookups, relationship traversal, pricing, versioning — anything with a definitive typed answer.

---

### GuildHouse (File-Based / Working Memory Tier)

Warm, lived-in. The guildhall where working knowledge accumulates.

This is markdown files on disk: feedback notes, project status, session outcomes, decision logs, checklists. Fast because it's just files (grep, cat, read). Always available because there's no service to go down. Organized by human convention (directories, naming patterns, index files) rather than by algorithm.

**Strengths:**
- Zero latency — no API calls, no embeddings, no service dependencies
- Human-readable and human-writable
- Natural temporal organization (files have dates, sessions have timestamps)
- Easiest tier to start with — just start writing markdown

**Weaknesses:**
- Cannot synthesize across sources (it returns files, not answers)
- Shallow — you get the document, but not connections between documents
- Scales poorly past ~500 files without good organization conventions
- Search is keyword-based (grep), not meaning-based

**Best for:** Recent project context, feedback and corrections, session outcomes, quick reference lookups — anything where you know roughly what you're looking for and need it fast.

---

### LakeHouse (Semantic / Vector Search Tier)

Deep. Finds meaning, not just keywords.

The intellectual heritage comes from the Databricks Lakehouse pattern: unified structured + unstructured storage in a single system. In practice, this is your vector document store — documents chunked, embedded, and searchable by meaning. Ask "how did the architecture evolve?" and it finds relevant passages even if they never use the word "architecture."

**Strengths:**
- Meaning-based retrieval — handles paraphrasing, synonyms, conceptual similarity
- Broad coverage — indexes everything, not just explicitly tagged facts
- Handles narrative and context well
- Scales to thousands of documents without degradation

**Weaknesses:**
- Higher token cost per query (~750-1,875 tokens depending on result count)
- Requires embedding infrastructure (indexing pipeline, vector store)
- Can return plausible-sounding but wrong results (semantic similarity != correctness)
- No temporal awareness — treats a 2-year-old document the same as yesterday's

**Best for:** Conceptual questions, narrative history, "what do we know about X" exploration, finding connections you didn't know existed.

---

## Why Three Tiers?

Each tier has a fundamental capability the others lack:

| Capability | Which Tier | Why Others Can't |
|------------|-----------|-----------------|
| Temporal validity ("what was true when?") | MemPalace | Files don't track validity ranges. Vectors don't know time. |
| Zero-latency project context ("what's happening now?") | GuildHouse | KG requires explicit entry. Vector search requires indexing. Files are instant. |
| Meaning-based retrieval ("what's related to this?") | LakeHouse | KG only finds explicit relationships. Files only find keyword matches. |

A system with only one tier has two blind spots. A system with two tiers has one. The three-tier architecture eliminates the structural blind spots — the remaining gaps are data quality, not architecture.

---

## The Router Layer

GuildHouse (the project, not the tier) sits above all three tiers. Its job:

1. **Classify the query** — What shape is this question? (See [Routing Table](routing-table.md))
2. **Pick the route** — Which tier(s) should handle it?
3. **Execute** — Run the retrieval, potentially chaining tiers (semantic finds names, structured fills details)
4. **Fast-path** — Short-circuit when confidence is high enough

The router is where the intelligence lives. The tiers are storage and retrieval engines. The router is the decision-maker.

```
        ┌──────────────┐
        │  User Query   │
        └──────┬───────┘
               │
        ┌──────▼───────┐
        │    Router     │  ← classifies, routes, fast-paths
        └──┬───┬───┬───┘
           │   │   │
     ┌─────▼┐ ┌▼────┐ ┌▼──────┐
     │ Mem  │ │Guild│ │ Lake  │
     │Palace│ │House│ │ House │
     └──────┘ └─────┘ └───────┘
     structured  files  semantic
```

---

## What You Actually Need

Not every system needs three tiers on day one. Build incrementally:

### Minimum Viable: File-Based Memory

Start with a `MEMORY.md` index and a directory of markdown files. Organize by type: feedback, projects, sessions, reference. This is GuildHouse's core tier and it's remarkably effective on its own.

**You get:** Fast project context, feedback tracking, session continuity.
**You miss:** Meaning-based search, temporal precision, cross-document synthesis.

### Better: Files + Semantic Search

Add a vector store over your markdown documents. Tools like QMD, Obsidian with Smart Connections, or any embedding pipeline will work.

**You get:** Everything above, plus conceptual search across your documents.
**You miss:** Temporal validity, typed relationships, exact-fact retrieval at minimal token cost.

### Full Architecture: All Three Tiers + Router

Add a knowledge graph (or structured store — it doesn't have to be a full graph database). Build the routing table through benchmarks.

**You get:** The complete system with no structural blind spots.
**You pay:** Setup complexity, maintenance of three systems, routing logic development.

### Decision Framework

| Signal | What to Add |
|--------|------------|
| You keep asking "when did X change?" and can't find it | Add structured store (temporal validity) |
| You know the answer is in your docs but keyword search misses it | Add semantic search |
| You need instant project context without waiting for search | Add file-based memory |
| You're running the wrong retrieval system for certain queries | Add routing logic |

---

## The Constraint Insight

The constraint in a retrieval system is never any individual tier's quality. It's routing intelligence.

In 24 benchmark races across 5 seasons, every optimization that produced a step-function improvement was a routing decision: "only ask the structured store for exact facts" or "always check file memory before semantic search for recency queries."

Individual tier improvements (better embeddings, more graph coverage, cleaner documents) produced marginal, linear gains. Routing improvements produced step-function gains.

This reflects a general systems principle: improve the system by managing the bottleneck, not by making every part faster. If the bottleneck is "asking the wrong system," then making any individual system better doesn't help. Fix the routing, and every system's contribution improves.
