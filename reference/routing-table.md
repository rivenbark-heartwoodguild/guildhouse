# The Production Routing Table

A 5-rule query router derived from 24 benchmark races across 5 seasons. These rules are generalized — they describe query shapes and retrieval strategies, not specific tools. Adapt them to whatever structured store, semantic search, and file-based memory you run.

---

## How to Read This

Each rule maps a **query shape** to the optimal retrieval route. "Query shape" means the TYPE of question, not the specific words. The router classifies the question first, then picks the route.

The token counts are approximate and reflect what we measured in benchmarks. Yours will differ based on your data density and system verbosity.

---

## The 5 Rules

| # | Query Shape | Example | Route | ~Tokens | Logic |
|---|-------------|---------|-------|---------|-------|
| 1 | **Exact fact** (date, status, price, count) | "When was the API key rotated?" | Structured store with field filter | 15-67 | Single typed fact. Structured wins by an order of magnitude. Semantic search returns paragraphs to extract one value. |
| 2 | **Broad entity** ("What is X?") | "What is the billing service?" | Structured + Semantic in parallel, synthesize | ~1,875 | Structure provides the skeleton (type, status, relationships). Semantics add narrative depth (why it exists, how it evolved). Neither alone gives the full picture. |
| 3 | **Narrative** (history, evolution, context) | "How did we end up with this architecture?" | Semantic search only | ~750 | Structured store has nothing to contribute. Narrative lives in documents, not typed triples. Running both wastes tokens. |
| 4 | **Relationship** (who connects to what) | "Which services depend on the auth module?" | Semantic first, extract entities, structured backfill, refined semantic | ~1,200 | Semantic finds the names and context. Structured fills in the precise details (versions, ownership, status). The second semantic pass uses the discovered entities to find deeper connections. |
| 5 | **Recent activity** (what happened lately?) | "What changed in the payment flow this week?" | File memory first, then semantic search | ~1,100 | Files provide the temporal index — dates, names, session outcomes. Semantic search then dives deep on the specific topics surfaced. Without files first, semantic search doesn't know WHEN to look. |

---

## Fast-Path Rules

Short-circuit the full pipeline when confidence is high enough:

**Structured fast-path:** If the structured store returns an exact answer with relevance >= 0.95, stop. Do not run semantic search. This saves ~600 tokens per query in benchmarks.

**Semantic fast-path:** If semantic search scores >= 0.90 and the structured store returns nothing, use the semantic result directly. Do not attempt structured backfill. This saves ~800 tokens per query.

The threshold values (0.95, 0.90) came from our benchmarks. Tune them for your system — too low and you'll get false positives, too high and the fast-path never fires.

---

## Token Savings from Fast-Path

In 24 benchmark races:
- Fast-path fired on ~40% of queries
- Average savings: ~700 tokens per fast-path activation
- Zero quality degradation on fast-path queries (the confidence thresholds prevent it)

At scale, these savings compound. A system handling 100 queries/session saves ~28,000 tokens just from knowing when to stop early.

---

## How to Customize

This routing table was derived from ONE system's benchmarks over 5 seasons. Your optimal routes may differ based on:

- **Data distribution** — If your structured store is sparse, Rule 2 might collapse to semantic-only. If your documents are thin, Rule 3 might need structured support.
- **System latency** — If your structured store is slow, parallel execution (Rule 2) matters more. If it's fast, sequential with early-exit might win.
- **Query mix** — If 80% of your queries are exact facts, invest in structured store quality. If 80% are narrative, invest in semantic search.

Run your own benchmarks (see [Drag Race Methodology](drag-race-methodology.md)) and adjust. The PATTERN (classify, route, fast-path) generalizes even if the specific thresholds don't.

---

## The Meta-Insight

The constraint is never any individual system's quality. It's routing intelligence.

Every optimization that moved the needle in our benchmarks was a routing decision — "only ask the structured store for pricing" or "always run file memory before semantic search for recency queries." Individual system improvements (better embeddings, more triples) produced marginal gains. Routing produced step-function improvements.

This mirrors a general principle: improve the system by managing the bottleneck, not by making every part faster.
