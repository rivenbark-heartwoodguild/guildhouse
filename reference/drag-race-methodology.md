# Drag Race Methodology — Benchmark Your Own System

How to run controlled benchmarks against your retrieval systems to derive YOUR routing table. The name comes from the format: same track, same conditions, best time wins.

---

## The Concept

"Drag racing" means running the same query against multiple retrieval systems simultaneously and scoring the results. Each system gets the identical input. You measure who gives the best answer, and who gives the best answer per token.

The goal is not to crown a winner — it's to learn which system wins which TYPES of queries. That knowledge becomes your routing table.

---

## Step 1: Design Your Benchmark Queries

Write 6 queries covering different question types. Use YOUR actual questions — the ones you ask your system regularly. Generic benchmark queries produce generic routing tables.

| Query Type | What It Tests | Example |
|------------|--------------|---------|
| Entity status | Exact fact retrieval | "What is the current version of the auth service?" |
| Temporal | Date/time precision | "When did we migrate to PostgreSQL 16?" |
| Narrative | Context and history | "How did the caching strategy evolve?" |
| Relationship | Connection discovery | "Which teams depend on the notification service?" |
| Recent activity | Recency awareness | "What changed in the API gateway this sprint?" |
| Specific lookup | Single-field precision | "What is the rate limit for the public endpoint?" |

**Important:** All 6 queries must have known correct answers. You need ground truth to score against. If you don't know the answer, you can't judge the systems.

---

## Step 2: Choose Your Competitors

Run each query against every retrieval system you have. Common setups:

| System | What It Represents |
|--------|-------------------|
| Knowledge graph / structured store | Typed facts, relationships, temporal validity |
| Vector search / semantic store | Meaning-based document retrieval |
| File-based memory | Direct file reads (grep, cat, recent files) |
| Hybrid router | Your combined system that routes across tiers |

You can race 2 systems or 5. More systems = more insight, but also more work per race.

---

## Step 3: Run the Race

For each query against each system, record:

| Metric | How to Measure |
|--------|---------------|
| **Token count** | Total tokens in the response (input + output if relevant) |
| **Relevance score** | 0.0 to 1.0 — how well does the response answer the question? |
| **Precision** | Categorize: `exact` (single correct fact), `structured` (organized correct answer), `prose` (correct but buried in text), `partial` (some correct, some missing), `none` (wrong or empty) |
| **Latency** | Wall-clock time (optional but useful) |

**Scoring relevance:** Use a judge — another LLM, a teammate, or yourself. The judge sees the query, the ground truth answer, and the system's response, then scores 0.0-1.0. Consistency matters more than absolute accuracy, so use the same judge for all races in a season.

---

## Step 4: Score the Season

Two championships per season:

### Quality Championship
Who gives the best answer regardless of cost?
- Rank systems by average relevance score across all 6 queries
- Note which system wins which query types

### Efficiency Championship
Who gives the best answer per token?
- Score: `(relevance / tokens) * 1000`
- A system scoring 0.9 relevance in 50 tokens beats one scoring 0.95 in 500 tokens

### Per-Query-Type Analysis
This is where the routing table comes from:
- For each query type, which system won quality? Efficiency?
- Are there query types where one system ALWAYS wins?
- Are there query types where the hybrid beats every individual system?

---

## Step 5: Derive Your Routing Table

After 6+ races, patterns emerge. Map them:

```
Query Type → Best System → Routing Rule
─────────────────────────────────────────
Exact fact → Structured store → Route 1: structured only
Broad entity → Hybrid (structured + semantic) → Route 2: parallel, synthesize
Narrative → Semantic only → Route 3: semantic only
...
```

Also identify fast-path opportunities:
- When does the first system checked give a definitive answer?
- What confidence threshold lets you skip the remaining systems?

---

## Step 6: Run Multiple Seasons

As your data grows and systems evolve, re-run the same 6 queries. Compare season-over-season:

- Did the routing table change? (It should, as your data distribution shifts.)
- Did individual systems improve? (Track this separately from routing.)
- Did new query types emerge? (Add them to the benchmark set.)

We ran 5 seasons over several months. The routing table stabilized after season 3, with minor adjustments in seasons 4-5. The biggest changes came from data growth (more documents made semantic search stronger) and structured store maturation (more typed facts made exact queries faster).

---

## What We Found

Over 5 seasons (24 races), the key findings were:

1. **The hybrid router won 19/24 quality awards.** No single system beats intelligent routing.

2. **Fast-path optimization was the biggest win.** Knowing when NOT to run the full pipeline saved more tokens than any individual system improvement. This was counterintuitive — we expected embedding quality or graph coverage to be the lever.

3. **Query classification accuracy matters more than retrieval quality.** A mediocre retrieval system with perfect routing beats a great retrieval system with random routing. Invest in the classifier.

4. **Diminishing returns on system improvement.** After season 2, improving individual systems (better embeddings, more triples, cleaner documents) produced marginal gains. Improving routing produced step-function gains.

5. **Your mileage will vary.** These findings reflect our data, our systems, our query patterns. The methodology generalizes. The specific results don't. Run your own races.

---

## Race Record Template

Use this structure for recording individual races:

```markdown
## Race: [Query Text]
**Season:** [N] | **Date:** [YYYY-MM-DD]
**Ground Truth:** [The known correct answer]

| System | Tokens | Relevance | Precision | Efficiency |
|--------|--------|-----------|-----------|------------|
| Structured | | | | |
| Semantic | | | | |
| File Memory | | | | |
| Hybrid | | | | |

**Quality Winner:** [system]
**Efficiency Winner:** [system]
**Notes:** [observations]
```

Store race results in a structured format (JSON works well) so you can analyze trends across seasons programmatically.
