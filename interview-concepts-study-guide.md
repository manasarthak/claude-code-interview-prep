# Data Engineering & Applied AI — Interview Concepts Study Guide

**Last updated:** July 3, 2026
**Purpose:** deep, correct understanding of the core concepts across data engineering, applied AI, and system design that recur in senior-level interviews.

---

## Meta — What Interview Screens Actually Test

Multiple-choice and short-form conceptual questions in DE / Applied AI / system design screens are **not** random algorithm trivia. They test whether you understand the *reason* behind an engineering decision — and the wrong answers are seeded with confident-sounding-but-subtly-wrong alternatives. Recognition alone won't save you; you need the underlying model.

The recurring topic groups worth mastering:

1. Data privacy & governance (PII, anonymisation, access control)
2. Dimensional modeling & warehousing (star/snowflake, grain, OLTP/OLAP)
3. Streaming architecture (Lambda precedence, watermarks, event-time, async)
4. Lakehouse / Delta specifics (time travel, VACUUM, shallow clone)
5. Snowflake operational model (warehouse size vs concurrency)
6. NoSQL selection by access pattern
7. Vector search & RAG (ANN, chunking, NLI limits)
8. Agentic AI patterns (ReAct vs plan, automation bias, graceful degradation)
9. Monolith vs microservices (trade-off, not doctrine)

Each section below covers the concept, the counter-intuitive right answer, the traps that look right, and where to go deeper in this repo.

---

## 1. Data Privacy & Governance

### 1.1 Anonymisation vs Pseudonymisation — The Reversibility Axis

The distinction is a single word: **reversibility**.

| | Anonymisation | Pseudonymisation |
|---|---|---|
| Re-identification | **Irreversible** — no key, no mapping, no path back to the individual | **Reversible** — the mapping exists somewhere (held by the data controller or a trusted third party) |
| Techniques | k-anonymity, l-diversity, t-closeness, differential privacy, aggregation-only outputs, generalization to buckets | Tokenization with lookup, deterministic encryption, hashing with kept salt, format-preserving encryption |
| GDPR status | **Falls outside GDPR scope** (no longer personal data) | **Still personal data** — GDPR protections continue to apply |
| Purpose | Publish/share data with no linkage risk | Protect operational data while keeping the ability to link records for legitimate processing |
| Example | Published census tract with cell suppression | User ID replaced by `pseudo_12345` in analytics tables; mapping kept in a locked vault |

**Common traps:**
- "Encryption is a form of anonymisation." **False.** Encryption is reversible with the key → pseudonymisation at best.
- "Hashing = anonymisation." **False**, if the input space is small and enumerable. `hash(email)` is trivially re-identifiable by brute-forcing every known email against the hash. It's pseudonymisation with a weak key.
- "Column removal = anonymisation." Often insufficient — quasi-identifiers (ZIP + DOB + sex) can re-identify most Americans without a name (Sweeney's 87% result).
- "Pseudonymised data doesn't need GDPR compliance." **False** — it's still personal data, still subject to the full regulation.

### 1.2 Enforcing PII Access Across Many Consumers — Enforcement at the Query Layer

The principle: **PII protection must live at a single system-controlled choke point that every consumer must pass through**, not by convention or per-team code.

The four approaches ranked:

| Approach | Robustness | Why |
|---|---|---|
| ✅ **Query-layer column/row-level access control** (Unity Catalog, Lake Formation, Snowflake dynamic data masking, Ranger) | **Best** | The masking policy is evaluated on every query based on the querying user's role. Impossible to bypass by writing new code. |
| Distribute redacted copies to each team | Weak | Copies proliferate; who audits which team has which copy? |
| App-level filtering + code review | Weak | One missed query leaks PII. Convention-based enforcement always breaks. |
| Nightly anonymised copy in a separate schema | Coarse | Stale by definition, and the raw lake still exists — a curious analyst can query it. |

**The general pattern:** the "right" answer is the one where a *system* enforces the rule, not a *process*. Any answer that requires humans to remember something is wrong.

**Related tools/terms to recognize:**
- **Column-level security (CLS)** — mask or hide specific columns per role.
- **Row-level security (RLS)** — filter rows per role (e.g., analysts see only their region).
- **Dynamic data masking** — Snowflake's mechanism: `CREATE MASKING POLICY email_mask AS (val string) RETURNS string -> CASE WHEN current_role() IN ('ANALYST_PII') THEN val ELSE '****@***.***' END`.
- **Tag-based policies** — tag columns with `PII: true`, then a single policy applies everywhere the tag appears. This is what Unity Catalog / Lake Formation are converging on.

Deeper: [Governance, Lineage, Observability](./data-engineer/14-governance-lineage-observability.md).

---

## 2. Dimensional Modeling & Data Warehousing

### 2.1 Star vs Snowflake — The Normalization Axis (nothing else)

- **Star schema:** dimensions are **denormalized** flat tables joined directly to the fact table. One JOIN from fact → dim. Fewer joins, faster queries, more storage redundancy.
- **Snowflake schema:** dimensions are **normalized** into hierarchical sub-tables (`product` → `category` → `department`). Less redundancy, more joins, slower analytical queries but easier to maintain if hierarchies change.

**Common traps** — these are *not* the axis:

- Storage format (row vs columnar) — orthogonal.
- Key type (natural vs surrogate) — orthogonal; both use surrogate keys.
- Row-count thresholds — arbitrary; either schema works at any scale.
- Which is "more modern" — both from Kimball's methodology; both alive.

**Why star usually wins in analytics:**
- Analytics engines optimize for scans + joins to a small dimension. Fewer joins → faster.
- Storage is cheap; denormalized dims cost ~nothing at warehouse scale.
- Column pruning and dictionary encoding make redundancy almost free.

Deeper: [Data Modeling](./data-engineer/02-data-modeling.md).

### 2.2 Fact Table Grain — Declare It at the Most Atomic Level

**Grain = what a single row in the fact table represents.**

The rule: declare the grain at the **most atomic (finest) level the business process produces**. If a ticket goes through `open → in_progress → resolved`, the grain is *one row per status change*, not one row per ticket.

Why atomic:
- **You can always aggregate up** — atomic → coarse via `GROUP BY` — but you can never disaggregate.
- **You retain full lineage** for causal analysis (why did resolution time spike? which state took long?).
- **Future questions** you haven't thought of yet often need the atomic level.

**Rules that follow:**
- **Never mix grains in one fact table.** A row that means "one order" and another that means "one order line" in the same table is a bug.
- **Multiple fact rows sharing a dimension foreign key is normal.** It's not a referential-integrity violation. That's exactly what "many facts per dimension" means.
- **Grain declaration is the first design decision.** Everything (dim modeling, aggregation strategy, partitioning) flows from it.

### 2.3 OLTP vs OLAP

| | OLTP | OLAP |
|---|---|---|
| Workload | Many small transactions | Large analytical scans |
| Operations | Point reads, single-row inserts/updates | Aggregations over many rows |
| Guarantees | ACID, high concurrency, low latency per txn | Read-optimized, columnar, high throughput |
| Storage | Row-oriented (usually B+ tree indexed) | Column-oriented (Parquet, ORC, columnar warehouse) |
| Example | E-commerce checkout: 10k orders/sec with stock + balance consistency | BI dashboard scanning 100M rows for revenue by category |

**Common trap:** "high-throughput concurrent single-record updates with consistency" → OLTP (Postgres, MySQL), *never* OLAP (Snowflake, BigQuery, Redshift). OLAP engines are designed for scans; they generally reject or slowly serve single-row updates.

Deeper: [Warehouses & Lakehouses](./data-engineer/03-warehouses-lakes-lakehouse.md).

---

## 3. Streaming & Pipeline Architecture

### 3.1 Lambda Architecture — the Precedence Rule

Lambda is:
- **Speed (stream) layer** — low-latency, approximate, fills the gap for recent data (last few minutes to hours).
- **Batch layer** — authoritative, deterministic, corrects results retroactively (nightly / hourly reprocess).
- **Serving layer** — merged view.

**The critical rule most miss:** once the batch layer finalizes a time window, its output **must take precedence** over the speed layer's earlier approximation. The classic Lambda flaw is a missing precedence / expiry mechanism — the speed layer keeps overwriting already-corrected windows, and the "corrected" values silently regress to approximations.

Concrete implementations of precedence:
- **Time-window ownership:** batch owns [t-∞, t-1h], stream owns [t-1h, now]. Serve batch for old, stream for new.
- **Version columns** on serving-layer rows — batch writes with `version = 'batch'`, always wins in a merge.
- **TTLs on stream-layer rows** — stream-layer rows expire once the batch layer catches up.

**Common traps:**
- "Lambda always gives consistent output" — false without a precedence mechanism.
- "Speed layer results are always authoritative" — false; they're approximate.
- "Kappa is strictly better than Lambda" — Kappa is simpler but requires that streaming be able to reprocess history from replay. Lambda persists as a hedge when reprocessing is expensive.

### 3.2 Late-Arriving Events in Stream Processing

Streams have stragglers. Events happen at time T (event time) but arrive at time T + δ (processing time). The right architecture:

- **Event-time processing** — bucket events by *when they happened*, not *when we saw them*. Requires the event to carry a trustworthy timestamp.
- **Watermarks** — the stream engine's estimate of "we've seen everything up to event-time W." Advances as new events arrive.
- **Allowed lateness** — keep the window open for a configured tolerance past its nominal close, so a late-arriving event can still land in the correct event-time window.
- **Side output / dead-letter path** — events past the lateness budget go here for batch reconciliation.

**Why this matters:** without watermarks + allowed lateness, an event delayed by network hiccup gets counted in the wrong window (or dropped), and stream results silently diverge from batch.

Frameworks:
- **Flink** — the canonical event-time engine. Watermarks are first-class.
- **Spark Structured Streaming** — supports watermarks via `withWatermark`.
- **Kafka Streams** — supports event-time with allowed grace.

Deeper: [Batch vs Streaming](./data-engineer/06-batch-vs-streaming.md).

### 3.3 Batch → Stream Migration — the Real Trade-offs

**What NOT to do:**
- **Widen micro-batch intervals** to "make the stream more batch-like." Defeats the purpose; you now have a slow batch job with all the operational overhead of streaming.
- **Buffer for exact global ordering.** Unbounded streams have stragglers forever; you'd never emit.
- **Assume state is free.** Streaming maintains state (windows, aggregates, joins) that batch discards each run. Streaming state grows without discipline.

**What to do:**
- **Match the streaming latency SLA to the actual business need.** If nightly is fine, stay batch.
- **Design idempotent sinks** so at-least-once + dedup is easier than exactly-once from scratch.
- **Externalize state** to Kafka + RocksDB or Flink checkpoints so the stream can recover.
- **Plan reprocessing** (replay from Kafka, restart from checkpoint, backfill from batch).

### 3.4 Async Processing (Web Apps) — the Actual Benefit

The primary benefit: **the server returns a response immediately** while the long work runs in the background, so the request thread isn't blocked.

**What async does NOT do:**
- Eliminate error handling — errors now happen out-of-band; you need a way to report them (status endpoint, webhook, retry queue).
- Guarantee completion — the background work can fail; you need retries and DLQs.
- Make anything faster — total work is unchanged; you just decouple response time from work time.

**Common trap:** the wrong answers usually say async "removes the need for error handling" or "guarantees the work finishes." Both false.

Deeper: [ETL vs ELT — idempotency & DLQ](./data-engineer/04-etl-vs-elt.md).

---

## 4. Delta Lake / Lakehouse — Time Travel and Recovery

### 4.1 Time Travel Fundamentals

Delta Lake keeps a **transaction log** (`_delta_log/` directory of JSON files + periodic Parquet checkpoints) recording every table version. Every commit creates a new version; every version references a set of Parquet data files.

Time travel query:

```sql
SELECT * FROM my_table VERSION AS OF 42;
SELECT * FROM my_table TIMESTAMP AS OF '2026-01-01T00:00:00Z';
```

Rollback / recovery:

```sql
RESTORE TABLE my_table TO VERSION AS OF 42;
RESTORE TABLE my_table TO TIMESTAMP AS OF '2026-01-01T00:00:00Z';
```

`RESTORE` writes a new commit that reverts the table's state — the old versions still exist in the log, so it's reversible.

### 4.2 The Three Delta Traps That Destroy History

**1. `VACUUM` deletes old unreferenced files.**

```sql
VACUUM my_table RETAIN 168 HOURS;
```

After the retention period, any Parquet file not referenced by a *current* version is deleted. **If you run `VACUUM` with a short retention, you can no longer time-travel past that point** — the log entries still exist but the data files are gone.

Default retention is 7 days; the `spark.databricks.delta.retentionDurationCheck.enabled=false` flag lets you go lower but Databricks intentionally makes it hard because this is destructive.

**Practice:** never `VACUUM` a table you might need to recover from. Set retention conservatively (30+ days for production).

**2. Shallow clone references the source files.**

```sql
CREATE TABLE clone_of_prod SHALLOW CLONE prod_table;
```

The clone has its own log but **points at the source's Parquet files.** If `prod_table` is corrupted, so is `clone_of_prod`. Also, if you `VACUUM prod_table`, the clone's referenced files can be deleted underneath it.

For a true recovery-friendly copy, use **DEEP CLONE**:

```sql
CREATE TABLE backup_of_prod DEEP CLONE prod_table;
```

This copies the actual data files. Slow, expensive, but independent.

**3. Manually replaying the transaction log is unsupported.**

The `_delta_log/` files look like readable JSON, and someone will suggest "just skip the bad commit." Don't. The log has ordering, checkpoint interactions, and consistency invariants that aren't documented as a stable API. Use `RESTORE`.

### 4.3 The Iceberg/Delta/Hudi Parallel

All three lakehouse table formats support time travel via a similar snapshot log mechanism. The traps generalize:

- Iceberg's `expire_snapshots` = Delta's `VACUUM`. Same destructiveness.
- Iceberg's `remove_orphan_files` = Delta's file cleanup.
- All three: shallow clones share files with the source; deep clones don't.

Deeper: [Table Formats — Iceberg, Delta, Hudi](./data-engineer/08-table-formats.md).

---

## 5. Snowflake — Warehouse Sizing vs Concurrency

### 5.1 The Two Independent Knobs

Snowflake separates **per-query power** from **concurrency**. Get these confused and you either overspend or get queuing.

- **Warehouse size** (XS, S, M, L, XL, 2XL, ..., 6XL) — per-query compute. Doubling the size roughly doubles per-query speed for well-parallelizable queries. Larger warehouses have more nodes, more memory, more slots.
- **Multi-cluster warehouse** — an auto-scaler that spins up *multiple identical warehouses* when queries queue. Each cluster serves an independent set of concurrent queries; results are merged transparently. Scales **out** to handle concurrency, then scales **in** when load drops.

### 5.2 The Right Answer for "Peak Concurrency"

For **200 concurrent queries in two daily windows**:

- ❌ Permanently size up (M → 2XL) — you get faster individual queries but the *slots* are still limited; you overprovision off-peak.
- ❌ Rely on the result cache — only helps *identical repeated* queries within 24h.
- ✅ **Multi-cluster warehouse with auto-scale.** Configure `MIN_CLUSTER_COUNT = 1`, `MAX_CLUSTER_COUNT = 10`, `SCALING_POLICY = 'STANDARD'`. At peak, Snowflake spins up more clusters; off-peak, they suspend.

### 5.3 The Load vs Query Separation

Best-practice pattern:

- **Load warehouse (`LOAD_WH`, sized L)** — dedicated to `COPY INTO`, Snowpipe, and heavy DBT model builds. Bursty, high-CPU, batch-friendly.
- **Query warehouse (`BI_WH`, multi-cluster S)** — serves Looker / QuickSight / analyst ad-hoc. Consistent low-latency, concurrency-heavy.
- **ETL warehouse (`ETL_WH`, sized M with auto-suspend 60s)** — scheduled transforms.

Each warehouse has independent auto-suspend, so idle warehouses cost nothing. Shared warehouses cause noisy-neighbor lag.

### 5.4 Common Traps

- "Auto-suspend saves cost" — true; default is 10 minutes; set aggressively (60s) for bursty workloads.
- "Larger warehouse = more concurrency" — false. Concurrency comes from multi-cluster, not size.
- "Result cache handles concurrency" — only for identical queries (query-text hash match, no underlying data change). Real dashboards vary parameters constantly.

---

## 6. NoSQL Data Model Selection

The rule: **match the model to the access pattern**, not the buzzword.

| Model | Access pattern that wins | Example | Why it wins |
|---|---|---|---|
| **Key-value** | Written once, read many by a single known key | Session cache keyed by session ID; user profile by user ID | O(1) point lookup; no scan overhead |
| **Wide-column** | Composite keys enabling range scans within a partition | "All sessions for user X, ordered by start time" | Partition key + clustering key; Cassandra, ScyllaDB, BigTable |
| **Document** | Self-contained records retrieved as a unit | Product catalog entries, order documents | Schema flexibility; embedded arrays / objects; MongoDB, Couchbase |
| **Graph** | Relationship-heavy, multi-hop queries | "Find all products bought by friends of my friends" | Traversals in O(deg × depth) rather than N joins; Neo4j, TigerGraph |
| **Time-series** | High-write, time-range queries on metrics | Sensor readings, application metrics | Optimized ingest + time-range scans; TimescaleDB, InfluxDB |
| **Search** | Full-text or fuzzy queries with ranking | Product search, log search | Inverted index; Elasticsearch, OpenSearch |
| **Vector** | Approximate-nearest-neighbor by embedding | RAG retrieval, similarity search | HNSW / IVF-PQ indexes; Pinecone, Weaviate, pgvector |

### 6.1 The Access-Pattern Interview Move

When asked "which NoSQL for X," **restate the access pattern first**:

> "The access pattern is 'read by known ID' — no scans, no ranges, no joins. That's a key-value pattern. Redis or DynamoDB."

If you can't state the access pattern in one sentence, you can't pick the right store.

### 6.2 Traps

- "MongoDB for time-series" — technically works, but purpose-built time-series DBs (TimescaleDB, InfluxDB) beat it on ingest and range queries by 10-100×.
- "Cassandra for arbitrary queries" — false; Cassandra requires you to model *for* the query. No secondary-index sanity like Postgres.
- "Graph DB for anything with relationships" — a customer→order relation is a foreign key, not a graph problem. Graph DBs win at *multi-hop* queries (2+ joins deep) where relational costs explode.

Deeper: [Distributed Systems Cheatsheet — SQL vs NoSQL](./data-engineer/17-distributed-systems-cheatsheet.md#sql-vs-nosql).

---

## 7. Vector Search & RAG

### 7.1 ANN (Approximate Nearest Neighbor) Indexing

Brute-force exact search over N vectors is O(N × d) per query — for N = 10M vectors of d = 768 dims, that's ~7.7B ops per query. Seconds.

**ANN indexes** trade a small recall loss for sublinear query time:

- **HNSW (Hierarchical Navigable Small Worlds)** — graph-based, expected O(log N × d) queries, high recall (95%+). The default in Pinecone, Weaviate, Qdrant, pgvector, Elasticsearch dense-vector.
- **IVF (Inverted File)** — cluster vectors with k-means into K centroids; at query time, search only the top-`nprobe` centroids. `IVF-PQ` adds product quantization for memory compression.
- **PQ (Product Quantization)** — compress each d-dim vector to a small code (~16 bytes vs 3 KB); typically combined with IVF for large scale.
- **DiskANN** — SSD-resident graph, serves 1B+ vectors from a single VM. Microsoft's Bing uses this.

**Common traps** — what does NOT fix slow ANN search:

- "Add more shards" → adds parallelism but each shard still does brute force.
- "Change distance metric" → cosine vs L2 doesn't change complexity.
- "Reduce embedding dimensions" → the embedding model produces a fixed dim; changing it means using a different model, which changes retrieval quality entirely.

**The right answer:** use an ANN index (HNSW / IVF-PQ).

Deeper: [Storage & Vector Search Structures](./python/13-storage-and-vector-structures.md).

### 7.2 Chunking Trade-offs

- **Smaller chunks** → tighter unit of meaning → higher retrieval **precision**, but concepts that span boundaries get **fragmented**.
- **Larger chunks** → preserves reasoning chains, but retrieval is fuzzier and wastes context budget on irrelevant material.
- **Chunk overlap (sliding window, 10–20%)** → carries preceding context forward, preserving reasoning chains across boundaries. The fix for "missing intermediate steps" in retrieved chunks.
- **Semantic chunking** → split at natural boundaries (paragraph, heading, topic shift) rather than fixed character count. Best precision but slowest to build.
- **Parent-child chunking** → embed small chunks for precision, return the larger parent for context. Best of both.

**Embedding dimensionality is fixed by the model.** OpenAI `text-embedding-3-large` is 3072-dim; you cannot "reduce it for shorter chunks." You can:
- Use a smaller model (different embedding).
- Apply Matryoshka reduction if the model supports it (some newer models do).
- PCA the vectors down (loses quality; rarely worth it).

### 7.3 NLI for Hallucination Detection

NLI (Natural Language Inference) models classify whether a hypothesis is **entailed / contradicted / neutral** given a premise. Used in RAG as a fact-checker: does the answer entail from the retrieved chunks?

**Real strength:** catches answers that flatly contradict the context.

**Real limitation:** NLI models often **miss numerical errors, date errors, and factual misattributions** because they're syntactically plausible. "The company was founded in 2007" vs "The company was founded in 2005" — the NLI model may score both as entailed if the premise mentions a founding date at all.

**Consequence for RAG design:**
- Don't rely on NLI as sole fact-checker for numbers/dates/names.
- Combine with exact-match regex on numeric extractions, structured extraction verification, or a stronger LLM-as-judge for those slots.
- RAGAS framework provides "faithfulness" score via LLM judgment, which is more robust than NLI alone.

Deeper: [RAG Fundamentals](./ai/rag/01-rag-fundamentals.md).

---

## 8. Agentic AI Patterns

### 8.1 ReAct (Reason + Act) — the Foundational Loop

ReAct interleaves reasoning traces and tool actions in a loop:

```
Thought: I need current AAPL price.
Action: search("AAPL price today")
Observation: $187.42
Thought: Now compute P/E — need EPS.
Action: search("AAPL EPS TTM")
Observation: $6.42
Thought: 187.42 / 6.42 = 29.19
Answer: AAPL P/E ≈ 29.2.
```

**Key properties:**
- Each observation feeds the next reasoning step.
- The model decides when it's done and emits a final answer.
- Requires tool integration + a driver loop; not a single-turn LLM call.

**Common traps:**
- "ReAct requires a vector DB" — false; ReAct is agnostic to what the tools are.
- "ReAct requires multiple model sizes" — false; typically one LLM.
- "ReAct is the same as chain-of-thought" — no. CoT is single-turn "think out loud." ReAct adds tools + iteration.

### 8.2 Plan-then-Execute vs Interleaved

- **Plan-then-execute:** LLM emits a full plan (steps 1..N) upfront, then a driver executes each step. **Limitation:** cannot adapt to actual step results — if step 3 fails or returns unexpected data, steps 4..N run on false assumptions.
- **Interleaved (ReAct-style):** LLM observes each step's output and adjusts the next thought. More adaptive, more tokens consumed.
- **Hybrid (replanning):** plan-then-execute with a "checkpoint after each step; replan if drift detected." Balances token cost with adaptivity.

**When each wins:**
- Plan-then-execute: deterministic, cheap, well-specified workflows (e.g., "extract these fields from this document").
- Interleaved: exploration, uncertain state, tool outputs that inform strategy (e.g., debugging, research).

### 8.3 Human-in-the-Loop & Automation Bias

**Automation bias:** humans stop rigorously reviewing outputs from a system they've come to trust. If an LLM approval becomes a *sufficient* gate on merges, humans rubber-stamp — the *systemic* result is that undetected errors ship.

**The wrong "fix":** ask the LLM to output confidence scores or self-classify uncertainty. This fails because the LLM cannot recognize the subtle cases it *already missed* — it wouldn't have missed them otherwise.

**The right fix is structural:**
- **Restrict the LLM to suggestions only.** Its output annotates the PR; it does not merge.
- **Keep a mandatory human approval on the merge path.** The human's action is *required*, not *optional*.
- **Rotate reviewers** so no single person becomes the "LLM's rubber stamp."
- **Sample audit** — periodically re-review LLM-approved changes with fresh eyes; use miss rate to calibrate trust.

**Common trap:** any answer that lets the LLM auto-merge below a confidence threshold is wrong. Confidence self-reports are unreliable *precisely for the failure modes you care about*.

### 8.4 Graceful Degradation on External API Failures

Pattern for calling a flaky downstream service:

1. **Retry with exponential backoff + jitter.** First retry at ~1s, next at ~2s, ~4s, ~8s, capped. Jitter (randomized delay) prevents thundering herd against a struggling service.
2. **Give up after N retries or T total time.** Don't retry forever.
3. **On final failure, produce a labelled degraded output.** Return the partial answer with an explicit `"warning": "external enrichment unavailable"` field. Never silently serve stale cache as fresh.
4. **Circuit-break** if failures are sustained — stop calling the downstream for a cooldown period, then probe.

**Anti-patterns:**
- Retry at full frequency against a struggling service (makes it worse, DDoS-yourself).
- Return stale cache without a freshness flag (users trust it).
- Fail-hard with a 500 (better than silent bad data, but degraded output is usually possible).

Deeper: [MCP Servers](./ai-coding-assistants/03-mcp-servers.md) · [Agent SDK](./ai-coding-assistants/07-agent-sdk-and-programmatic-use.md) · [Distributed Systems — Resilience Patterns](./data-engineer/17-distributed-systems-cheatsheet.md#resilience-patterns-mcq-vocabulary).

---

## 9. Monolith vs Microservices — the Trade-off Framework

The question tests whether you understand **it's a trade-off**, not doctrine.

| | Monolith | Microservices |
|---|---|---|
| Deployment | One artifact | Many independent artifacts |
| Team boundary | Feature branches | Own the service end-to-end |
| Failure isolation | Weak (one bug can take down everything) | Better (blast radius contained per service) |
| Latency | Local calls (µs) | Network hops (ms) |
| Complexity | Low ops | High ops (service discovery, tracing, retries, circuit breakers) |
| Data | Usually one DB | DB per service (mostly) |
| Best for | Small teams, predictable workload, early stage | Large orgs, mature scale, independent-scaling needs |

**The right answer** for a small team with predictable load: **monolith**. The operational overhead of microservices (service discovery, distributed tracing, config management, inter-service auth, retry policies, network debugging) is real and requires headcount to absorb.

**Absolute claims are red flags:**
- "Microservices eliminate failure risk" — false; they *shift* it.
- "Monoliths can't share state without external cache" — false; a monolith shares memory trivially.
- "Microservices are always more scalable" — false; a well-designed monolith scales fine for many workloads.

Deeper: [Distributed Systems — Microservices vs Monolith](./data-engineer/17-distributed-systems-cheatsheet.md#microservices-vs-monolith).

---

## 10. Quick-Reference Cheat Sheet (updated, expanded)

| Problem | Answer |
|---|---|
| `denied ... for routine 'sum'` (Snowflake / most SQL) | Remove space between function name and paren: `sum(` |
| Enforce PII across many consumers | Query-layer column/row-level access control (Unity Catalog, Lake Formation, Snowflake dynamic masking) |
| Reversible de-identification | Pseudonymisation (still GDPR-covered) |
| Irreversible de-identification | Anonymisation (out of GDPR scope) |
| "Encryption = anonymisation" | **False** — encryption is reversible with key |
| Star vs Snowflake schema axis | **Normalization** (not storage, keys, scale) |
| Most atomic fact grain | One row per event / state change |
| Never mix grains | Correct; separate fact tables for each grain |
| Multiple fact rows share dim FK | Normal, not a referential-integrity issue |
| 10k concurrent single-row txns w/ ACID | OLTP (Postgres, MySQL, Spanner) |
| Large scans + aggregations | OLAP (Snowflake, BigQuery, Redshift) |
| Lambda: stream overwrites batch | Missing precedence / expiry mechanism |
| Late events counted in correct window | Event-time watermarks + allowed lateness |
| Batch → stream migration: what NOT to do | Widen micro-batch intervals; buffer for global ordering |
| Primary benefit of async processing | Server returns response immediately; work happens in background |
| Recover corrupted Delta table | `RESTORE TABLE ... TO VERSION AS OF n` |
| Delta `VACUUM` risk | Deletes old files — destroys time-travel history past retention |
| Delta shallow clone | Points at source files; corruption / VACUUM in source breaks clone |
| Manual `_delta_log` editing | Unsupported, breaks invariants |
| Peak query concurrency in Snowflake | Multi-cluster warehouse auto-scaling |
| Snowflake per-query speed | Warehouse size (XS → 6XL) |
| Separate load & query warehouses | Best practice; avoids noisy neighbor |
| Write-once, read-by-key | Key-value store (Redis, DynamoDB) |
| Composite keys with range scans | Wide-column (Cassandra, BigTable) |
| Self-contained JSON records | Document (MongoDB) |
| Multi-hop relationship queries | Graph (Neo4j) |
| Slow similarity search at scale | ANN index (HNSW / IVF-PQ / DiskANN) |
| More shards fix slow ANN? | No — adds parallelism, not sublinearity |
| Concepts split across chunks | Increase chunk overlap (sliding window) |
| Reduce embedding dims to fit chunks | **No** — dim is fixed by the model |
| NLI misses which errors | Numerical, date, factual misattribution |
| Reason + Act loop | ReAct |
| Plan-then-execute limitation | Cannot adapt to actual step results |
| LLM approvals + humans rubber-stamp | Automation bias — restructure so LLM is suggestion-only |
| LLM confidence scores fix automation bias | **No** — LLM can't recognize its own subtle misses |
| Transient API failure | Retry w/ exponential backoff + jitter; on final fail, labelled degraded output |
| Retry at full frequency against struggling service | **Anti-pattern** |
| Silent stale cache as fresh | **Anti-pattern** |
| Small team, predictable load | Monolith |
| Microservices eliminate failure | **False** — they shift and structure it |

---

## 11. Cross-Section Concepts That Recur (memorize the transferable rules)

Interviewers test the same *shape* of reasoning across topics:

1. **Enforcement at a single system-controlled choke point** > convention or per-team code. (Applies to: PII access, data quality, schema evolution, auth.)
2. **Adaptive > pre-planned** when the environment can surprise you. (Applies to: ReAct vs plan, retry backoff, circuit breakers, streaming vs batch.)
3. **Structural safeguards > human vigilance.** (Applies to: HITL / automation bias, code review, on-call rotation, audit trails.)
4. **Preserve maximum atomicity / detail; aggregate on read.** (Applies to: fact grain, log storage, event sourcing, raw-zone in lake.)
5. **Match the tool to the access pattern**, not to fashion. (Applies to: NoSQL choice, OLTP vs OLAP, vector index type, SQL vs NoSQL, monolith vs microservices.)
6. **Reversibility matters — keep an undo path.** (Applies to: Delta time travel, pseudonymisation-with-key, RESTORE, staged rollouts.)
7. **Approximate but bounded > exact but slow.** (Applies to: ANN search, HyperLogLog, watermarks, retries with backoff.)
8. **Label degradation explicitly; never silently substitute.** (Applies to: RAG citations, graceful API failure, streaming late events, cache staleness flags.)

These meta-principles let you *derive* the right answer even when the specific fact is fuzzy.

---

## 12. General Approach for Multiple-Choice / Rapid-Fire Screens

- **Time budget:** ~30–60 sec per question. Flag anything > 90 sec; return after all easy ones. Time management is the biggest killer.
- **Skip and return:** most timed-test UIs support flagging; use it aggressively.
- **When 2 options look right:** pick the **more specific / more correct / less-absolute** one. Extreme wording ("always," "never," "eliminates") is usually the wrong answer; nuanced wording ("trades off," "typically," "in most cases") is usually right.
- **Watch for "NOT / never / always" phrasing** — mentally circle before answering.
- **Negative-marking check:** confirm at test start. If none, answer everything. If yes, only answer > 50%-sure ones.
- **The distractors are seeded with confident-sounding-but-subtly-wrong text.** If an answer parrots a slogan ("microservices are more scalable"), be suspicious.
- **Trust the trade-off answer.** Interviewers rarely give a doctrinal "X is always best" as the right answer — the right answer usually acknowledges the trade-off explicitly.

---

## 13. Companion Files (deep dives referenced above)

- [Data Modeling](./data-engineer/02-data-modeling.md) — star / snowflake, SCDs, grain
- [Warehouses & Lakehouses](./data-engineer/03-warehouses-lakes-lakehouse.md) — OLTP / OLAP, columnar, medallion
- [ETL vs ELT](./data-engineer/04-etl-vs-elt.md) — idempotency, DLQs, backfills
- [Batch vs Streaming](./data-engineer/06-batch-vs-streaming.md) — Kafka, Flink, watermarks, Lambda/Kappa
- [Table Formats](./data-engineer/08-table-formats.md) — Iceberg / Delta / Hudi
- [CDC](./data-engineer/09-cdc.md) — Debezium, DMS, log-based patterns
- [Data Quality & Contracts](./data-engineer/11-data-quality-contracts.md) — schema evolution, contracts
- [Partitioning & Performance](./data-engineer/13-partitioning-performance.md)
- [Governance, Lineage, Observability](./data-engineer/14-governance-lineage-observability.md) — PII, GDPR, RBAC/ABAC
- [System Design for DE](./data-engineer/15-system-design-de.md) — the framework + 4 worked examples
- [Distributed Systems Cheatsheet](./data-engineer/17-distributed-systems-cheatsheet.md) — CAP, ACID, K8s, APIs, resilience
- [SQL Deep Dive](./data-engineer/01-sql-deep-dive.md) — dialect reference and common gotchas
- [Pandas Data Manipulation](./data-engineer/16-pandas-data-manipulation.md) — pandas + pandas↔PySpark
- [LLM Fundamentals](./ai/llms/01-llm-fundamentals.md) — tokens, transformer, KV cache
- [RAG Fundamentals](./ai/rag/01-rag-fundamentals.md) — chunks, embeddings, ANN, evals
- [Prompting Fundamentals](./ai/prompting/01-prompting-fundamentals.md) — CoT, ReAct, injection defense
- [MCP Servers](./ai-coding-assistants/03-mcp-servers.md)
- [Agent SDK](./ai-coding-assistants/07-agent-sdk-and-programmatic-use.md)
- [Storage & Vector Search Structures](./python/13-storage-and-vector-structures.md) — HNSW, IVF-PQ, DiskANN
- [ML Metrics](./statistics/06-ml-metrics.md) — precision / recall / F1 / AUC / CV / leakage
- [Classical ML Algorithms](./statistics/08-classical-ml-algorithms.md) — algorithm zoo

---

## 14. Suggested Study Rhythm

**One week out** — read this file end to end, then the deep-dive companion files. Take notes on anything that surprises you.

**Three days out** — re-drill the "Common Traps" and "Quick-Reference Cheat Sheet" until the answer is instant recognition. Simulate: cover the answer column, quiz yourself on the "Problem" column.

**Night before** — 30-minute skim of §10 (cheat sheet), §11 (transferable rules), §12 (general approach). Sleep. Recognition + patterns > cramming at 2am.

**Day of** — read the question carefully. Identify the trap phrasing. Prefer the trade-off / nuanced answer. Flag uncertain ones and return.

---

*Every topic here recurs across senior DE / applied AI / system-design interviews. The depth is what lets you answer confidently, not just recognize the word.*
