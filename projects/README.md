# Projects — High-Value Portfolio Pieces

**Last updated:** April 24, 2026
**Sister tracks:** [AI Engineering](../ai/) · [Data Engineering](../data-engineer/)

---

## Why This Folder Exists

Certifications get you past the first résumé screen; **projects get you past the technical interview.** A hiring manager reading your résumé in 2026 will skim your bullet points in 12 seconds and ask one question: *"can this person actually build the thing on day one?"* Projects are how you answer that.

This folder ranks portfolio projects by the **interview leverage** they generate per hour invested. Each project is designed to:

1. Demonstrate a skill that appears in real 2026 job postings (not 2020 ones).
2. Have a clear "story" you can tell in a 45-min interview round.
3. Force you to make trade-offs you'll be grilled on.
4. Produce an artifact (repo, blog post, demo video) you can link from your résumé.

---

## How to Use This List

- **Pick 2 projects, not 7.** A polished, well-documented project beats five half-finished ones every time. Hiring managers can smell a "tutorial-followed" repo at 50 paces.
- **Build in public.** A blog post per project is worth more than the code itself. Write up the trade-offs you made, the dead ends, the cost numbers.
- **Pair with a certification.** A project that backs up the claim on your résumé compounds: *"AWS DEA-C01 certified + built [Project X]"* beats either alone.
- **Use real data.** Don't invent toy data. Use a public dataset (NYC taxi, Stripe sample, Common Crawl, OpenStreetMap, GH Archive) so the volume matches reality.

---

## The Project Roadmap (Ranked by Interview Leverage)

### Tier S — Highest Leverage (do at least one of these)

#### S1. End-to-End Lakehouse with CDC, Iceberg, and dbt
**Discipline:** Data Engineering
**Effort:** 30–50 hours
**Pairs with:** [AWS DEA-C01](../data-engineer/README.md#certifications)

Build a pipeline that ingests CDC from a Postgres OLTP into an S3 + Iceberg lakehouse, models the data with dbt on Athena, and exposes the result via QuickSight or Streamlit.

**Concepts demonstrated:**
- [CDC](../data-engineer/09-cdc.md) (Debezium or DMS)
- [Iceberg](../data-engineer/08-table-formats.md) (partition evolution, time travel)
- [dbt](../data-engineer/10-dbt.md) (incremental models, tests, contracts)
- [Airflow](../data-engineer/05-orchestration-airflow.md) (orchestration)
- [Data quality](../data-engineer/11-data-quality-contracts.md) (Great Expectations or dbt tests)
- [Governance](../data-engineer/14-governance-lineage-observability.md) (Glue catalog, column-level masking)

**Source dataset suggestion:** [Stripe sample data](https://github.com/stripe/example-data) seeded into Postgres + simulated transactions via a load script.

**Stretch goals:**
- Run on AWS for $20/mo via free tier + Glue/Athena pay-per-query.
- Add OpenLineage emission for [lineage](../data-engineer/14-governance-lineage-observability.md).
- Implement a GDPR-delete job that runs in O(partition).

**Interview story:** "I built this end-to-end and discovered MERGE-on-read killed read performance until I added a 4-hour compaction job. Here's the cost graph and how I tuned it."

---

#### S2. RAG System over a Real Corpus with Evals
**Discipline:** AI Engineering
**Effort:** 25–40 hours
**Pairs with:** [AWS AI Practitioner / ML Specialty](../ai/) and résumé claims of "shipped LLM features"

Build a [RAG](../ai/rag/01-rag-fundamentals.md) system over a non-trivial corpus (your company's docs, a 10K filing, the Pile of Law, arXiv) with a real eval harness that scores retrieval quality, faithfulness, and answer correctness.

**Concepts demonstrated:**
- [Embeddings](../ai/rag/01-rag-fundamentals.md) and chunking strategies
- Vector DB selection (Pinecone vs pgvector vs Chroma vs Weaviate)
- Hybrid search (BM25 + dense)
- Reranking (Cohere Rerank, ColBERT)
- [Prompting](../ai/prompting/01-prompting-fundamentals.md) for RAG
- Evals (RAGAS, custom LLM-as-judge)
- Cost monitoring per query

**Source dataset suggestion:** [arXiv abstracts](https://www.kaggle.com/datasets/Cornell-University/arxiv) or your company's public docs.

**Stretch goals:**
- A/B harness comparing two embedding models or chunking strategies.
- A simple Streamlit UI with citations.
- A blog post showing the eval scores before/after each design change.

**Interview story:** "Naive chunking dropped retrieval recall to 60%. I switched to semantic-aware chunks + a reranker and recall jumped to 88%. The eval harness made the trade-off measurable."

---

#### S3. Real-Time Streaming Analytics with Kafka, Flink, and ClickHouse (or Iceberg)
**Discipline:** Data Engineering (senior signal)
**Effort:** 40–60 hours
**Pairs with:** [Confluent Certified Developer](../data-engineer/README.md) or AWS DEA-C01

Build a clickstream pipeline that processes 1M events/day through [Kafka](../data-engineer/06-batch-vs-streaming.md) → [Flink](../data-engineer/06-batch-vs-streaming.md) → ClickHouse (or Iceberg + Athena), with a real-time dashboard.

**Concepts demonstrated:**
- Streaming [windowing](../data-engineer/06-batch-vs-streaming.md) and watermarks
- Exactly-once semantics
- Schema registry + Avro
- [Backpressure](../data-engineer/06-batch-vs-streaming.md) and lag monitoring
- Sub-second OLAP serving

**Source dataset suggestion:** Generate clickstream synthetically via a Python load generator (or replay [GH Archive](https://www.gharchive.org/) events).

**Interview story:** "When I sized the Kafka cluster I assumed 1KB events; real events were 8KB. Throughput dropped 8×. I added compression at the producer and re-ran the capacity calculation. Here's the dashboard."

---

### Tier A — Strong Differentiators

#### A1. Multi-Source Identity Resolution Service
**Discipline:** Data Engineering with strong systems-design flavor
**Effort:** 30–40 hours

Take 3+ messy data sources (e.g. Salesforce CRM, Stripe customers, support tickets) and build a deduplication / identity-resolution pipeline using deterministic + probabilistic matching (e.g. [Splink](https://moj-analytical-services.github.io/splink/) or [Zingg](https://www.zingg.ai/)).

**Demonstrates:**
- [SQL](../data-engineer/01-sql-deep-dive.md) at high difficulty
- [Data modeling](../data-engineer/02-data-modeling.md) (master data management)
- Probabilistic record linkage
- [Data quality](../data-engineer/11-data-quality-contracts.md)

**Interview story:** "Deterministic matching caught 60%; adding probabilistic with a 95% threshold caught another 25% with a false-positive rate I measured at 0.3%."

---

#### A2. Agentic AI Workflow with MCP Tools
**Discipline:** AI Engineering
**Effort:** 20–30 hours

Build an [agent](../ai-coding-assistants/07-agent-sdk-and-programmatic-use.md) using the Anthropic SDK or LangGraph that uses [MCP servers](../ai-coding-assistants/03-mcp-servers.md) to interact with real services (your GitHub, a SQL database, a Slack workspace) to complete a multi-step task.

**Demonstrates:**
- Agent loops, tool use, error recovery
- [MCP](../ai-coding-assistants/03-mcp-servers.md) protocol
- [Token optimization](../ai-coding-assistants/04-token-optimization-and-context.md)
- Eval harness for agent reliability

**Interview story:** "The agent kept hallucinating tool schemas until I tightened the system prompt and added a self-validation step. I have a regression suite of 30 tasks; pass rate went from 40% to 91%."

---

#### A3. Cost-Optimized ML Feature Store on AWS
**Discipline:** Data + ML Engineering
**Effort:** 35–50 hours

Build a feature store split into online (DynamoDB / ElastiCache) and offline (Iceberg) using [Spark](../data-engineer/07-spark-fundamentals.md) for batch features and [Flink](../data-engineer/06-batch-vs-streaming.md) for streaming features. Show the same feature consumed by training and inference with consistency.

**Demonstrates:**
- Lambda-style architecture
- [Feature store](https://www.featurestore.org/) concepts
- Train/serve skew monitoring

---

#### A4. SQL Engine in Rust / Python (toy but instructive)
**Discipline:** Data Engineering, depth signal
**Effort:** 60–80 hours

Implement a tiny SQL engine that parses, plans, and executes a subset of SQL over Parquet files. Apache DataFusion is a good study reference.

**Demonstrates:**
- Deep [SQL](../data-engineer/01-sql-deep-dive.md) understanding
- [Query optimization](../data-engineer/13-partitioning-performance.md)
- Vectorized execution
- Columnar storage internals

This is an "I'm going for staff" signal. Don't do it for a mid-level role.

---

### Tier B — Solid but Common (do only if Tier S/A doesn't fit your time budget)

#### B1. NYC Taxi Pipeline (the canonical DE first project)
S3 → Spark → Redshift / Snowflake. Useful but every candidate has done it. Differentiate by adding [data quality](../data-engineer/11-data-quality-contracts.md) gates, [lineage](../data-engineer/14-governance-lineage-observability.md), and a published cost comparison across warehouses.

#### B2. Snowflake / Databricks Hands-On Cert Project
Pick the [SnowPro Core](../data-engineer/README.md) or Databricks DE Associate cert and build the official capstone project.

#### B3. Fine-tune a Small LLM on a Domain
Take a 7B model and LoRA-fine-tune it on a domain corpus. Useful only if you're targeting an ML/AI role; less leverage if you're targeting DE.

---

## Project Documentation Template

For each project you ship, write a README that answers:

```
1. The problem in one sentence
2. The architecture diagram (use draw.io or excalidraw)
3. The data volume + cost
4. The trade-offs you made (X vs Y, why X, what you gave up)
5. What broke and how you fixed it
6. What you'd do differently with more time
7. How to run it locally / in cloud
8. Demo video or screenshots
```

The interview round will literally walk through this README. If it's good, the round flows. If it's weak, the round becomes a survival exercise.

---

## Anti-Patterns to Avoid

- **Tutorial clones**: any repo that looks like the result of "follow this Medium post."
- **Repos with no README**: signals "I didn't really do this."
- **No data volume**: a pipeline that handles 100 rows is a tutorial. 100M rows is a project.
- **No cost discussion**: senior engineers always know what their stuff costs.
- **No tests**: the lack of tests in a portfolio repo signals "I don't ship to production."
- **Architecture that's the same as 200 other portfolios**: differentiate via a real problem, real data, real numbers.

---

## Suggested 6-Month Plan

| Month | Focus | Output |
|---|---|---|
| 1 | [Cert](../data-engineer/README.md#certifications) prep + S1 architecture sketch | Architecture diagram + cost model |
| 2 | S1 build (CDC + Iceberg + dbt) | Working pipeline + repo + blog post |
| 3 | Cert exam + start S2 | Pass DEA-C01; RAG corpus + chunker |
| 4 | S2 evals + writeup | Eval harness + blog post |
| 5 | A2 (agent) build + iteration | Agent repo + eval suite |
| 6 | Polish, blog, interview prep | All projects in `pinned` repos on GitHub |

---

## Resources for Project Inspiration

- [Data Engineering Zoomcamp (DataTalksClub)](https://github.com/DataTalksClub/data-engineering-zoomcamp) — free, project-driven curriculum.
- [Made With ML — Goku Mohandas](https://madewithml.com/) — production ML walkthroughs.
- [Awesome MLOps](https://github.com/visenger/awesome-mlops)
- [Awesome Data Engineering](https://github.com/igorbarinov/awesome-data-engineering)
- [Hello Interview's Real-World Systems](https://www.hellointerview.com/) — paid but patterns map directly to portfolio ideas.
- [Uber Engineering Blog](https://www.uber.com/en-IN/blog/engineering/data/) — clone these systems on a smaller scale.

---

## Project Files in This Folder

(Coming soon — each will get its own subfolder with full architecture, code references, and interview talking points.)

- `s1-lakehouse-cdc-iceberg/` — *(planned)*
- `s2-rag-with-evals/` — *(planned)*
- `s3-streaming-clickstream/` — *(planned)*

---

*Pick a project. Set a 4-week deadline. Ship it publicly. Repeat. That's how you build the résumé you want.*
