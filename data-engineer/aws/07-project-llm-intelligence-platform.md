# Capstone Project — LLM Intelligence Platform

**One-line pitch:** A serverless GenAI data platform on AWS that ingests only high-signal LLM content — arXiv papers, official vendor documentation (Anthropic / OpenAI / Google / Meta / Mistral / DeepSeek), HuggingFace model cards, and curated practitioner discourse — filters out AI-slop with a Bedrock-based quality classifier, extracts model comparisons + best practices, and serves semantic search across the actually-useful state of LLM knowledge.

**Focus of the build:** **AWS data engineering**. The LLM angle is a differentiator, but every design decision below is a DE decision (choice of service, cost trade-off, IAM boundary, ingestion pattern). You should be able to whiteboard this architecture in an interview and defend every arrow.

**Timeline:** 4 weekends, parallel with the [14-day study plan](./README.md#the-14-day-study-plan). Each weekend implements the AWS domain you just studied.

**Hard cost cap:** **$30 total build spend**, **$5/month idle**, **$8 in any 24-hour period**. Enforced via Budget alarm — instructions in §6 below.

---

## Table of contents

1. [Executive summary](#1-executive-summary)
2. [The approach — signal-quality tiers](#2-the-approach)
3. [Data sources](#3-data-sources) (deep, per-source)
4. [Architecture](#4-architecture)
5. [Design decisions & interview defensibility](#5-design-decisions)
6. [Cost cap strategy](#6-cost-cap-strategy)
7. [Weekend 1 build — Domain 1 (Ingestion + Transformation)](#7-weekend-1)
8. [Weekend 2 build — Domain 2 (Data Store Management)](#8-weekend-2)
9. [Weekend 3 build — Domain 3 (Operations + Analysis)](#9-weekend-3)
10. [Weekend 4 build — Domain 4 (Security + Governance)](#10-weekend-4)
11. [Common pitfalls](#11-common-pitfalls)
12. [README template + interview pitch](#12-readme-and-pitch)
13. [Extensions](#13-extensions)

---

## 1. Executive summary

### Business problem being solved

Signal-to-noise in LLM discourse is broken. Any question — "which model is best for structured extraction in November 2026?" — surfaces 40% marketing, 30% listicles, 20% Reddit anecdotes, and 10% real signal. Anyone building on LLMs wastes hours daily filtering that mess.

**This platform aggregates only the sources that actually matter**, enriches them with LLMs (meta by design), and exposes semantic search + trend analytics.

### What DE skills it proves

- **Multi-source ingestion** — 6+ sources with different shapes (REST API, Atom XML, sitemap crawl, Firebase streaming, OAuth-authenticated REST).
- **Rate-limit engineering** — each source has different limits; you handle exponential backoff, token buckets, retry with jitter.
- **Batch + streaming + CDC** — proves all three of Domain 1's ingestion patterns.
- **Cost-first design** — every service picked has an alternative that would have cost 3–10× more; you know why you didn't pick it.
- **Governance from day one** — Lake Formation column masking, Macie scan, VPC endpoints, no NAT gateway.
- **Real IaC** — 100% CDK, one command `cdk deploy`, one command `cdk destroy`.
- **LLM-in-pipeline** — quality classification, entity extraction, embeddings. Real Bedrock rate limits and retry logic.

### What LLM builders will recognize

The pattern maps 1:1 to internal tools at **Anthropic, OpenAI, Google DeepMind, Meta AI, Mistral, HuggingFace, LangChain, LlamaIndex, LiteLLM, Braintrust, Arize, W&B Weave, Perplexity, Cursor**. Every one of these companies runs (or needs) something of this shape internally.

---

## 2. The approach — signal-quality tiers

The novel design choice — and the thing that separates this from "another RAG demo" — is **explicit signal-quality tiering**.

Instead of "scrape everything about LLMs," we tag every document with a `signal_tier` at ingest time:

| Tier | Definition | Trust weight | Examples |
|---|---|---|---|
| **1 — Canonical** | Peer-reviewed papers, official vendor research | 1.0 | arXiv cs.CL papers, Anthropic Research posts, HuggingFace Daily Papers |
| **2 — Authoritative** | Official product docs, model cards | 0.8 | docs.claude.com, HuggingFace model cards, LMSYS leaderboard |
| **3 — Practitioner** | Real engineers discussing real problems, quality-filtered | 0.5 | Top HackerNews stories (score > 50), r/LocalLLaMA top posts, tracked GitHub issues |

**Explicit blocklist (never ingested):**
- Medium / Substack posts unless from a small allowlist of respected authors.
- Marketing blogs from AI vendors (except the `/research/` sub-paths).
- Generic tech news (VentureBeat, TechCrunch).
- LinkedIn posts.

**Every Bedrock enrichment call is gated by a quality classifier** that filters again — documents that "pass allowlist but fail quality" are moved to a `low_signal/` prefix and not embedded, saving cost.

This tiering is a **DE decision, not an ML decision.** It maps to:
- **Partition column** in Iceberg for query pruning.
- **Lake Formation LF-tag** for permission scoping (analysts see Tier 1 + 2; researchers see all).
- **Cost lever** — Tier 3 gets cheaper enrichment (classifier only, no embedding until upvoted).

---

## 3. Data sources

For each source: **API endpoint, rate limit, ingestion pattern, DE lesson, cost.**

### Source A: arXiv (Tier 1 — the canonical paper firehose)

**Endpoint:** `http://export.arxiv.org/api/query`
**Query pattern:** `?search_query=cat:cs.CL+OR+cat:cs.LG+OR+cat:cs.AI&sortBy=submittedDate&sortOrder=descending&max_results=100`
**Rate limit:** arXiv asks for 3-second delays between requests; hard limit ~4 req/sec but they ask you to be gentler. **We respect this — DE hygiene.**
**Data shape:** Atom XML with entries containing title, authors, abstract, category tags, PDF link, submission date.
**Ingest cadence:** Lambda every 6 hours (arXiv drops daily but we're generous).
**Volume:** ~50–200 new papers/day matching our filters.

**Ingestion pattern:**
```
EventBridge cron (every 6h) → Lambda
  ↓
Lambda queries arXiv API with `submittedDate >= last_watermark`
  ↓
For each result: write raw Atom XML to s3://.../raw/arxiv/dt=YYYY-MM-DD/hh=HH/
  ↓
Update watermark in DynamoDB (single-item table `ingest_watermarks`)
```

**DE lesson:** watermark-based incremental ingestion (Skill 1.1.11 replayability + Skill 1.2.9 volume/velocity/variety). We store the watermark in DynamoDB rather than S3 because we want atomic conditional updates on it.

**Cost:** ~$0 (arXiv is free, Lambda well within free tier).

---

### Source B: HuggingFace Daily Papers (Tier 1 — curated by community)

**Endpoint:** `https://huggingface.co/api/daily_papers`
**Rate limit:** With HF token: ~5000 req/hour. Without: 300/hour. **Use a token from Secrets Manager.**
**Data shape:** JSON array with paper metadata + arXiv IDs + upvote counts + curator's summary.
**Ingest cadence:** Lambda once daily at 16:00 UTC (after HF's daily paper drop).
**Volume:** ~5–15 papers/day.

**Ingestion pattern:**
```
EventBridge cron (16:00 UTC daily) → Lambda
  ↓
Fetch → write raw JSON to s3://.../raw/hf_daily/dt=YYYY-MM-DD/papers.json
```

**DE lesson:** deduplication downstream via arXiv ID (same paper often appears in both A and B). This becomes a **Glue join** in the silver layer — great interview talking point.

**Cost:** ~$0.

---

### Source C: Vendor research pages (Tier 1 — Anthropic, OpenAI, DeepMind)

**Endpoints:**
- `https://www.anthropic.com/sitemap.xml` → filter to `/research/*`
- `https://openai.com/sitemap.xml` → filter to `/index/*` (their research posts)
- `https://deepmind.google/sitemap.xml` → filter to `/research/*`

**Method:** sitemap → diff against `seen_urls` in DynamoDB → fetch HTML → extract main content with `trafilatura` (Python lib) or BeautifulSoup.
**Rate limit:** none stated, but **be a good citizen** — sequential requests, 2-second delay, User-Agent identifies you.
**Data shape:** HTML → cleaned Markdown-ish.
**Ingest cadence:** Weekly.
**Volume:** ~5–20 new posts/week across all three.

**Ingestion pattern:**
```
EventBridge cron (weekly, Monday 09:00 UTC) → Lambda
  ↓
For each vendor:
  fetch sitemap → filter URLs → diff against DynamoDB(seen_urls)
  ↓
  For each new URL:
    fetch HTML (respect robots.txt, 2s delay)
    extract main text with trafilatura
    write to s3://.../raw/vendor_research/{vendor}/dt=YYYY-MM-DD/{slug}.json
    insert URL into DynamoDB(seen_urls)
```

**DE lesson:** **idempotent crawling** — the DynamoDB `seen_urls` table makes re-runs safe. This maps to Domain 1 Skill 1.2.7 (troubleshoot common transformation failures) and Skill 1.1.11 (replayability).

**Cost:** ~$0.

---

### Source D: Vendor documentation (Tier 2 — best-practices layer)

**Endpoints (sitemap-based):**
- `https://docs.claude.com/sitemap.xml`
- `https://platform.openai.com/sitemap.xml`
- `https://ai.google.dev/gemini-api/sitemap.xml`
- Similar for Meta Llama, Mistral, DeepSeek, Cohere.

**Method:** same as Source C, but chunk by H2 headings for RAG-friendly retrieval.
**Volume:** ~500 pages initially, ~5–20 changed pages/week.

**Ingestion pattern:**
```
Initial one-shot backfill: run Lambda in an EventBridge one-off with a large timeout.
  ↓
Ongoing: weekly sitemap diff → fetch changed only.
```

**DE lesson:** chunking strategy is a **downstream transform decision**, not an ingest one. Store raw HTML; chunk in the Glue silver step. This separates concerns and lets you re-chunk without re-crawling.

**Cost:** ~$0.

---

### Source E: HuggingFace Hub API — model cards (Tier 2 — model releases)

**Endpoint:** `https://huggingface.co/api/models?sort=likes&limit=100&filter=text-generation`
**Rate limit:** ~5000/hour with token.
**Data shape:** JSON with model ID, downloads, likes, tags, license, description.
**Ingest cadence:** Daily.
**Volume:** ~100 models tracked, ~5–10 new/day, delta detection catches trending.

**Ingestion pattern:**
```
EventBridge cron (daily 12:00 UTC) → Lambda
  ↓
Fetch top-100 by likes + top-100 by downloads
  ↓
Write snapshot to s3://.../raw/hf_models/dt=YYYY-MM-DD/models.json
  ↓
Downstream Glue job computes daily deltas (which models jumped in downloads?)
```

**DE lesson:** **snapshot + delta** pattern — you keep every daily snapshot in raw, compute deltas in silver. Enables historical rewind ("what did the top-100 look like on 2026-06-15?"). Maps to Skill 2.3.1 (S3 load/unload) + Skill 2.4.2 (data change characteristics).

**Cost:** ~$0.

---

### Source F: HackerNews (Tier 3 — real-time practitioner firehose, quality-filtered)

**Endpoint:** Firebase realtime DB — `https://hacker-news.firebaseio.com/v0/`
- `/updates.json` — real-time subscription (SSE-like)
- `/item/{id}.json` — item details

**Rate limit:** **None declared** — but be gentle, ~100 req/sec is fine.
**Data shape:** JSON per item (story + comments).
**Ingest cadence:** Real-time via streaming.
**Volume:** ~10 events/sec across HN; filtered to LLM-adjacent = ~1 event/min.

**Ingestion pattern:**
```
Lambda subscribes to Firebase /updates.json (or polls every 30s if streaming setup is complex)
  ↓
For each new story/comment:
  fetch item details
  keyword-filter title/text for LLM terms (llama, claude, gpt, kimi, deepseek, mistral, etc.)
  score-filter (story score > 20 OR is comment on tracked story)
  ↓
Write to Kinesis Data Streams "hn_stream" (on-demand mode)
  ↓
Firehose subscribes → buffers 60s / 5MB → S3 raw
```

**DE lesson:** **the classic exam pattern** — Kinesis Data Streams for real-time replay + custom consumers, Firehose for managed S3 delivery. Explains why we use both instead of just Firehose (Skill 1.1.1, 1.1.10).

**Cost:** ~$0.10/weekend (Kinesis On-Demand pay-per-GB).

---

### Source G: Reddit r/LocalLLaMA + r/MachineLearning (Tier 3)

**Endpoint:** `https://oauth.reddit.com/r/LocalLLaMA/top?t=day&limit=100`
**Rate limit:** 100 req / 10min per OAuth app.
**Auth:** OAuth 2.0 client credentials, refresh every hour.
**Data shape:** JSON per post.
**Ingest cadence:** Hourly.
**Volume:** ~200 top posts / subreddit / day.

**Ingestion pattern:**
```
Reddit OAuth credentials stored in Secrets Manager (rotation Lambda)
  ↓
EventBridge cron (hourly) → Lambda
  ↓
Get access token (refresh if expired)
  ↓
Fetch top posts for each subreddit
  ↓
Score-filter (score > threshold), dedupe against DynamoDB
  ↓
Write to s3://.../raw/reddit/dt=YYYY-MM-DD/{subreddit}/hh=HH/
```

**DE lesson:** **Secrets Manager + rotation** (Skill 4.1.3, 4.2.2). OAuth token rotation is a real interview probe: how do you handle expiry, refresh, and thundering herd on token refresh? Answer: cache in memory across Lambda warm invocations, refresh on 401.

**Cost:** ~$0.

---

### Source H: RDS (metadata — CDC path to prove Domain 1)

We ingest something from RDS via DMS purely to prove the CDC pattern. Content: **tracked models registry** — a curated table of models being tracked, with metadata (vendor, family, first-seen date, hero benchmarks). Analysts add entries via a simple form; changes replicate to the lake.

**Method:** RDS Postgres → DMS full-load + CDC → Kinesis → Firehose → S3 raw.
**Volume:** trivial (~100 rows, ~5 changes/week).

**Ingestion pattern:**
```
RDS t4g.micro in private subnet with `tracked_models` table
  ↓
DMS replication instance (t3.micro, deleted between sessions)
  ↓
DMS task: full-load + CDC → Kinesis
  ↓
Firehose → S3 raw
```

**DE lesson:** proves you know DMS (Skill 1.1.2, 2.4.3). **DMS instance is deleted at the end of each session** to save $15/mo idle cost — a real cost-engineering habit.

**Cost:** ~$0.30/weekend when active; $0 idle.

---

## 4. Architecture

```
┌────────────────────────────────────────────────────────────────────────┐
│                            INGEST TIER                                  │
│                                                                         │
│  A. arXiv API           ▶ Lambda (6h)   ▶ S3 raw/arxiv/                │
│  B. HF Daily Papers     ▶ Lambda (daily)▶ S3 raw/hf_daily/             │
│  C. Vendor research     ▶ Lambda (wk)   ▶ S3 raw/vendor_research/      │
│  D. Vendor docs         ▶ Lambda (wk)   ▶ S3 raw/vendor_docs/          │
│  E. HF Hub models       ▶ Lambda (daily)▶ S3 raw/hf_models/            │
│  F. HackerNews          ▶ Lambda (RT)   ▶ Kinesis ▶ Firehose ▶ S3      │
│  G. Reddit              ▶ Lambda (hr)   ▶ S3 raw/reddit/               │
│  H. RDS metadata        ▶ DMS ▶ Kinesis ▶ Firehose ▶ S3 raw/repos/    │
│                                                                         │
│  All Lambdas use exponential-backoff retry (Skill 1.1.9)               │
│  Watermarks in DynamoDB "ingest_state" table                           │
└────────────────────────────────┬───────────────────────────────────────┘
                                 ▼
┌────────────────────────────────────────────────────────────────────────┐
│                           TRANSFORM TIER                                │
│                                                                         │
│  Glue Job 1 (bronze→silver flat):                                       │
│    - Reads all raw sources                                              │
│    - Parses formats (Atom XML, JSON, HTML→text via trafilatura)        │
│    - Attaches signal_tier column                                        │
│    - Writes Iceberg via S3 Tables to silver/documents/                 │
│                                                                         │
│  Glue Job 2 (Bedrock enrichment):                                      │
│    - Step 1: Nova Lite quality classifier (few-shot)                   │
│              PASS → continue; FAIL → move to low_signal/               │
│    - Step 2: Nova Lite structured extraction:                          │
│              {category, models_mentioned, techniques, best_practice}    │
│    - Step 3: Titan Embeddings v2 (1024-dim) — ONLY for PASS docs       │
│    - Writes to silver/documents_enriched/ (Iceberg)                    │
│    - Batches 20 docs per Bedrock call (cost + throughput win)          │
│                                                                         │
│  Glue Data Quality DQDL rules at exit of job 2:                        │
│    - IsComplete "doc_id", "signal_tier"                                │
│    - ColumnValues "signal_tier" in ["1","2","3"]                       │
│    - Uniqueness "doc_id" > 0.99                                        │
│                                                                         │
└────────────────────────────────┬───────────────────────────────────────┘
                                 ▼
┌────────────────────────────────────────────────────────────────────────┐
│                             STORE TIER                                  │
│                                                                         │
│  S3 Iceberg tables via S3 Tables:                                       │
│    bronze/ = raw as-landed                                              │
│    silver/documents (flat + enriched)                                   │
│    silver/models_daily (HF Hub snapshots + deltas)                     │
│    gold/topics_weekly (aggregations for dashboards)                    │
│                                                                         │
│  RDS PostgreSQL + pgvector:                                            │
│    document_vectors(doc_id PK, signal_tier, embedding vector(1024))    │
│    HNSW index (m=16, ef_construction=64)                               │
│                                                                         │
│  Redshift Serverless (base storage free):                              │
│    External schema over silver Iceberg tables                          │
│    Analyst queries + weekly reports                                    │
│                                                                         │
│  DynamoDB:                                                              │
│    ingest_state (watermarks, single-item)                              │
│    seen_urls (dedup)                                                    │
│    hn_hot (last 24h of HN items, TTL 86400)                            │
│                                                                         │
└────────────────────────────────┬───────────────────────────────────────┘
                                 ▼
┌────────────────────────────────────────────────────────────────────────┐
│                              SERVE TIER                                 │
│                                                                         │
│  API Gateway → Lambda (semantic_search):                                │
│    POST /search {"query": "...", "tier_filter": [1,2]}                 │
│    → embed query with Bedrock Titan                                    │
│    → pgvector KNN with tier filter                                     │
│    → return top-K with LLM-generated one-line explanation              │
│                                                                         │
│  Streamlit on Fargate (or local machine to save $):                    │
│    - Trending techniques by week                                        │
│    - Model comparison view                                              │
│    - Semantic search UI                                                 │
│    - Data source health monitor                                        │
│                                                                         │
│  Athena workgroup for ad-hoc analyst SQL                               │
│  Optional: Bedrock Agent for NL Q&A over the lake                      │
└────────────────────────────────────────────────────────────────────────┘

──────  ORCHESTRATION  ──────  Step Functions ← EventBridge Scheduler
──────  OBSERVABILITY  ──────  CloudWatch dashboards, alarms → SNS email
──────  GOVERNANCE     ──────  Lake Formation LF-tags (pii, tier),
                              Macie one-time scan, KMS CMKs per service
──────  DEPLOY         ──────  AWS CDK (Python) → CloudFormation
                              CodePipeline for CI/CD (optional weekend 4)
──────  NETWORK        ──────  VPC with endpoints (S3, Kinesis, Bedrock,
                              Secrets Manager, DynamoDB). NO NAT gateway.
```

---

## 5. Design decisions & interview defensibility

The core of interview prep: **every service choice below has a rationale + rejected alternatives + what would change at scale**.

### 5.1 Kinesis Data Streams + Firehose (both) vs Firehose-only

**Choice:** Both. HN Lambda writes to KDS; Firehose subscribes and delivers to S3.

**Alternatives rejected:**
- **Firehose-only** (Lambda writes direct to Firehose): can't replay, no other consumers. Fine for MVP but fails Skill 1.1.11.
- **KDS-only** (custom Lambda consumer writes S3): more code, more failure surface. Firehose is a solved problem.

**Why both:** KDS gives us the replay + fanout guarantees the exam explicitly tests (Skill 1.1.10). Firehose handles the managed delivery to S3 with format conversion.

**At 10× scale:** KDS on-demand → provisioned with N shards. Add enhanced fan-out for multiple consumers.

**Interview probe you'll get:** *"Why not MSK?"* — you don't have Kafka expertise, don't need Connect, don't need Kafka's exactly-once semantics. Kinesis is simpler + cheaper for this volume.

### 5.2 RDS Postgres + pgvector vs Aurora Serverless v2 vs OpenSearch Serverless

**Choice:** RDS Postgres db.t4g.micro with pgvector extension.

**Alternatives rejected:**
- **Aurora Serverless v2**: 0.5 ACU minimum × $0.12/hr = **$43/mo idle**. Kills the cost budget.
- **OpenSearch Serverless**: 2 OCU minimum × $0.24/hr = **$350/mo idle**. Completely unfit for a demo budget.
- **Bedrock Knowledge Bases with OpenSearch Serverless backend**: same problem — $350/mo idle.
- **Amazon MemoryDB**: durable Redis but limited vector features + expensive.

**Why RDS:** free tier for 12 months. After that, ~$12/mo. Stop the instance between sessions → $1/mo idle storage. pgvector HNSW is proven at 1M+ vectors, which is way more than we'll have.

**At 10× scale:** Move to Aurora Postgres for read replicas + auto-scaling. At 100× or when latency matters: OpenSearch Serverless or dedicated vector DB.

**Interview probe:** *"What's the HNSW `m` and `ef_construction`?"* — defaults (16 / 64) are fine for ≤ 1M vectors. Higher `m` = higher recall but slower build + more memory.

### 5.3 S3 Tables (managed Iceberg) vs self-managed Iceberg on S3

**Choice:** S3 Tables.

**Alternatives rejected:**
- **Self-managed Iceberg**: you own compaction, snapshot expiration, orphan file cleanup. Extra 20% engineering effort.
- **Delta Lake on S3 via EMR**: locks you into EMR compute. Iceberg is engine-agnostic.
- **Plain Parquet + Hive-style partitions**: no ACID, no schema evolution, no time travel. Non-starter.

**Why S3 Tables:** AWS handles the maintenance jobs that break most Iceberg self-managed deployments. Costs a small premium but saves engineering hours.

**At 10× scale:** stays the same — S3 Tables scales to petabytes.

**Interview probe:** *"What does `OPTIMIZE` do in Iceberg?"* — rewrites small files into larger ones (target 128MB–1GB). Fixes the small-files problem that kills query performance.

### 5.4 Bedrock Nova Lite (not Claude) for enrichment

**Choice:** Nova Lite for classification + extraction. Titan for embeddings. Claude reserved for the search-time explanation (if we splurge).

**Alternatives rejected:**
- **Claude Sonnet 4.5**: $3/M input, $15/M output. 50× more expensive than Nova Lite. Not needed for structured extraction.
- **Claude Opus**: even worse. Waste on classification.
- **Self-hosted Llama**: infrastructure headache, cold-start cost, no free win at this scale.

**Why Nova Lite:** $0.06/M input, $0.24/M output. On classification with 5-shot prompt + JSON output schema, accuracy is > 92% (measure this yourself with an eval set).

**At 10× scale:** stay Nova Lite; use Provisioned Throughput if throttling.

**Interview probe:** *"How do you handle Bedrock throttling in a Glue job?"* — batch 20 docs/call, exponential backoff with jitter, respect the retry-after header. Move to Provisioned Throughput at production scale.

### 5.5 VPC endpoints, no NAT Gateway

**Choice:** Interface endpoints for Bedrock / Secrets Manager / Kinesis / DynamoDB. Gateway endpoints for S3 / DynamoDB. No NAT.

**Alternatives rejected:**
- **NAT Gateway**: $32/mo per AZ, $0.045/GB data processing. Kills the budget.
- **NAT Instance**: cheaper but you manage the EC2. Not worth the ops.
- **All-public subnets**: security no-go.

**Why endpoints:** ~$7/mo per endpoint × 4 = $28/mo — but only when in use. Delete between weekends if truly budget-constrained. Traffic stays on the AWS backbone.

**At 10× scale:** endpoints scale linearly. NAT Gateway becomes worth it when you make many cross-service calls not covered by endpoints.

**Interview probe:** *"Gateway vs interface endpoint?"* — Gateway: S3, DynamoDB only, free, route-table entry. Interface: PrivateLink ENI in your subnet, ~$7/mo + per-GB, covers most services.

### 5.6 Lake Formation LF-tags vs per-table permissions

**Choice:** LF-tags.

**Alternatives rejected:**
- **Per-table grants:** every new table = new grants for every role. Doesn't scale beyond ~20 tables.
- **IAM-only** (no Lake Formation): can't do column or row-level access on S3 data.

**Why LF-tags:** define the policy once ("tier=3 requires researcher role"), apply the tag many times. New tables inherit tag defaults.

**At 10× scale:** LF-tags remain the pattern. Add ABAC via IAM Identity Center at the auth layer.

### 5.7 Signal-tier as partition column

**Choice:** Yes, `signal_tier` is a partition column in the silver Iceberg table.

**Rationale:** analysts + dashboards almost always filter by tier. Partitioning here gives us:
- Query pruning (Skill 2.4.5).
- LF-tag alignment (grant researcher role to `signal_tier=3`).
- Cost visibility per tier (Athena bytes scanned reports by partition).

**Trade-off:** 3 partitions is low cardinality — normally we'd avoid, but the enforcement + query patterns win.

### 5.8 Signal tiering as ingest-time metadata, not derived

**Choice:** tier is written into the raw JSON at ingestion time by the ingest Lambda (from a hardcoded source registry).

**Alternative rejected:** derive tier in the silver Glue job. Cleaner at first — but adds latency and prevents downstream systems from tier-filtering at read time.

---

## 6. Cost cap strategy

### Hard commitments

- **Total build spend: $30 across 4 weekends.**
- **Idle: $5/month max after project done.**
- **Any 24 hours: $8 max.**

### Setup (do this before Weekend 1 — 10 minutes)

1. **AWS Budgets → Create budget:**
   - Type: Cost budget.
   - Name: `LLMIntelPlatform`.
   - Amount: `$25/month`.
   - Alerts: 50%, 80%, 100% of budgeted (email + SNS).
2. **Cost Anomaly Detection:** enable, set threshold `$5` daily surprise.
3. **Enable Cost Explorer** (free) and tag every CDK stack with `Project=LLMIntel`.

### Per-service targets

| Service | Weekend 1 | Weekend 2 | Weekend 3 | Weekend 4 | Idle/mo |
|---|---|---|---|---|---|
| S3 storage | $0.10 | $0.20 | $0.30 | $0.40 | $0.50 |
| Bedrock (Nova Lite + Titan) | — | $6 | $2 | $0.50 | $0 |
| Kinesis (On-Demand) | $0.20 | $0.10 | $0.10 | $0 | $0 |
| Firehose | $0.05 | $0.05 | $0 | $0 | $0 |
| Glue Jobs | $1 | $3 | $1 | $0.50 | $0 |
| Glue Crawler | $0.20 | $0.10 | $0.10 | $0 | $0 |
| RDS db.t4g.micro (free tier) | $0.30 | $0.50 | $0.50 | $0.50 | $0 (free) / $2 |
| Redshift Serverless | $0 | $1.50 | $2 | $0.50 | $0 |
| DynamoDB (On-Demand) | $0 | $0 | $0 | $0 | $0 |
| Lambda + API GW | $0 | $0 | $0.10 | $0 | $0 |
| CloudWatch | $0.30 | $0.30 | $0.30 | $0.30 | $0.50 |
| KMS (2 CMKs) | $0.50 | $0.50 | $0.50 | $0.50 | $2 |
| VPC endpoints | $0.50 | $0.50 | $0.50 | $0.50 | delete when idle |
| Macie | — | — | — | $2 | $0 (disable after) |
| DMS (deleted between sessions) | $0.30 | $0 | $0 | $0 | $0 |
| **Weekend total** | **$3.5** | **$12.75** | **$7.5** | **$5.20** | **$3** |
| **Cumulative** | **$3.5** | **$16.25** | **$23.75** | **$28.95** | |

**We're right at $29 cumulative — inside the $30 hard cap.**

### Contingency knobs (if you overrun)

1. **Move Bedrock to cheaper batch pattern** — enrich 5K docs instead of 10K, halve cost.
2. **Skip Weekend 4's Macie scan** — write a manual note "would enable Macie in production" — saves $2.
3. **Skip Redshift** — Athena covers analytics, saves $4 across weekends.
4. **Kill VPC endpoints between weekends** — saves $6 idle, requires redeploy each weekend.

### Between-weekend routine (10-second habit)

At end of each session, run:
```bash
# From your project directory
cd cdk
python scripts/pause.py    # Stops RDS, deletes DMS instance
```

The `pause.py` script we'll write does:
- `rds stop-db-instance --db-instance-identifier llm-intel-rds`
- `dms delete-replication-instance --replication-instance-arn ...`
- `aws ce get-cost-and-usage --time-period Start=$(date -u +%Y-%m-01),End=$(date -u +%Y-%m-%d)` → print running total.

This should take 30 seconds and become muscle memory.

### The "I forgot" safety net

At the end of the project, running `cdk destroy --all` cleanly tears everything down. Total sunk cost to that point is whatever the budget shows. No lingering `$50/mo` surprise the next month.

---

## 7. Weekend 1 — Domain 1 (Ingestion + Transformation)

### Study connection

You studied Kinesis / MSK / Firehose / DMS / Glue / Step Functions / EventBridge / Lambda / CDK / Bedrock. Now you build them.

### Objectives

By end of Sunday evening:
- Three of the six ingest sources landing in S3 (arXiv batch, HuggingFace Daily Papers batch, HackerNews streaming).
- Basic Glue transform silver-layer table.
- Step Functions orchestration.
- **Cost so far: ~$3.50.**

### Prep (Friday evening, 30 min)

Get these ready before Saturday morning so you don't context-switch:

1. **AWS account** with root MFA + IAM admin user.
2. **Budget alarm** at $25/mo (see §6).
3. **Personal access tokens** stored ready to copy into Secrets Manager after deploy:
   - GitHub PAT (fine-grained, read-only, expires in 60 days). *Even though we don't ingest GitHub Archive anymore, we might for GitHub issues in Weekend 3.*
   - HuggingFace read token.
   - Reddit OAuth: create app at `https://www.reddit.com/prefs/apps`, note client_id + client_secret.
4. **Local tooling:** `aws-cli`, `cdk`, `python 3.11+`, `docker` (for Glue local testing later).
5. **Region choice:** `us-east-1` (best Bedrock model availability, cheapest bandwidth).

### Saturday morning (3 hours) — Foundation

**Step 1 — CDK bootstrap (30 min)**

```bash
mkdir llm-intel && cd llm-intel
cdk init app --language=python
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cdk bootstrap
```

**What you're learning:** CDK synthesizes to CloudFormation. `cdk bootstrap` creates a stack in your account for CDK assets (S3 bucket for large templates, ECR for Lambda container images).

**Interview probe:** *"What's the difference between CDK and CloudFormation?"* — CDK is a Python/TS library that generates CloudFormation JSON. You get type checking, IDE completion, reusable constructs. CloudFormation is the underlying declarative engine.

**Step 2 — Base stack: VPC + KMS + logs (1 hour)**

Create `stacks/base_stack.py`:
- VPC with 2 AZs, private + public subnets, **NO NAT gateway** (this is deliberate).
- VPC gateway endpoint for S3.
- Interface endpoints for: Secrets Manager, KMS, DynamoDB (created lazily as we need them).
- 2 KMS Customer Managed Keys: `key-s3` and `key-data`. Set key policy allowing the account root + specific service roles.
- CloudWatch log group `/llm-intel` with 30-day retention.

**What you're learning:**
- Why no NAT: it's $32/mo. Endpoints let Lambdas in the VPC reach AWS services privately.
- KMS envelope encryption: our S3 CMK will be referenced by every S3 bucket. Every write invokes GenerateDataKey.
- Log retention: without setting it, logs are `Never expire` by default — surprise cost.

**Interview probe:** *"Why 2 KMS keys and not 1?"* — separation of concerns + blast radius. If we ever rotate or delete the data key, S3 is unaffected. Different key policies for different service principals.

**Step 3 — S3 buckets (30 min)**

Create `stacks/storage_stack.py`:
- `s3-raw`: raw landing zone. SSE-KMS with `key-s3`. Versioning on. Lifecycle: → IA at 30d, → Glacier at 90d.
- `s3-silver`: transformed data (Iceberg tables live here).
- `s3-gold`: aggregated tables for BI.
- `s3-logs`: access logs + CloudTrail sink.
- Block public access on all.

**Interview probe:** *"Why versioning on raw?"* — protection against accidental Lambda overwrite (idempotent re-runs). Version-based restore is your undo.

**Step 4 — DynamoDB state tables (30 min)**

- `ingest_watermarks` — single-item table for each source's watermark.
- `seen_urls` — dedup for crawlers, TTL 90 days.
- On-demand billing mode.

**Cost checkpoint:** at end of morning, you've deployed maybe $0.10 of infrastructure. Alarm hasn't triggered.

### Saturday afternoon (3 hours) — Ingestion #1: arXiv + HuggingFace batch

**Step 5 — arXiv Lambda (1 hour)**

Create `lambdas/ingest_arxiv/handler.py`:
- Query arXiv API with `submittedDate` watermark from DynamoDB.
- 3-second sleep between paginated calls (arXiv politeness).
- Write raw Atom XML gzipped to S3.
- Update watermark on success.
- On failure: don't update watermark (safe retry).

**Deploy Lambda + EventBridge cron (every 6h). Invoke manually once to backfill.**

**Interview probe:** *"What if the arXiv API is down during our scheduled run?"* — Lambda fails, EventBridge invokes it again on next schedule; watermark unchanged so no data loss. If down for hours, exponential backoff would keep hammering — we could add a circuit breaker via DynamoDB flag.

**Step 6 — HuggingFace Papers Lambda (30 min)**

Similar pattern, simpler (just fetch + write, no watermark math because we get all daily papers).

**Step 7 — Validate in Athena (30 min)**

- Glue Crawler over `s3://.../raw/arxiv/` and `s3://.../raw/hf_daily/`.
- Athena workgroup pointing to a results bucket + KMS-encrypted.
- Query: `SELECT COUNT(*) FROM raw_arxiv_papers WHERE date_key >= current_date - 7`.

**Interview probe:** *"Glue crawler vs Athena partition projection?"* — crawler runs on a schedule + costs $0.44 per DPU-hour + latency. Partition projection is defined at table creation, computed at query time, free, zero latency. We'll switch to projection in Weekend 3.

**Step 8 — Glue Job v1: raw → silver flat (1 hour)**

- PySpark job that reads raw arXiv XML + HF JSON, unifies into `silver.documents` with schema:
  - `doc_id`, `source`, `signal_tier`, `title`, `authors`, `text`, `published_at`, `url`, `raw_ref`.
- Writes as Iceberg via S3 Tables.
- Uses Glue bookmarks for incremental processing.

**Interview probe:** *"Why Iceberg vs plain Parquet?"* — ACID, schema evolution, time travel, hidden partitioning. Iceberg lets Glue append while Athena queries — no locking issues.

### Sunday (2.5 hours) — Ingestion #2: HackerNews streaming + orchestration

**Step 9 — Kinesis Data Streams (on-demand) (15 min)**

Just one stream: `hn_stream`. On-demand mode (pay per GB).

**Step 10 — HackerNews polling Lambda + Kinesis → Firehose → S3 (1.5 hour)**

- Lambda polls HN Firebase every 30 seconds.
- Fetches new items since last-max-id.
- Keyword-filter title/text for LLM terms (list of ~30 terms — llama, claude, gpt, kimi, deepseek, mistral, gemini, etc.).
- Score threshold: story score > 20 OR comment on a tracked story.
- Publishes filtered items to KDS.
- Firehose: subscribes to KDS, buffers 60s / 5MB, writes to `s3://raw/hn/dt=YYYY-MM-DD/`.

**Interview probe:** *"Why KDS + Firehose and not Lambda direct to S3?"* — replay ability (Skill 1.1.11). If Firehose is misconfigured or we later want a real-time consumer for alerting, we plug into KDS.

**Step 11 — Step Functions orchestration (45 min)**

State machine `nightly-pipeline`:
- 1. Wait for 03:00 UTC (via EventBridge Scheduler).
- 2. Run Glue Crawler in parallel: arXiv, HF, HN, Vendor Research (last one is nothing yet, placeholder).
- 3. Run Glue Job v1 (raw → silver).
- 4. On success: publish to SNS `pipeline-notifications`.
- 5. On failure: retry 3× with exponential backoff, then fail to SNS.

**Interview probe:** *"Step Functions Standard vs Express?"* — Standard: exactly-once, 1-year duration, per-state-transition pricing. Express: at-least-once, 5-min max, high-volume. We use Standard for daily ETL; nobody's paying $2 per state transition for scheduled jobs.

**Step 12 — Weekend 1 test + cost check (30 min)**

- Run the state machine manually. All steps succeed?
- Athena: `SELECT source, COUNT(*) FROM silver.documents GROUP BY source`. Should show `arxiv`, `hf_daily`, `hn`.
- Cost Explorer: what did I spend? Should be $3–5.
- **Run the `pause.py` script:** `python scripts/pause.py` — stops RDS (which we haven't created yet), deletes DMS (also not yet), prints cost total.

### Interview drills — Weekend 1

Practice defending these:

1. **Explain the ingest architecture at 100k events/second.** Answer: current design breaks — KDS on-demand caps at ~1000 records/sec per shard × 128 shards default. Move to provisioned mode + enhanced fan-out. Add multiple Firehose delivery streams for parallel S3 writes. Consider MSK if you need >GB/sec throughput.
2. **How do you handle at-least-once vs exactly-once from HN → S3?** KDS-to-Firehose is at-least-once. Dedup happens downstream in Glue on `hn_id`. We accept that duplicates land in S3 raw; the silver layer is idempotent.
3. **What if arXiv changes their API?** Ingest is decoupled from downstream — raw XML lives in S3 forever. We can re-parse in Glue when the format changes without re-ingesting.
4. **What's your DR story?** S3 versioning + cross-region replication (not implemented in MVP but described in extensions). Watermarks in DynamoDB with PITR = 35 days. Rerun state machine from any historical date via input parameter.

---

## 8. Weekend 2 — Domain 2 (Data Store Management)

### Study connection

Redshift / DynamoDB / Iceberg / S3 Tables / Glue Catalog / HNSW/IVF / pgvector.

### Objectives

- Bedrock enrichment pipeline (quality classifier → structured extraction → embeddings).
- RDS pgvector with HNSW index working end-to-end.
- Redshift Serverless external schema over silver Iceberg.
- Semantic search working from a Jupyter notebook or CLI.
- **Cost so far: ~$16.**

### Saturday (3 hours)

**Step 1 — Bedrock setup (30 min):** request model access for Nova Lite + Titan Embeddings v2. Wait for approval (usually instant).

**Step 2 — Glue Job v2 with Bedrock enrichment (2 hours):**

Three-stage pipeline in one Glue job:

```python
# Pseudocode
docs_df = spark.read.format("iceberg").load("silver.documents")
new_docs = docs_df.filter(col("enriched_at").isNull()).limit(1000)  # batch cap

# Stage 1: quality classifier (Nova Lite)
# Batch 20 docs → single Bedrock InvokeModel call → parse JSON response
classified = new_docs.mapPartitions(quality_classify_udf)
passed = classified.filter(col("quality_pass") == True)
failed = classified.filter(col("quality_pass") == False)

# Failed docs → low_signal/, no more enrichment
failed.write.format("iceberg").mode("append").save("silver.low_signal")

# Stage 2: structured extraction (Nova Lite)
# {category, models_mentioned, techniques, best_practice_summary}
extracted = passed.mapPartitions(extract_structured_udf)

# Stage 3: embeddings (Titan)
# Batch 25 texts per call
embedded = extracted.mapPartitions(embed_udf)

# Write to silver enriched + vector table
embedded.select("doc_id", "category", "models_mentioned", ...).write.format("iceberg").mode("append").save("silver.documents_enriched")
```

**Interview probes on Bedrock:**
- *"How do you handle Bedrock rate limits inside a Glue job?"* — batch 20 docs/call, exponential backoff with jitter on ThrottlingException, respect the `Retry-After` response header. If persistent, switch to Provisioned Throughput.
- *"Why batch inside a Spark partition, not row-by-row?"* — row-by-row = 1 Bedrock call per row × N rows. Batching = N/20 calls. 20× cost reduction + 20× throughput.
- *"What's the JSON output schema and how do you enforce it?"* — few-shot prompt with 3 example inputs + expected JSON outputs. Use Bedrock's `response_format` if supported for the model, otherwise parse + validate with Pydantic + retry on parse failure.

**Step 3 — RDS Postgres + pgvector (30 min):**

- `RDS db.t4g.micro`, private subnet, encrypted with `key-data`.
- User-data script: `CREATE EXTENSION vector;`.
- Master password in Secrets Manager (no rotation yet — Weekend 4).
- Skill 2.1.3 quote: *"using indexing algorithms like Hierarchical Navigable Small Worlds (HNSW) with Amazon Aurora PostgreSQL"* — we're using this exact pattern on RDS instead of Aurora for cost.

### Sunday (3 hours)

**Step 4 — pgvector schema + HNSW index (30 min):**

```sql
CREATE TABLE document_vectors (
    doc_id text PRIMARY KEY,
    signal_tier smallint NOT NULL,
    published_at timestamptz,
    embedding vector(1024) NOT NULL
);

CREATE INDEX ON document_vectors USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- HNSW parameters:
-- m: max connections per node in the graph (16 = good default)
-- ef_construction: candidate list size during build (64 = higher recall, slower build)
-- ef_search: candidate list size at query time (SET LOCAL ef_search = 100)
```

**Step 5 — Glue → RDS sync step (1 hour):**

Use RDS Data API (avoids VPC-in-Lambda networking headache):
- Glue Job step: batch-insert embeddings using `INSERT ... ON CONFLICT (doc_id) DO UPDATE`.
- 1000 rows per batch statement.

**Step 6 — Semantic search test (30 min):**

From a local Python script (or Athena, or Redshift):
```python
query = "structured extraction with JSON schema"
q_emb = bedrock.invoke(model_id="titan-embed", body={"inputText": query})
results = rds_data.execute_statement(sql=f"""
    SELECT doc_id, 1 - (embedding <=> '{q_emb}') AS similarity
    FROM document_vectors
    WHERE signal_tier IN (1, 2)
    ORDER BY embedding <=> '{q_emb}'
    LIMIT 10
""")
```

Should return 10 semantically-relevant docs, ranked. **This is the money moment.**

**Step 7 — Redshift Serverless workgroup + external schema (45 min):**

- Redshift Serverless workgroup, 8 base RPU, encrypted.
- `CREATE EXTERNAL SCHEMA silver FROM DATA CATALOG DATABASE 'llm_intel_silver' IAM_ROLE '<role>' CREATE EXTERNAL DATABASE IF NOT EXISTS;`
- Query: `SELECT source, category, COUNT(*) FROM silver.documents_enriched WHERE published_at >= current_date - 30 GROUP BY 1, 2`.

**Interview probes on stores:**
- *"When would you move from RDS pgvector to a dedicated vector DB?"* — > 10M vectors, sub-100ms p99 required, need hybrid sparse+dense, or need cross-region replication. Then OpenSearch Serverless or Pinecone.
- *"Redshift Serverless vs provisioned RA3?"* — Serverless: pay per RPU-second, 0 idle cost, elastic. RA3: fixed cost, predictable, cheaper at sustained load. For a demo, Serverless every time.
- *"External schema vs COPY into Redshift?"* — external = query Iceberg in-place from S3. COPY = load into Redshift storage. External wins for infrequent scans of huge data; COPY wins for repeated scans of the same slice.

**Step 8 — S3 Lifecycle policies + Weekend 2 test (30 min):**

- Raw → Standard-IA at 30d → Glacier IR at 90d.
- Silver → keep in Standard (queried frequently).
- Run everything end-to-end. Cost check. Pause script.

### Interview drills — Weekend 2

1. **Explain HNSW.** Graph-based ANN. Each node has ~M neighbors; layered structure lets you skip to distant neighborhoods first, refine at lower layers. Expected O(log n × d) queries at high recall.
2. **What's `ef_search` for and how do you tune it?** Query-time candidate list. Higher `ef_search` = higher recall, higher latency. Tune per query type — 40 for tight recall, 100+ for near-exhaustive.
3. **How do you handle a new embedding model with different dimensions?** Iceberg schema evolution: add `embedding_v2 vector(2048)` column. Backfill old rows. Cutover queries. Drop old column after grace period. Time travel gives us the safety net.
4. **Iceberg time travel — how does it work under the hood?** Every write creates a new snapshot in the metadata log. Snapshots reference specific Parquet data files. `SELECT ... FOR VERSION AS OF N` reads the file list from snapshot N.

---

## 9. Weekend 3 — Domain 3 (Operations + Analysis)

### Objectives

- Athena partition projection (kill the crawler).
- Glue Data Quality DQDL rules.
- Semantic search REST API.
- Streamlit dashboard.
- CloudWatch monitoring + alarms.
- **Cost so far: ~$24.**

### Saturday (3 hours)

**Step 1 — Athena partition projection (45 min):**
Replace crawlers with projection for the time-partitioned raw tables.

```sql
CREATE EXTERNAL TABLE raw.arxiv_papers (
  ...
) PARTITIONED BY (dt string)
LOCATION 's3://.../raw/arxiv/'
TBLPROPERTIES (
  'projection.enabled' = 'true',
  'projection.dt.type' = 'date',
  'projection.dt.range' = '2020-01-01,NOW',
  'projection.dt.format' = 'yyyy-MM-dd',
  'storage.location.template' = 's3://.../raw/arxiv/dt=${dt}/'
);
```

Then delete the Glue crawler for arXiv. Cost saved: ~$0.20/week.

**Step 2 — Glue Data Quality DQDL (45 min):**

Attach ruleset to Glue Job v2:
```
Rules = [
  IsComplete "doc_id",
  IsComplete "signal_tier",
  Uniqueness "doc_id" > 0.99,
  ColumnValues "signal_tier" in [1,2,3],
  ColumnLength "text" between 100 and 100000,
  RowCount > 10
]
```

Job fails on rule violation — the data contract behavior.

**Step 3 — Search API (1.5 hours):**

- API Gateway `/search` POST endpoint.
- Lambda handler:
  ```python
  def handler(event, context):
      body = json.loads(event["body"])
      q_emb = bedrock.invoke_titan_embed(body["query"])
      tiers = body.get("tier_filter", [1, 2, 3])
      results = rds_data.execute(f"""
          SELECT doc_id, title, url, signal_tier,
                 1 - (embedding <=> '{q_emb}') AS similarity
          FROM document_vectors dv
          JOIN documents_enriched de ON dv.doc_id = de.doc_id
          WHERE signal_tier = ANY({tiers})
          ORDER BY embedding <=> '{q_emb}'
          LIMIT 10
      """)
      # Optional: Nova Lite one-line explanation of why each result matches
      return {"statusCode": 200, "body": json.dumps(results)}
  ```
- IAM auth or API key (not open to public).

### Sunday (3 hours)

**Step 4 — Streamlit dashboard on ECS Fargate or local (1.5 hour):**

Streamlit app with pages:
- **Trending techniques** — Athena query over silver_enriched for `techniques` extraction, 30-day window.
- **Model comparison** — filter by two models, show docs that mention both.
- **Semantic search** — hits your API from Step 3.
- **Source health** — DynamoDB `ingest_watermarks` + last row count per source.

Deploy option A: **Fargate Spot** with 0.25 vCPU + 512MB = $2/mo. Option B: **run locally**, connect to AWS via SigV4 — $0.

**Step 5 — CloudWatch (1 hour):**
- Dashboard with widgets: Glue job duration, Bedrock invocation count, Kinesis IteratorAge, API p99 latency, RDS CPU.
- Alarms:
  - Glue job failure count > 0 in 15 min.
  - Bedrock throttling errors > 10 in 5 min.
  - Kinesis IteratorAge > 5 min.
  - API 5xx rate > 1%.
- SNS topic → your email.

**Step 6 — CloudTrail management events + Logs Insights saved queries (30 min):**
- CloudTrail: management events (free).
- Logs Insights saved queries:
  - Errors per Lambda per hour.
  - Top slowest Glue job runs.
  - Auth failures on API Gateway.

### Interview drills — Weekend 3

1. **DQ rule violation — what happens next?** DQDL fails the Glue job. Step Functions catches the failure, retries once, then sends SNS alert. Analysts see a stale gold table until we fix upstream.
2. **API latency budget?** p50: 200ms (Titan embed 100ms + pgvector KNN 50ms + Nova Lite explanation 300ms). Improvement: cache embeddings for hot queries in DynamoDB.
3. **How would you scale the search API to 1000 rps?** Provisioned concurrency on Lambda. Read replica on RDS. Cache embeddings + top-K results by query hash in DynamoDB with TTL. Prewarm HNSW `ef_search` tuning.

---

## 10. Weekend 4 — Domain 4 (Security + Governance)

### Objectives

- IAM least-privilege audit.
- KMS per-service CMKs with correct policies.
- Lake Formation with LF-tags.
- Macie one-time PII scan.
- VPC endpoints (finalize, kill any residual NAT paths).
- Secrets Manager rotation on RDS master password.
- CI/CD pipeline (optional if time).
- **Cost so far: ~$29 — inside the cap.**

### Saturday (3 hours)

**Step 1 — IAM audit (1 hour):**

Open every role in the CDK output. For each:
- Replace `s3:*` with specific actions.
- Scope resources to specific ARNs, not `*`.
- Add `Condition` blocks: `aws:SourceVpc`, `aws:SecureTransport`, `aws:ResourceTag/Project`.

**Step 2 — KMS refinement (30 min):**
- Confirm 2 CMKs: `key-s3` (all S3), `key-data` (RDS + Kinesis + DynamoDB + Redshift).
- Key policies grant only the specific service roles that need `Encrypt/Decrypt/GenerateDataKey`.

**Step 3 — Lake Formation with LF-tags (1.5 hour):**
- Register `s3://silver/` and `s3://gold/` with LF.
- Define tags:
  - `tier ∈ {1, 2, 3}`
  - `pii ∈ {none, low, high}`
  - `domain ∈ {llm}`
- Apply to Iceberg tables.
- Create roles:
  - `AnalystRole` — tables where `tier IN (1, 2) AND pii = 'none'`.
  - `ResearcherRole` — tables where `tier IN (1, 2, 3) AND pii IN ('none', 'low')`.
  - `AdminRole` — everything.
- Test: assume AnalystRole → query silver → tier-3 data invisible.

### Sunday (3 hours)

**Step 4 — Macie (30 min):**
- Enable Macie one-time job on `s3://raw/`.
- Schedule: once, then disable to avoid recurring cost.
- EventBridge rule: Macie finding → Lambda → tag the offending prefix with `pii=high` LF-tag.
- Reality check: for public LLM data (papers, docs), Macie will find near-zero PII. That's fine — you've proven the pattern works.

**Step 5 — VPC endpoints finalization (30 min):**
- Confirm interface endpoints for: Secrets Manager, KMS, Bedrock, Kinesis, DynamoDB, Athena.
- Gateway endpoints for: S3, DynamoDB.
- Cost per endpoint: ~$0.24/day × 6 = $43/mo if all active permanently. **Delete when idle** — VPC endpoints are the biggest idle cost.
- Alternative: parametrize with `enabled` flag, deploy stack with `enabled=false` to remove endpoints when done.

**Step 6 — Secrets Manager rotation (45 min):**
- Enable rotation on RDS master password. AWS auto-provisions a rotation Lambda.
- 30-day rotation schedule.
- Test by rotating manually and verifying app still connects.

**Step 7 — CI/CD pipeline (optional, 1.5 hour):**
- CodePipeline: GitHub source → CodeBuild (`cdk synth`) → CodeDeploy (`cdk deploy`).
- Fine to skip if time is tight — put it in the "extensions" list.

**Step 8 — Final polish (30 min):**
- Take screenshots of every dashboard.
- Record 90-second demo video.
- Write the README.

### Interview drills — Weekend 4

1. **How does IAM policy evaluation work?** Explicit DENY anywhere → deny. Else explicit ALLOW → allow. Else implicit deny.
2. **KMS envelope encryption at 3AM.** KMS returns plaintext + encrypted-under-CMK data key. Encrypt data locally with plaintext key. Store encrypted-data + encrypted-data-key. Decrypt: `Decrypt(encrypted-data-key)` → plaintext key → decrypt data. Reason: KMS payload limit is 4KB.
3. **LF-tags at 100-table scale.** Same policies — new tables inherit default tags, permissions auto-apply. Manual per-table grants would explode.
4. **Cross-account access to your search API.** External account creates IAM user or role → your API Gateway's resource policy grants that principal → they call with SigV4 signed requests. Or: create a Lambda authorizer that validates JWT tokens from their IdP.

---

## 11. Common pitfalls

1. **arXiv API returns Atom XML with escaped HTML** — parse cleanly with `feedparser` or handle `&amp;` explicitly. Test on a small sample first.
2. **HuggingFace token in Lambda env var** — never. Put in Secrets Manager, fetch on cold start, cache in module-scope global for warm invocations.
3. **Bedrock InvokeModel throttling** — happens fast at Glue-job scale. Batch 20 docs/call, `boto3` `Retry` config with `exponential_backoff`, `mode='adaptive'`. Consider Provisioned Throughput ($$$) at real scale.
4. **RDS in VPC, Lambda outside VPC** — Lambda can't reach RDS. Options: put Lambda in VPC (cold start), or use **RDS Data API** (no VPC needed). We use Data API.
5. **pgvector `<=>` returns *distance*, not *similarity*** — small numbers = more similar. `1 - distance` = cosine similarity. Common confusion in API responses.
6. **HNSW build blows RAM on t4g.micro** — building 100k vectors with `m=16` needs ~2GB RAM. If OOM, drop to `m=8` or upsize to t4g.small ($24/mo → **exceeds budget**). Alternative: build the index in batches of 10k rows.
7. **VPC endpoints charged per hour** — even at midnight. Set `enabled=False` at end of each session's CDK context; redeploy on Saturday morning.
8. **DMS instance forgotten** — $15/mo silent burn. `pause.py` script MUST delete the DMS instance at end of session.
9. **CloudWatch Logs 5GB free tier easily exceeded** — set retention on every log group (7 days for dev), and be careful with `print()` inside Glue jobs (goes to CloudWatch).
10. **Bedrock model access takes 24 hours for some models** — enable Nova Lite + Titan Embeddings NOW so they're ready Weekend 2.

---

## 12. README template + interview pitch

### README (repo landing page)

```markdown
# LLM Intelligence Platform

A serverless data-engineering platform on AWS that ingests **only high-signal LLM content** — arXiv papers, official docs from Anthropic / OpenAI / Google / Meta / Mistral / DeepSeek, HuggingFace model cards, and curated practitioner discourse — filters out AI-slop with a Bedrock quality classifier, extracts model comparisons + best practices, and serves semantic search across the actually-useful state of LLM knowledge.

100% AWS CDK. Runs at ~$5/month idle. Covers every domain of the AWS Data Engineer Associate cert.

![architecture](docs/architecture.png)

## Signal-quality tiering

Every document gets a `signal_tier` at ingest:
- **Tier 1 — Canonical:** arXiv papers, official vendor research posts, HuggingFace Daily Papers
- **Tier 2 — Authoritative:** vendor docs, HuggingFace model cards, LMSYS leaderboard
- **Tier 3 — Practitioner:** HackerNews (score > 20), r/LocalLLaMA + r/MachineLearning (top posts), GitHub issues on major projects

A Bedrock Nova Lite quality classifier gates enrichment — docs that fail the gate are moved to `low_signal/` and not embedded.

## Services used

S3 · Kinesis Data Streams · Firehose · DMS · RDS PostgreSQL + pgvector (HNSW) · DynamoDB · Redshift Serverless · Glue (Jobs, Catalog, Crawler, Data Quality) · Athena (partition projection) · Lambda · Step Functions · EventBridge · Bedrock (Nova Lite + Titan Embeddings) · API Gateway · Lake Formation (LF-tags) · Macie · IAM · KMS · Secrets Manager · VPC endpoints (no NAT!) · CloudWatch · CloudTrail · SNS · CDK (Python)

## Notable engineering choices

- **RDS db.t4g.micro + pgvector over Aurora Serverless v2** — 20× cheaper for demo scale
- **S3 Tables (managed Iceberg) over self-managed** — no compaction headache
- **Bedrock Nova Lite for enrichment** — 50× cheaper than Claude, sufficient for structured extraction
- **VPC endpoints instead of NAT gateway** — saves $32/mo
- **Athena partition projection instead of Glue crawler** — no crawler cost, zero-latency partitions
- **20-doc batching per Bedrock call** — 15× cost reduction
- **Signal-tier as partition column + LF-tag** — query pruning + access control in one design

## Try it

git clone …
cdk bootstrap && cdk deploy --all

## Cost

~$5/month idle. ~$25 total build across 4 weekends. `cdk destroy` cleans everything.

## What I learned

- Bedrock rate-limit patterns at Glue-job scale (adaptive retry, batching, Provisioned Throughput trade-offs).
- Iceberg vs Delta trade-offs (S3 Tables managed compaction changed my design).
- pgvector HNSW parameter tuning for the 100k–1M vector range.
- Lake Formation LF-tag design that scales to 100+ tables.
- Cost engineering — cut naïve $80/month architecture to $5 by choosing VPC endpoints, Nova Lite, and RDS over Aurora Serverless v2.

## Author

Manas — [LinkedIn] — [Portfolio]
```

### The 60-second interview pitch

> "I built a serverless data-engineering platform on AWS that ingests only high-signal LLM content — arXiv papers, official docs from Anthropic, OpenAI, Google DeepMind, Mistral, Meta, plus HuggingFace model cards and curated practitioner discourse from HackerNews and Reddit. The novel design decision was explicit signal-quality tiering — every document gets a `signal_tier` at ingest, from Tier 1 canonical papers down to Tier 3 practitioner posts, plus an explicit blocklist for AI-slop sources. Enrichment goes through Bedrock — Nova Lite for a quality classifier that gates the pipeline, then structured extraction of models mentioned, techniques discussed, and best practices, then Titan embeddings. Everything lands in an Iceberg lakehouse on S3 Tables with Lake Formation column-level access control based on tier. Semantic search runs on RDS Postgres with pgvector and an HNSW index — I picked RDS over Aurora Serverless v2 to save 40 dollars a month on idle cost. The whole thing is 100% CDK, runs at about five bucks a month idle, and covers every domain of the AWS Data Engineer Associate cert. My favorite piece was cost engineering — I cut the naïve architecture from about 80 dollars a month down to five by choosing VPC endpoints instead of a NAT gateway, Nova Lite instead of Claude for classification, RDS instead of Aurora Serverless, and Athena partition projection instead of Glue crawlers."

Practice that until it flows. It hits every DE hire signal (services, cost, LLM integration, IaC).

---

## 13. Extensions

If you want to go beyond the 4-weekend MVP:

- **Bedrock Agent** with tool use — natural-language Q&A over the lake ("what's the practitioner consensus on Claude 4.5 vs GPT-5 for structured extraction in the last 30 days?").
- **Trend anomaly detection** — Lambda that runs weekly, flags spikes in `technique=X` mentions per model → SNS alert. Blog-worthy chart.
- **A/B benchmark** two Bedrock models on classification accuracy → log results → publish a comparison. Genuinely useful research; also great content marketing for the project.
- **Cross-region DR** — S3 CRR + DynamoDB Global Table + Aurora Global Database. Not strictly needed but demos the "at scale" answer.
- **LMSYS leaderboard tracking** — daily snapshot, detect model rank changes, publish alerts.
- **Blog post** — "What I Learned Building an LLM Intelligence Platform on AWS." Post to your site + HackerNews. If it hits front page, it's a hiring event.
- **Streamlit → Next.js** if you want a slicker public demo.
- **Public API** — release the semantic search API publicly (rate-limited) as a free tool for the LLM community. Instant portfolio credibility.

---

## Resources

- [arXiv API docs](https://info.arxiv.org/help/api/index.html)
- [HuggingFace Hub API docs](https://huggingface.co/docs/hub/api)
- [HackerNews Firebase API docs](https://github.com/HackerNews/API)
- [Reddit OAuth docs](https://www.reddit.com/dev/api/)
- [pgvector docs](https://github.com/pgvector/pgvector) — HNSW parameter tuning guide.
- [S3 Tables docs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables.html)
- [Bedrock InvokeModel API](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_runtime_InvokeModel.html)
- [Glue Data Quality DQDL reference](https://docs.aws.amazon.com/glue/latest/dg/dqdl.html)
- [AWS CDK Python API](https://docs.aws.amazon.com/cdk/api/v2/python/)
- [trafilatura](https://trafilatura.readthedocs.io/) — clean HTML text extraction.

---

*This project is designed to be one you can be **proud of**, defend deeply in interviews, and finish inside a modest budget. Start Weekend 1 the same day you finish reading Domain 1.*
