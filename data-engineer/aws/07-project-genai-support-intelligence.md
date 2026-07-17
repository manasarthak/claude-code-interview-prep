# Capstone Project — GenAI Support Intelligence Platform

**Goal:** build a production-shape data-engineering project on AWS that (a) covers ~85% of DEA-C01 in-scope services, (b) incorporates LLMs for a 2026 resume differentiator, and (c) has a story recruiters can visualize in 30 seconds.

**Timeline:** 3–4 weekends of focused work (16–24 hours total). Extend or trim per your schedule. Start after (not during) exam prep unless you have extra time.

**Budget:** ~$30–60 for the entire build if you clean up promptly. Uses the AWS free tier heavily.

**Deliverables:**
1. Public GitHub repo with CDK infrastructure code + application code.
2. Architecture diagram (draw.io / excalidraw).
3. README that recruiters can read in 5 minutes.
4. 90-second demo video (Loom / YouTube unlisted).
5. Optional: blog post writeup.

---

## The One-Sentence Pitch

> Built a serverless GenAI-powered data platform on AWS that ingests customer support tickets from batch + streaming sources, enriches them with Amazon Bedrock LLMs (classification, summarization, embeddings), lands them in a governed Iceberg lakehouse with Lake Formation column-level PII protection, exposes semantic search via Aurora PostgreSQL pgvector, and serves analytics through QuickSight — all deployed via AWS CDK with monitoring, alarms, and Macie-driven PII discovery.

That single paragraph on your resume + a repo link is the pitch. Every phrase maps to a specific service or capability the DEA-C01 covered.

---

## Why This Project Wins on a Resume

1. **Covers ~15 in-scope services** — S3, Kinesis, Firehose, DMS, Glue, Lambda, Step Functions, EventBridge, Iceberg / S3 Tables, Aurora + pgvector, Redshift, Athena, QuickSight, IAM, KMS, Lake Formation, Macie, Bedrock, CloudWatch, CDK.
2. **LLM integration is the 2026 differentiator** — matches the new Skills 1.2.10 (LLM for data), 2.1.8 (vector index types), 2.4.6 (vectorization). Also demonstrates familiarity with the AI-native data patterns.
3. **Serverless-first** — proves you know cost management. Runs at ~$50/month peak; free tier + auto-suspend keeps idle cost near zero.
4. **Governance built in** — Lake Formation + Macie + KMS. This is what enterprise teams look for. Most portfolio projects skip this.
5. **Real business scenario** — "we ingest, enrich, and search support tickets" is instantly explainable to non-technical hiring managers.

---

## The Architecture

### High-level

```
┌─────────────────────────────────────────────────────────────────────┐
│                          INGEST                                      │
│                                                                      │
│  Historical CSV     ▶  S3 Raw       (batch)                         │
│  RDS Postgres       ▶  DMS  ▶  Kinesis  ▶  Firehose  ▶  S3 Raw     │
│  Real-time chat     ▶  Kinesis Data Streams                         │
│                                                                      │
└──────────────────────────────┬──────────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       ENRICH (LLM step)                              │
│                                                                      │
│  Batch: Glue Job (Spark) reads S3 Raw ▶  Bedrock InvokeModel ▶      │
│         classify + summarize + Titan embeddings                      │
│                                                                      │
│  Streaming: Lambda reads Kinesis ▶ Bedrock ▶ writes to DynamoDB     │
│             (hot state) + Firehose (S3)                              │
│                                                                      │
└──────────────────────────────┬──────────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       STORE (governed lakehouse)                     │
│                                                                      │
│  S3 Iceberg tables via S3 Tables (bronze / silver / gold)            │
│      registered in Glue Data Catalog + Lake Formation                │
│  Aurora PostgreSQL + pgvector (HNSW index) for semantic search       │
│  Redshift Serverless for BI-style aggregation                        │
│                                                                      │
└──────────────────────────────┬──────────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          SERVE                                       │
│                                                                      │
│  API Gateway  ▶  Lambda  ▶  Aurora pgvector  →  semantic search API │
│  QuickSight (via Redshift + Athena) → dashboards                    │
│  Athena workgroups → ad hoc SQL for analysts                        │
│  Bedrock Agent (optional) → natural-language Q&A over data          │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘

  ────────────  ORCHESTRATION  ────────────
  Step Functions  ▶  parent workflow (crawl → enrich → publish)
  EventBridge Scheduler  ▶  daily trigger
  Glue Workflow  ▶  crawler chain
  Glue Data Quality  ▶  DQDL rules

  ────────────  OBSERVABILITY  ────────────
  CloudWatch Logs (all Lambda + Glue)  •  Logs Insights queries
  CloudWatch Metrics + Alarms  •  SNS notifications
  CloudTrail organization trail  ▶  centralized S3
  Macie PII scan  ▶  Lambda auto-tag  ▶  Lake Formation LF-tag

  ────────────  SECURITY  ────────────
  IAM roles (per-Lambda, per-Glue-job, per-service)
  KMS CMKs for S3, DynamoDB, Kinesis, Aurora, Redshift
  Lake Formation column-level (mask PII to analyst role)
  VPC endpoints (S3, Kinesis, Bedrock)
  Secrets Manager (Aurora master password, rotation)

  ────────────  DEPLOY  ────────────
  AWS CDK (Python)  ▶  CloudFormation
  CI/CD via CodeBuild + CodePipeline
```

---

## The Data Model

### The story

You run a fictional company **HelpSpring** whose support team handles ~50,000 tickets/month across email, chat, and a legacy CRM. Analysts want to know:
- What are the top 10 issue categories this month?
- Which product areas have rising ticket volume?
- Semantic search: "show me tickets similar to this one" for the support team.
- LLM Q&A: "what were the most common billing complaints last week?"

### Data sources (simulated)

1. **Historical tickets CSV** — 100k rows loaded into S3 once (batch).
2. **RDS PostgreSQL** — a "current tickets" table you'll DMS-replicate into S3 as CDC.
3. **Real-time chat messages** — generated by a load script writing to Kinesis.

Ticket shape:
```json
{
  "ticket_id": "T-2026-000123",
  "customer_id": "C-98765",
  "customer_email": "user@example.com",
  "created_at": "2026-07-15T10:22:00Z",
  "channel": "email",
  "subject": "billing charge unexpected",
  "body": "I was charged twice for my subscription this month...",
  "priority": "medium",
  "status": "open"
}
```

### Enrichment fields (added by LLM)

- `category` — Bedrock classifies into {Billing, Bug, Feature Request, Account, Other}.
- `sentiment` — {Positive, Neutral, Negative}.
- `summary` — 2-sentence Bedrock summary.
- `pii_detected` — Macie tag.
- `embedding` — 1024-dim Bedrock Titan embedding.

---

## Weekend-by-Weekend Build Plan

### Weekend 1 — Foundation & Ingest (~6 hrs)

**Goals:** land raw data in S3 from all three sources. Glue Catalog + Athena works.

**Tasks:**
1. **CDK bootstrap** — create the project, `cdk init app`, deploy the base stack (VPC, KMS CMK, log group).
2. **S3 buckets** — raw, silver, gold, logs. Enable versioning + SSE-KMS. Add lifecycle rules (raw → IA after 30d, → Glacier after 90d).
3. **RDS Postgres** in a private subnet. Seed with sample ticket table (100 rows).
4. **DMS task** — full-load + CDC from RDS → Kinesis → Firehose → S3.
5. **Batch load script** — Python that uploads a big historical CSV to S3.
6. **Kinesis Data Streams** — a "chat_stream" topic. Write a producer script (Python) that pushes 10 events/second for a few minutes.
7. **Glue Crawler** — over `s3://raw/tickets/`. Registers table in Data Catalog.
8. **Athena workgroup** — configured with query result location + KMS.
9. **Query in Athena** — validate: `SELECT COUNT(*), channel FROM tickets_raw GROUP BY channel`.

**Test at end of weekend:** three data sources land in `s3://raw/*`. Athena queries the catalog and returns counts.

### Weekend 2 — Transform & Enrich with Bedrock (~6 hrs)

**Goals:** LLM enrichment pipeline works. Iceberg tables in silver layer.

**Tasks:**
1. **Enable Bedrock** in your region. Request model access for **Claude Sonnet** (or Nova Lite for cheaper) and **Titan Text Embeddings v2**.
2. **Glue Job (PySpark)** — reads `s3://raw/tickets/`, calls Bedrock InvokeModel per row (or micro-batch) for:
   - Classification (few-shot prompt).
   - Summarization (2 sentences).
   - Sentiment.
   - Embedding via Titan.
3. **Rate-limit handling** — SDK retries with exponential backoff + Glue job DPU tuning.
4. **Output:** write to `s3://silver/tickets_enriched/` as **Iceberg via S3 Tables** (managed).
5. **Streaming path:** Lambda reads from Kinesis, calls Bedrock, writes to DynamoDB (hot state) + Firehose (S3 silver append).
6. **Glue Data Quality (DQDL)** — rules on the silver table:
   ```
   Rules = [
     IsComplete "ticket_id",
     Uniqueness "ticket_id" > 0.99,
     ColumnValues "category" in ["Billing","Bug","Feature","Account","Other"],
     ColumnValues "sentiment" in ["Positive","Neutral","Negative"]
   ]
   ```
7. **CloudWatch dashboard** for pipeline health: Glue job duration, Bedrock invocation count, DQ failure rate.

**Test at end of weekend:** enriched data in silver Iceberg table. Athena `SELECT category, sentiment, COUNT(*) FROM silver.tickets_enriched GROUP BY category, sentiment`.

### Weekend 3 — Vector Search + Governance (~6 hrs)

**Goals:** semantic search via Aurora pgvector. Lake Formation column-level security. Macie scan.

**Tasks:**
1. **Aurora PostgreSQL** cluster + serverless v2. Install pgvector extension.
2. **Vector index:**
   ```sql
   CREATE TABLE ticket_vectors (
       ticket_id text PRIMARY KEY,
       embedding vector(1024)
   );
   CREATE INDEX ON ticket_vectors USING hnsw (embedding vector_cosine_ops);
   ```
3. **Glue job step** to also write embeddings to Aurora (from silver enrichment step).
4. **API Gateway + Lambda** — `POST /search` endpoint:
   - Body: `{"query": "billing double charged"}`.
   - Lambda: embed via Bedrock Titan → query Aurora pgvector → return top-K ticket IDs + summaries.
5. **Lake Formation setup:**
   - Register the silver database with LF.
   - Grant permissions to two roles: `AnalystRole` (masked email), `AdminRole` (full).
   - Column-level rule: `customer_email` visible only to `AdminRole`.
6. **Macie scan** on the raw bucket. Alert via EventBridge → Lambda that tags PII columns in LF.
7. **Redshift Serverless** workgroup + external schema over the silver Iceberg table. Test SQL: `SELECT category, DATE_TRUNC('week', created_at), COUNT(*) FROM silver.tickets GROUP BY 1, 2`.
8. **QuickSight dashboard** — connect to Redshift, build 4 charts (volume by category over time, sentiment breakdown, top complaints word cloud, average resolution time). Set up SPICE refresh daily.

**Test at end of weekend:** `curl POST /search` returns semantically-relevant tickets. QuickSight dashboard renders.

### Weekend 4 — Orchestration, Observability, Polish (~6 hrs)

**Goals:** it all runs on a schedule. Alerts wired. Deployment via CI/CD. README + demo video.

**Tasks:**
1. **Step Functions parent workflow:**
   - EventBridge Scheduler → Step Functions at 2am daily.
   - States: run Glue crawler → Glue enrichment job → Glue data quality → publish to gold layer → refresh QuickSight SPICE → SNS success notification.
   - Catch/Retry: fail state → SNS error → PagerDuty (or your email).
2. **CloudWatch alarms:**
   - Glue job failure count > 0 in 15 min.
   - DMS replication lag > 15 min.
   - Bedrock throttling errors > 10/min.
   - Kinesis IteratorAge > 5 min.
3. **CloudTrail data events** on the S3 buckets. CloudTrail Lake for centralized query.
4. **CDK CI/CD:** CodeBuild + CodePipeline that triggers on git push, runs `cdk synth` + tests + `cdk deploy`.
5. **Cost tagging** — every resource tagged with `Project=SupportIntel`, `Environment=Dev`.
6. **README** — write it (see template below).
7. **Architecture diagram** — draw.io export as PNG in repo.
8. **Demo video** — 90 seconds screen recording:
   - 5s: "This is Support Intelligence, a serverless GenAI data platform I built on AWS."
   - 20s: Show data landing in S3 via load script; show streaming producer.
   - 20s: Show Bedrock enrichment step running in Step Functions.
   - 20s: `curl POST /search` demo, showing semantic results.
   - 15s: QuickSight dashboard.
   - 10s: "Deployed with 100% CDK, covers all four DEA-C01 exam domains. Repo in description."

**Test at end of weekend:** cold start from `cdk deploy` on a fresh account works. `git push` triggers redeploy.

---

## Cost Estimates (running the demo)

Ballpark for a single dev environment left running:

| Service | Monthly cost |
|---|---|
| Aurora Serverless v2 (0.5 ACU min, idle most of the time) | ~$40 |
| S3 storage (<10 GB) | ~$1 |
| Glue jobs (a few DPU-hours/day) | ~$5 |
| Bedrock invocations (Claude Sonnet ~$3 per 1M input tokens; Nova Lite ~$0.06 per 1M) | ~$2–10 |
| Redshift Serverless (idle-only) | ~$0 (base storage free) |
| DynamoDB On-Demand | ~$0 |
| Lambda + API Gateway (low traffic) | ~$0 |
| Kinesis (1 shard) | ~$11 |
| DMS instance (t3.micro) | ~$15 |
| CloudWatch Logs | ~$1 |
| QuickSight Enterprise (1 user, monthly) | $18 |
| **Total (running 24/7)** | **~$95** |
| **With aggressive turn-off** (Aurora paused, DMS deleted, single-shot demo) | **~$30** |

**Cost management tips:**
- Aurora: pause when not testing (auto-pause at 5 min idle).
- Kinesis: delete stream when idle; recreate for demo. Or use On-Demand.
- DMS: delete replication instance between test runs.
- QuickSight: use Standard edition for personal demo ($9/mo) or free tier trial.
- Bedrock: switch demo to Nova Lite ($0.06/M input tokens) for embeddings + classification. Reserve Claude for the summary step.
- Delete everything when done: `cdk destroy` and verify billing.

**Set an AWS Budget alarm at $20/week.** Peace of mind.

---

## The README Template (recruiters read this)

```markdown
# GenAI Support Intelligence Platform

A serverless data-engineering platform on AWS that ingests customer support tickets from batch + streaming sources, enriches them with Amazon Bedrock LLMs (Claude, Titan), and provides semantic search + BI analytics — deployed entirely via AWS CDK with Lake Formation governance.

## Architecture

![architecture](docs/architecture.png)

## Highlights

- **Ingest:** S3 batch + RDS→DMS→Kinesis CDC + Kinesis streaming
- **Enrich:** Glue + Lambda calling Bedrock (classification, summarization, embeddings)
- **Store:** S3 Iceberg tables (S3 Tables) + Aurora PostgreSQL pgvector (HNSW) + Redshift Serverless
- **Serve:** semantic search API (API Gateway + Lambda) + QuickSight dashboards + Athena
- **Orchestrate:** Step Functions + EventBridge Scheduler + Glue Data Quality
- **Govern:** Lake Formation column-level security, Macie PII discovery, KMS CMKs
- **Deploy:** 100% AWS CDK (Python), CI/CD via CodePipeline

## Services used

<15 in-scope AWS services listed>

## Demo

[90-second video](https://loom.com/...)

## Try it yourself

git clone …
pip install -r requirements.txt
cdk bootstrap
cdk deploy --all

## Cost

~$30–60/month while running. `cdk destroy` cleans everything.

## What I learned

- Bedrock rate-limit patterns at scale (batching + provisioned throughput trade-offs).
- Iceberg vs S3 Tables — why managed compaction changed my design.
- Aurora pgvector HNSW parameters (m, ef_construction) for recall/latency trade.
- Lake Formation LF-tags vs explicit permissions at repo scale.

## Author

Manas — [LinkedIn](...) — [Portfolio](...)
```

---

## Common Pitfalls (and how to avoid them)

1. **Bedrock throttling.** Default request-per-second limits are low. Solutions: batch, add exponential-backoff retries, request quota increase, or use Provisioned Throughput for demo runs.
2. **Aurora Serverless slow cold start.** First query after auto-pause takes ~30s. Warm-up a job before your demo.
3. **KMS-per-service policy sprawl.** Use one CMK per major service (S3, Aurora, Kinesis) with clear key policies rather than per-resource.
4. **DMS replication instance NAT.** RDS→DMS→S3 needs proper networking. Put DMS instance in private subnet with a VPC endpoint for S3.
5. **Glue job Bedrock in loop.** Naïve `.foreach()` calling Bedrock per row will be slow. Batch rows to Bedrock in groups of 10–20 per Spark task, use Pandas UDFs.
6. **pgvector index tuning.** Default `m=16, ef_construction=64` is fine for ≤1M vectors. For higher recall or bigger corpus, tune.
7. **Lake Formation "principal not registered."** Grant `DATA_LAKE_ADMIN` on the target account. Common early error.
8. **Costs snowballing.** Set budget alarms immediately. Don't wait until day 4 to check billing.

---

## Optional Extensions (post-MVP)

- **Bedrock Agent** with tool use — natural-language Q&A over the ticket lake ("what were the top complaints last week?").
- **Cross-region Global Table** for the DynamoDB hot cache.
- **Multi-account setup** with Control Tower + Organizations.
- **Terraform version** alongside CDK as portfolio proof of multi-IaC.
- **DBT** models on top of Redshift for analyst-friendly transformations.
- **Streamlit / Next.js frontend** for the search API — makes the demo video pop.
- **A/B test two Bedrock models** on classification and log the accuracy metrics.

---

## The Interview Story

When asked "tell me about a project" in interviews:

> "I built a serverless GenAI data platform on AWS end-to-end. It ingests customer support tickets from a batch S3 dump, a Postgres CDC feed via DMS, and a real-time Kinesis chat stream. Every ticket gets enriched by Bedrock — Claude for classification and summarization, Titan for embeddings — which land in an S3 Tables Iceberg lakehouse governed by Lake Formation with column-level PII masking. The embeddings power a semantic-search API on Aurora pgvector with HNSW indexing, and Redshift Serverless feeds QuickSight dashboards. The whole thing is a single `cdk deploy` — no ClickOps — and I wired CloudWatch alarms, CloudTrail data events, and Macie for continuous compliance. It runs at about $30–60/month and covers every domain of the AWS Data Engineer cert. The trickiest part was managing Bedrock throttling at Glue-job scale — I ended up batching 20 rows per invocation and using provisioned throughput for the demo runs."

That's a 90-second pitch that sells you as a **senior-shape data engineer who understands AWS deeply and knows how to ship**. Practice it.

---

## Resources

- [AWS CDK Python docs](https://docs.aws.amazon.com/cdk/api/v2/python/)
- [Bedrock InvokeModel API](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_runtime_InvokeModel.html)
- [pgvector docs](https://github.com/pgvector/pgvector)
- [S3 Tables docs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables.html)
- [AWS Solutions Constructs](https://docs.aws.amazon.com/solutions/latest/constructs/) — pre-built CDK patterns.
- [Real-world DE projects: Andreas Kretz](https://learndataengineering.com/) — for structural inspiration.
- [System Design for DE](../15-system-design-de.md) — the DE framework this project follows.

---

*Build this, ship it, and put it at the top of your resume. Then use the exam prep + this project as your interview foundation.*
