# 10-Day Study Plan — Data Engineering + Applied AI Deep Revision

**Goal:** cover every important topic in this repo with enough depth to answer conceptual MCQs, defend design choices in interviews, and reason from first principles. Data Engineering finishes first (Days 1–6), then Applied AI (Days 7–9), then a capstone review (Day 10).

**Daily time budget:** 2–3 focused hours. If you have less, cut the "stretch" sub-tasks. Do not do more than 3 hours — retention drops sharply past that.

**How to read each file:**
1. Start with **why the file exists** (top of file) — 1 min.
2. Read the **BEGINNER** section carefully.
3. Skim **INTERMEDIATE** for the concepts, deep-read the parts that are new.
4. **ADVANCED** — read once, then move on.
5. Always finish with the **Interview-Ready Cheat Sheet** at the bottom.
6. If a concept doesn't stick, write it in your own words in a scratch file.

---

## Day 1 — SQL + Data Modeling Foundations

**Why:** SQL is the most-tested single topic. Data modeling shows up in every design MCQ.

**Read (~2 hrs):**
- [data-engineer/01-sql-deep-dive.md](./data-engineer/01-sql-deep-dive.md) — full file. Pay special attention to:
  - The 6-clause execution order.
  - Window functions (`ROW_NUMBER`, `RANK`, `DENSE_RANK`, `LAG`, `LEAD`, frame clauses).
  - `NULL` handling functions (`COALESCE`, `NULLIF`) + the integer-division / cast trap.
  - String functions per dialect (MySQL / MS SQL / Oracle / Postgres).
  - `UNION` vs `UNION ALL`.
  - CTEs + recursive CTEs.
- [data-engineer/02-data-modeling.md](./data-engineer/02-data-modeling.md) — full file. Internalize:
  - Star vs snowflake **normalization axis** (not storage / keys / scale).
  - Fact table **grain** — most atomic level; never mix grains.
  - Slowly Changing Dimensions (SCD Type 1 / 2 / 3 / 6).
  - Surrogate vs natural keys.

**Stretch (~30 min):**
- [data-engineer/16-pandas-data-manipulation.md](./data-engineer/16-pandas-data-manipulation.md) §14 — SQL→pandas bridge only. Just enough to reinforce SQL patterns in a second language.

**Self-check (5 min):**
1. Write the SQL for "top 3 orders per user by date" from memory.
2. Explain star vs snowflake in one sentence — is the axis storage, keys, or normalization?
3. What's the difference between `WHERE` and `HAVING`?
4. What does `NULLIF(x, 0)` protect you from?
5. `UNION` vs `UNION ALL` — which is the safer default?

---

## Day 2 — Storage & Lakehouse Layer

**Why:** OLTP-vs-OLAP and Delta time-travel show up constantly. Partitioning is the "why is my query slow" answer.

**Read (~2.5 hrs):**
- [data-engineer/03-warehouses-lakes-lakehouse.md](./data-engineer/03-warehouses-lakes-lakehouse.md) — full file. Internalize:
  - OLTP vs OLAP (row vs columnar, txn vs scan, latency vs throughput).
  - Parquet internals (row groups, column chunks, min/max stats).
  - MPP architecture.
  - Medallion (bronze/silver/gold) architecture.
  - Warehouse vs lake vs lakehouse (when each wins).
- [data-engineer/08-table-formats.md](./data-engineer/08-table-formats.md) — full file. Deep-focus on:
  - **Delta Lake time travel + `RESTORE`.**
  - **The `VACUUM` trap** (deletes old files → destroys history past retention).
  - **Shallow vs deep clone** (shallow references source files; deep is independent).
  - Iceberg hidden partitioning + partition evolution.
  - Hudi copy-on-write vs merge-on-read.
  - When to pick Delta vs Iceberg vs Hudi.
- [data-engineer/13-partitioning-performance.md](./data-engineer/13-partitioning-performance.md) — full file. Internalize:
  - File-sizing target (128 MB – 1 GB).
  - Small-files problem + compaction strategy.
  - Z-order / liquid clustering.
  - Predicate pushdown, partition pruning.

**Self-check:**
1. What happens if you run `VACUUM my_table RETAIN 24 HOURS` and then try `RESTORE ... TO VERSION 3 WEEKS AGO`?
2. Difference between shallow clone and deep clone?
3. Why do analytics engines love columnar storage?
4. What's the "small files problem" and why does it kill query performance?
5. Iceberg vs Delta — one concrete difference.

---

## Day 3 — Pipelines: ETL/ELT, Orchestration, CDC

**Why:** pipeline patterns are the daily bread of DE and heavily tested.

**Read (~2.5 hrs):**
- [data-engineer/04-etl-vs-elt.md](./data-engineer/04-etl-vs-elt.md) — full file. Internalize:
  - ETL vs ELT trade-off (compute location, governance).
  - **Idempotency patterns** (truncate-load, MERGE, partition replace).
  - **Incremental loads** (watermark, hash).
  - Backfills — partition-by-date so each day can be reprocessed in isolation.
  - Dead letter queues.
  - At-least-once vs exactly-once semantics.
- [data-engineer/05-orchestration-airflow.md](./data-engineer/05-orchestration-airflow.md) — full file. Internalize:
  - DAG structure, TaskFlow API.
  - Schedule semantics + `{{ ds }}`.
  - Sensors (especially deferrable).
  - Pools (concurrency limiting).
  - Datasets (data-aware scheduling).
  - Airflow vs Dagster vs Prefect vs Step Functions.
- [data-engineer/09-cdc.md](./data-engineer/09-cdc.md) — full file. Internalize:
  - Log-based vs polling CDC.
  - Debezium architecture (Kafka Connect + connectors).
  - AWS DMS full-load + CDC.
  - `MERGE` sink pattern.
  - Outbox pattern (transactional CDC).
  - Postgres replication slots.

**Self-check:**
1. Write pseudocode for an idempotent daily aggregation.
2. What's the difference between at-least-once + idempotent sink and exactly-once?
3. Why do we prefer log-based CDC over polling?
4. What problem does the outbox pattern solve?
5. Airflow sensor vs deferrable sensor — when does it matter?

---

## Day 4 — Streaming + Data Quality + dbt

**Why:** streaming has the most trap MCQs (Lambda precedence, watermarks). Data quality and dbt round out the transformation stack.

**Read (~2.5 hrs):**
- [data-engineer/06-batch-vs-streaming.md](./data-engineer/06-batch-vs-streaming.md) — full file. Deep-focus on:
  - Kafka partitions / producers / consumers / consumer groups / offsets.
  - Kinesis vs Kafka vs RabbitMQ vs SQS.
  - Flink vs Spark Structured Streaming vs Kafka Streams.
  - Windowing (tumbling, hopping, session).
  - **Event-time vs processing-time; watermarks + allowed lateness.**
  - **Lambda vs Kappa architectures + the precedence rule.**
  - Exactly-once semantics per layer.
- [data-engineer/11-data-quality-contracts.md](./data-engineer/11-data-quality-contracts.md) — full file. Internalize:
  - Six data quality dimensions (accuracy / completeness / consistency / timeliness / uniqueness / validity).
  - Great Expectations + dbt tests.
  - Data contracts (schema, semantics, SLAs).
  - Freshness monitoring + anomaly detection.
  - Circuit breakers on quality failures.
- [data-engineer/10-dbt.md](./data-engineer/10-dbt.md) — full file. Internalize:
  - Materializations (view / table / incremental / ephemeral / snapshot).
  - `is_incremental()` + incremental strategy.
  - Tests + macros + snapshots.
  - dbt slim CI + dbt mesh.

**Self-check:**
1. Why do watermarks matter for a nightly join between a stream and a batch table?
2. What's the Lambda precedence rule? What breaks without it?
3. Give an example of an at-least-once source + idempotent sink combo achieving effectively-exactly-once.
4. What's the difference between a dbt view and an incremental model?
5. Great Expectations vs dbt tests — when would you use each?

---

## Day 5 — Cloud, Governance, Distributed Systems

**Why:** cloud specifics (AWS + Snowflake concurrency) and CAP/ACID/BASE are core MCQ material. PII / anonymisation is a whole section.

**Read (~2.5 hrs):**
- [data-engineer/12-aws-data-stack.md](./data-engineer/12-aws-data-stack.md) — full file. Internalize:
  - S3, Glue (catalog / crawlers / ETL), Athena (partition projection), Redshift (DISTKEY / SORTKEY).
  - Kinesis Data Streams vs Firehose vs Data Analytics.
  - MSK, DMS, Lake Formation, Step Functions, MWAA.
  - IAM roles, encryption at rest (KMS), Macie for PII scanning.
- [data-engineer/14-governance-lineage-observability.md](./data-engineer/14-governance-lineage-observability.md) — full file. Deep-focus on:
  - **Anonymisation vs pseudonymisation** (reversibility axis, GDPR status).
  - PII enforcement at the query layer (column/row-level access control).
  - GDPR / CCPA / HIPAA / PCI / SOX regimes.
  - RBAC vs ABAC.
  - OpenLineage, DataHub, Marquez, Atlan.
  - GDPR delete pattern (partition-friendly).
  - Audit trails, DR.
- [data-engineer/17-distributed-systems-cheatsheet.md](./data-engineer/17-distributed-systems-cheatsheet.md) — full file. Internalize:
  - **CAP theorem, PACELC, consistency models.**
  - **ACID vs BASE + isolation levels.**
  - Sharding (range / hash / consistent hashing / rendezvous).
  - Replication (sync / async / leaderless).
  - Caching strategies + eviction policies + cache problems (stampede / penetration / avalanche).
  - Load balancing (L4 vs L7).
  - Docker + Kubernetes basics.
  - REST / GraphQL / gRPC.

**Self-check:**
1. Encryption ≠ anonymisation — why?
2. How would you enforce "analysts see masked emails, admins see raw" across 50 downstream teams?
3. CAP theorem: what does a leaderless database like Cassandra sacrifice during a partition?
4. Cache-aside vs write-through — when does each win?
5. Consistent hashing solves what specific problem?

---

## Day 6 — Applied DE Capstone: Spark + System Design + Review

**Why:** Spark is the last big DE deep-dive; system design ties everything together; today closes DE.

**Read (~2.5 hrs):**
- [data-engineer/07-spark-fundamentals.md](./data-engineer/07-spark-fundamentals.md) — full file. Internalize:
  - Driver / executor / task model.
  - Lazy evaluation, RDD lineage, DAG scheduler.
  - Narrow vs wide transformations, shuffle mechanics.
  - Join strategies (broadcast / sort-merge / shuffle-hash).
  - Adaptive Query Execution (AQE).
  - Skew handling (salting, AQE skew).
  - Debugging via Spark UI.
  - PySpark vs Scala vs SQL API.
- [data-engineer/15-system-design-de.md](./data-engineer/15-system-design-de.md) — full file. Deep-focus on:
  - The six-step DE system design framework (clarify → interfaces → sketch → specialize → trade-offs → operate).
  - The eight big DE trade-offs table.
  - Capacity estimation cheat sheet.
  - The 4 worked examples (daily analytics, real-time fraud, multi-DB CDC to lakehouse, 1B-event clickstream).

**Review (~30 min):**
- Skim [data-engineer/README.md](./data-engineer/README.md) — make sure every topic in the index sounds familiar.
- Skim your notes from Days 1–5. Any concept still fuzzy? Re-read that section.

**Self-check — the DE end-of-track quiz:**
1. Design an idempotent pipeline that ingests CDC from Postgres into an Iceberg lakehouse with 5-min freshness. Sketch it out on paper.
2. When does Spark pick a broadcast join over a sort-merge join?
3. What's data skew, and how do you detect + fix it in Spark?
4. Compare the write path of a B+ tree, an LSM tree, and Delta Lake.
5. You need 200 concurrent Snowflake queries at peak. What do you configure?

If you can answer these confidently, DE track is done. If any is fuzzy, spend 30 more minutes on the relevant file before moving to AI.

---

## Day 7 — LLM Fundamentals + Prompting

**Why:** every Applied AI MCQ starts with LLM basics. Prompting is close behind.

**Read (~2.5 hrs):**
- [ai/llms/01-llm-fundamentals.md](./ai/llms/01-llm-fundamentals.md) — full file. Internalize:
  - Tokens, tokenization, context window.
  - Temperature, top-p sampling.
  - Transformer architecture (attention Q/K/V, positional encoding, layer norm, FFN).
  - Causal masking (decoder-only).
  - KV cache (why inference is memory-bound).
  - MQA / GQA / sliding-window attention.
  - Quantization (INT4 / INT8 / GGUF).
  - Speculative decoding.
  - Chinchilla scaling laws.
  - Reasoning models (o1/o3-style).
- [ai-coding-assistants/04-token-optimization-and-context.md](./ai-coding-assistants/04-token-optimization-and-context.md) — full file. Internalize:
  - What eats tokens in a session (system prompt, CLAUDE.md, tools, files, history).
  - Context management strategies.
  - Caching, prefix caching.
- [ai/prompting/01-prompting-fundamentals.md](./ai/prompting/01-prompting-fundamentals.md) — full file. Deep-focus on:
  - Zero-shot vs few-shot.
  - **Chain-of-thought (CoT) + Self-consistency.**
  - **ReAct (reason + act loop).**
  - Structured output prompting.
  - Prompt chaining.
  - **Prompt injection + defense (delimiters, hierarchy, canary tokens).**
  - Temperature and sampling strategies.

**Self-check:**
1. Explain attention (Q/K/V) in two sentences.
2. What does KV cache do and why is it the inference bottleneck?
3. When do you pick temperature 0 vs 0.7?
4. Chain-of-thought vs ReAct — key difference?
5. How would you defend a customer-service chatbot against prompt injection?

---

## Day 8 — RAG + Vector Search

**Why:** RAG is 30–40% of Applied AI questions. Vector index specifics (HNSW, IVF) are recurring MCQs.

**Read (~2.5 hrs):**
- [ai/rag/01-rag-fundamentals.md](./ai/rag/01-rag-fundamentals.md) — full file. Deep-focus on:
  - The end-to-end pipeline (ingest → chunk → embed → store → retrieve → generate).
  - **Chunking strategies** (fixed / recursive / semantic / parent-child) + **overlap** for cross-boundary reasoning.
  - Embedding models + asymmetric search.
  - Vector DB choice (Pinecone / Weaviate / Qdrant / Chroma / pgvector).
  - Dense vs sparse vs hybrid retrieval + reranking.
  - RAGAS evaluation (context relevance / faithfulness / answer relevance / correctness).
  - Agentic RAG.
  - HyDE, query decomposition, step-back prompting.
  - Semantic caching.
  - **When RAG loses to long context and vice versa.**
  - Failure modes (missing chunks, hallucinate despite context, stale index).
- [python/13-storage-and-vector-structures.md](./python/13-storage-and-vector-structures.md) — sections on vector search specifically. Internalize:
  - Flat (brute-force) index — O(N × d), only viable for small N.
  - **IVF (Inverted File)** — cluster then search top-nprobe centroids.
  - **PQ (Product Quantization)** — compress vectors to small codes.
  - **HNSW (Hierarchical Navigable Small Worlds)** — graph-based ANN, expected O(log N × d), high recall. **Current default.**
  - **DiskANN** — SSD-resident for billion-scale.
  - Distance metrics (cosine vs L2 vs inner product).
  - Which index to pick at what scale.
- [ai-coding-assistants/01-project-context-files.md](./ai-coding-assistants/01-project-context-files.md) — full file. Understand:
  - CLAUDE.md and equivalents.
  - When context files help vs hurt (they're a system-prompt tax).

**Self-check:**
1. What does NOT fix slow ANN search? (Rule out: more shards, changing distance metric, reducing embedding dim.)
2. Concepts split across chunk boundaries — what's the fix?
3. Long context vs RAG — when does each win?
4. NLI-based hallucination detection: what does it miss?
5. Explain HNSW in three sentences.

---

## Day 9 — Agents, MCP, Automation Safeguards

**Why:** agentic AI is the fastest-growing MCQ topic. Automation bias appears as a "human-in-the-loop failure" question.

**Read (~2.5 hrs):**
- [ai-coding-assistants/03-mcp-servers.md](./ai-coding-assistants/03-mcp-servers.md) — full file. Internalize:
  - What MCP is (open protocol; USB for AI).
  - Tools, resources, prompts.
  - Client-server architecture; local processes.
  - Auth patterns; sandboxing.
- [ai-coding-assistants/07-agent-sdk-and-programmatic-use.md](./ai-coding-assistants/07-agent-sdk-and-programmatic-use.md) — full file. Deep-focus on:
  - Agent loop (perceive → reason → act → observe).
  - Tool use, tool schemas.
  - **ReAct pattern in code.**
  - **Plan-then-execute vs interleaved** patterns and their limitations.
  - Error recovery, retries within the loop.
  - Eval harnesses for agents.
- [ai-coding-assistants/05-hooks-security-automation.md](./ai-coding-assistants/05-hooks-security-automation.md) — full file. Internalize:
  - Hooks as deterministic guardrails.
  - Pre / post tool-use hooks.
  - Auth + policy at the tool boundary.
  - **Automation bias** and the structural fix (LLM as suggestions-only, mandatory human approval).
  - **Graceful degradation** on external API failures (backoff + jitter + labelled degraded output).
- [ai-coding-assistants/02-skills-commands-plugins.md](./ai-coding-assistants/02-skills-commands-plugins.md) — full file. Understand:
  - What skills / commands / plugins are.
  - When to reach for each.
  - How to compose them.

**Self-check:**
1. ReAct in one sentence — what makes it different from a single-turn LLM call?
2. Plan-then-execute limitation — why can it fail silently on step 4 when step 2 returned unexpected data?
3. What's automation bias, and why don't LLM confidence scores fix it?
4. Retrying a struggling downstream service — right pattern?
5. MCP vs a custom REST tool — what does MCP standardize?

---

## Day 10 — Capstone Review + ML Metrics

**Why:** synthesis and self-quizzing. If a section still feels fuzzy, this is when you catch it.

**Read (~2 hrs):**
- [interview-concepts-study-guide.md](./interview-concepts-study-guide.md) — **full read**. Treat this as the definitive synthesis. It cross-references every earlier day and adds the meta-principles that let you derive answers you don't remember. Especially:
  - §10 Quick-Reference Cheat Sheet — cover the "Answer" column, quiz yourself on the "Problem" column.
  - §11 Cross-Section Transferable Rules — internalize all 8.
  - §12 Test Strategy.
- [statistics/06-ml-metrics.md](./statistics/06-ml-metrics.md) — full file. This appears in ML-DE MCQs constantly:
  - Confusion matrix + precision / recall / F1 / F_β.
  - AUC-ROC vs AUC-PR.
  - Log loss + calibration.
  - Bias-variance trade-off.
  - CV variants (k-fold / stratified / group / **time-series**).
  - Class imbalance handling (SMOTE, class weights, threshold tuning).
  - **Data leakage sources.**
- [statistics/08-classical-ml-algorithms.md](./statistics/08-classical-ml-algorithms.md) — full file. Cram the one-line summary + the 15 MCQ traps.

**Self-check — the full sweep (~1 hr):**

Cover the answer, produce it from memory. Target: get 24/30 without hints.

**Data Engineering (Days 1–6):**
1. Star vs snowflake schema — the axis?
2. Most atomic fact grain — why?
3. OLTP vs OLAP — one-sentence separation.
4. Lambda architecture — the precedence rule?
5. Late-arriving events in streaming — the fix?
6. Async processing — the *primary* benefit and what it does NOT do?
7. Delta Lake — how do you recover from a corrupted batch load?
8. `VACUUM` risk?
9. Snowflake — knob for per-query speed vs knob for concurrency?
10. Key-value store vs wide-column — one access-pattern discriminator.
11. Encryption vs anonymisation — the distinction?
12. Enforcing PII across 20 teams — best mechanism?
13. Idempotent insert pattern — SQL sketch?
14. Broadcast join vs sort-merge join in Spark — when each?
15. CAP theorem: what does Cassandra sacrifice during partition?

**Applied AI (Days 7–9):**
16. What's a token?
17. Temperature 0 vs 0.7 — one use case each.
18. Transformer attention — one-line explanation.
19. ANN indexing — what does NOT fix slow search?
20. Chunk overlap solves what problem?
21. Embedding dim — is it tunable per chunk size? Why or why not?
22. NLI hallucination check — what does it miss?
23. ReAct — the loop.
24. Plan-then-execute limitation.
25. Automation bias — structural fix.
26. Graceful degradation on API failure — right pattern.

**ML Metrics (Day 10):**
27. When does accuracy mislead?
28. AUC-ROC vs AUC-PR — when to prefer each?
29. Precision vs recall — cancer screening example.
30. Bias-variance in one line.

If you get **≥ 24/30**, you're solid. If **20–23**, spend one more hour on your weakest section. If **< 20**, extend to Day 11 with a focused re-read of the weakest 2–3 topics.

---

## Rules for the 10 Days

1. **One day per plan-day.** Don't compress; retention comes from spacing.
2. **Do the self-check at the end of every day.** If you can't answer a question, note it and re-read that section tomorrow morning (10 min) before starting the new day.
3. **Sleep matters more than the last hour of reading.** Long-term memory consolidates during sleep.
4. **No new topics on Day 10.** Only synthesis and self-quiz.
5. **Write short notes** in your own words for anything that felt hard — the act of rewriting locks it in.
6. **Test yourself on the "Quick-Reference Cheat Sheet"** of every file — this is what MCQs recognize.
7. **If you're behind by end of Day 6:** cut Spark from Day 6 (it's the least MCQ-tested of the DE topics) and prioritize System Design + Review. Everything else is non-negotiable.

---

## Optional Day 11 (buffer)

If you skipped anything or want extra practice:

- Re-do the Day 10 self-check without notes.
- Read [statistics/03-hypothesis-testing.md](./statistics/03-hypothesis-testing.md) if any DS-flavored role is in play.
- Read [python/06-hash-tables.md](./python/06-hash-tables.md) and [python/13-storage-and-vector-structures.md](./python/13-storage-and-vector-structures.md) for infra-level MCQ depth.
- Practice a few of the 8 tasks in [data-engineer/16-pandas-data-manipulation.md](./data-engineer/16-pandas-data-manipulation.md) §17.

---

## Progress Tracker

Print this and check each box as you finish it.

- [ ] **Day 1** — SQL + Data Modeling
- [ ] **Day 2** — Warehouses / Lakehouses / Table Formats / Partitioning
- [ ] **Day 3** — ETL/ELT / Orchestration / CDC
- [ ] **Day 4** — Streaming / Data Quality / dbt
- [ ] **Day 5** — AWS / Governance / Distributed Systems
- [ ] **Day 6** — Spark / System Design / DE Review
- [ ] **Day 7** — LLM Fundamentals / Prompting
- [ ] **Day 8** — RAG / Vector Search
- [ ] **Day 9** — Agents / MCP / Automation Safeguards
- [ ] **Day 10** — Capstone + ML Metrics + Self-Quiz

---

*Follow the plan. Trust the sequence — DE topics build on each other; AI depends on the vocabulary from Day 7. By Day 10 you'll have depth on every topic in the repo, not just recognition.*
