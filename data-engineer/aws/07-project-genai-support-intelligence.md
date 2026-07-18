# Capstone Project — OSS Community Intelligence Platform

**Real problem, real data:** ingest **GitHub Archive** (every public GitHub event, ~5M events/day, freely available since 2015) for a curated set of ML/data-infra repos (PyTorch, TensorFlow, Spark, Ray, LangChain, Iceberg, DuckDB, etc.), enrich issues/PRs/discussions with Amazon Bedrock LLMs, expose semantic search + trend analytics.

**Why this project, not "dummy support tickets":** this is the exact shape of internal data platform that companies like **NVIDIA, Databricks, HuggingFace, OpenAI, Anthropic, Snowflake, and Confluent run** to understand OSS community health, developer pain points, and adoption trends. Recruiters at those companies will *instantly* recognize the pattern — "I could hand this to you Monday and you'd know what to do."

**Timeline:** parallel with your 14-day study plan. Each weekend implements the AWS domain you just finished studying.

**Budget target:** **$10–25 total across 4 weekends** of active work, ~$1/month at rest. Full breakdown below.

**Deliverables:** public GitHub repo, architecture diagram, 5-min README, 90-second demo video.

---

## The One-Sentence Pitch (for résumé + interviews)

> Built a serverless GenAI-powered data platform on AWS that ingests the entire GitHub Archive event stream for the top ML/data-infrastructure OSS projects, enriches issues/PRs with Amazon Bedrock for classification, sentiment, severity, and embeddings, lands them in a governed Iceberg lakehouse on S3, and exposes semantic search + trend analytics — deployed via AWS CDK with column-level access control (Lake Formation), Macie PII discovery, and CloudWatch observability.

---

## Why This Data Source

**GitHub Archive** (gharchive.org):
- **Real, public data** — every push, PR, issue, review, star, fork, discussion since 2015.
- **Massive scale** — ~5M events/day, ~150M events/month, ~2TB/year JSON.
- **Free** — hosted at `https://data.gharchive.org/YYYY-MM-DD-H.json.gz` (one file per hour).
- **Directly relevant** — this is *the* dataset for OSS community intelligence.

Company relevance:
- **NVIDIA / AMD** — track CUDA / ROCm ecosystem issues, PyTorch/JAX/Triton adoption.
- **Databricks / Snowflake / Confluent** — track Spark / Delta / Iceberg / Kafka issues + PRs.
- **HuggingFace / OpenAI / Anthropic** — track LangChain / vLLM / Ollama / model-repo activity.
- **AWS / GCP / Azure** — track their own SDK repo health.

The pitch to a hiring manager: *"I built the data platform your DevRel team wishes they had."*

---

## Architecture (single diagram)

```
┌──────────────────────────────────────────────────────────────────────┐
│                          INGEST                                       │
│                                                                       │
│  Batch:    gharchive.org  ▶ Lambda scheduler  ▶  S3 raw               │
│            (one .json.gz per hour, last 30 days = ~600GB but we       │
│             only need ~2-5GB for demo)                                │
│                                                                       │
│  Live:     GitHub Events API  ▶  Lambda poller  ▶  Kinesis Streams   │
│            (real-time events from the tracked repos)                  │
│                                                                       │
│  CDC:      RDS Postgres (repo metadata, curated list, star counts)    │
│            ▶  DMS  ▶  Kinesis  ▶  Firehose  ▶  S3 raw                 │
│                                                                       │
└──────────────────────────────┬───────────────────────────────────────┘
                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│                          ENRICH (LLM step)                            │
│                                                                       │
│  Glue Job (PySpark) reads S3 raw → filters to tracked repos →         │
│    calls Bedrock (Nova Lite for cheap classification + Titan          │
│    embeddings) for each issue/PR/discussion:                          │
│                                                                       │
│    - category   : bug | feature | question | docs | ci | perf         │
│    - severity   : low | medium | high | critical                      │
│    - sentiment  : positive | neutral | negative                       │
│    - summary    : 2 sentences                                          │
│    - embedding  : 1024-dim vector                                     │
│                                                                       │
│  Streaming path: Lambda → Bedrock → DynamoDB (hot cache) + Firehose   │
│                                                                       │
└──────────────────────────────┬───────────────────────────────────────┘
                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│                          STORE                                        │
│                                                                       │
│  S3 Iceberg tables (via S3 Tables)  ▶  Glue Data Catalog             │
│    bronze/  = raw JSON as-landed                                      │
│    silver/  = flattened + enriched Iceberg tables                     │
│    gold/    = aggregations (repo × week × category)                   │
│                                                                       │
│  RDS PostgreSQL + pgvector (HNSW)   ← embeddings for search           │
│    (free tier t4g.micro — no Aurora Serverless)                       │
│                                                                       │
│  Redshift Serverless (external schema over Iceberg)                   │
│    ← analyst SQL for weekly reports                                   │
│                                                                       │
│  DynamoDB on-demand ← hot state for streaming path                    │
│                                                                       │
└──────────────────────────────┬───────────────────────────────────────┘
                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│                          SERVE                                        │
│                                                                       │
│  API Gateway → Lambda → RDS pgvector       (semantic search API)      │
│  Athena (partition projection)              (ad-hoc analyst SQL)      │
│  Streamlit dashboard on ECS Fargate (free)  (or QuickSight if paid)   │
│  Optional: Bedrock Agent for NL Q&A over the lake                     │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘

── ORCHESTRATION ──   Step Functions ← EventBridge Scheduler (hourly)
── OBSERVABILITY  ──   CloudWatch Logs + Insights, alarms → SNS email
── GOVERNANCE     ──   Lake Formation LF-tags, Macie PII scan, KMS CMK
── DEPLOY         ──   AWS CDK (Python), CodeBuild+CodePipeline
```

---

## The Cost Breakdown (real, minimized)

### While actively building (per weekend of ~6 hrs work)

| Service | Config | Cost per weekend |
|---|---|---|
| Bedrock — Nova Lite | ~$0.06 per 1M input tokens, ~$0.24 per 1M output. Enrich ~10k events (avg 200 tokens in / 100 out) = 2M input + 1M output tokens | ~$0.35 |
| Bedrock — Titan Embeddings v2 | $0.02 per 1M tokens. 10k events × 200 tokens = 2M tokens | ~$0.04 |
| Kinesis Data Streams — **On-Demand** | Per-GB ingested only. Light demo traffic (<1 GB) | ~$0.10 |
| Kinesis Firehose | $0.029 per GB ingested + $0.017 per GB delivered. ~1 GB round trip | ~$0.05 |
| S3 storage | ~10 GB × $0.023/GB-month × ¼ month | ~$0.06 |
| S3 requests | Free tier covers most; overage ~$0 | ~$0 |
| Glue Jobs | $0.44 per DPU-hour. 2 DPU × 30 min per job × 4 jobs = 4 DPU-hrs | ~$1.76 |
| Glue Crawlers | $0.44 per DPU-hour × 0.5 hr | ~$0.22 |
| Glue Data Catalog | 1M objects free. You'll be well under. | ~$0 |
| Lambda + API Gateway | Free tier covers everything at this scale | ~$0 |
| RDS Postgres db.t4g.micro | **Free tier for 12 months on new account**; else $0.016/hr = ~$12/mo. **Stop the instance between weekends** = only storage cost | free tier: $0; paid: ~$0.30 per weekend of runtime |
| DynamoDB On-Demand | Free tier: 25 GB + 25 RCU + 25 WCU free forever | ~$0 |
| Redshift Serverless | $0 base storage; 8 RPU × $0.375/hr only when queried. Assume 30 min query time | ~$1.50 |
| CloudWatch Logs | 5 GB free/month; overage $0.50/GB | ~$0 |
| CloudTrail | Management events free; **don't enable data events for now** | ~$0 |
| KMS CMKs (2) | $1/CMK/month + $0.03 per 10k requests. Prorated per weekend | ~$0.60 |
| DMS replication instance (t3.micro) | $0.02/hr = $0.20 for a 10-hr session. **Delete between weekends.** | ~$0.20 |
| **Total per active weekend** | | **~$5** |
| **Total across 4 weekends** | | **~$20** |

### Between weekends (idle)

Stop / delete the following when not working:
- RDS instance: stop it → pay only ~$0.50/wk storage.
- DMS replication instance: delete it → $0.
- Kinesis On-Demand: no idle cost — leave it.
- Redshift Serverless: no idle cost — leave it.
- Glue jobs: no idle cost — leave them.
- Lambda / API GW: no idle cost — leave them.
- S3 / DynamoDB / RDS storage: ~$1/week total.

**Idle cost:** ~$1–2 per week between weekends.

### After the project is done

Two options:
- **Keep it deployed as a portfolio piece** ($3–5/month idle).
- **`cdk destroy` and rely on GitHub repo + README + demo video** ($0/month, redeployable anytime).

### Hard cost ceilings — set these NOW

1. **AWS Budget alarm at $15/month.** Email + SNS notification.
2. **Bedrock model access:** enable **Nova Lite** (cheap) and **Titan Embeddings v2** only. Do NOT enable Claude Opus (expensive) unless doing a specific demo.
3. **Kinesis On-Demand, not provisioned.** Provisioned 1 shard = $11/month whether you use it or not.
4. **Skip QuickSight Enterprise** ($18/user/month). Use Streamlit on Fargate free tier or your local machine.
5. **Delete DMS instance** at the end of every session (5 seconds via `cdk destroy` on the DMS stack).

**Total realistic project cost: $15–25.** Less than one AWS certification exam voucher.

---

## Parallel Study + Build Timeline (aligned with the 14-day plan)

Each weekend implements the AWS domain you just studied that week. This is deliberate — you'll retain the exam material 3–5× better than reading alone.

### Weekend 1 — Domain 1 (Ingestion + Transformation)

**You've just studied:** S3, Kinesis, MSK, DMS, Glue, EMR, Lambda, Step Functions, EventBridge, MWAA, CDK, Bedrock.

**Build this weekend (~6 hrs):**

1. **CDK bootstrap** (30 min)
   ```bash
   npm i -g aws-cdk
   cdk init app --language=python
   cdk bootstrap
   ```
2. **Base stack** (30 min): VPC (2 AZs, 1 NAT — or 0 NAT + use VPC endpoints to save $32/mo), 2 KMS CMKs (one for S3, one for RDS), a base log group.
3. **Ingest — batch from GitHub Archive** (1 hr):
   - Lambda function on EventBridge Scheduler (every hour) that fetches `https://data.gharchive.org/YYYY-MM-DD-H.json.gz`, writes to `s3://.../raw/gha/dt=YYYY-MM-DD/h=H/events.json.gz`.
   - For the initial backfill: run the Lambda manually 24 times (last 24 hours = 5 GB) or 168 times for a week.
4. **Ingest — real-time via Kinesis** (1.5 hr):
   - Lambda poller against `https://api.github.com/repos/{owner}/{repo}/events` for a hardcoded list of 10 repos (start with pytorch/pytorch, apache/spark, langchain-ai/langchain, huggingface/transformers, etc.).
   - Runs every 60 seconds via EventBridge Scheduler.
   - Writes new events to Kinesis Data Streams (On-Demand mode).
   - Firehose subscribes → buffers 60s / 5 MB → writes to `s3://.../raw/live/dt=YYYY-MM-DD/`.
5. **Ingest — CDC from RDS** (1 hr):
   - RDS Postgres db.t4g.micro with a `tracked_repos` table (repo_full_name, category, priority, notes).
   - DMS replication instance + task: full-load + CDC → Kinesis → Firehose → `s3://.../raw/repos/`.
   - **Delete DMS instance at end of session.**
6. **Glue Crawler** (30 min): crawler over `s3://.../raw/` → creates 3 tables in the Data Catalog.
7. **Glue Transform Job (PySpark, no Bedrock yet)** (1 hr):
   - Reads all three raw sources, flattens JSON, writes to `s3://.../silver/events_flat/` as **Iceberg via S3 Tables**, partitioned by `event_date`.
8. **Step Functions state machine** (30 min): daily trigger → crawler → transform job → success SNS. Basic version; no error handling yet.
9. **Validate in Athena** (30 min): `SELECT type, COUNT(*) FROM silver.events_flat WHERE event_date = current_date GROUP BY type ORDER BY 2 DESC LIMIT 20`. Should show `PushEvent`, `IssuesEvent`, `PullRequestEvent`, etc.

**End of Weekend 1 test:** three data sources landing in S3; Athena returns real event counts; Step Functions runs the daily transform end-to-end.

**Cost this weekend:** ~$3–5.

### Weekend 2 — Domain 2 (Data Store Management)

**You've just studied:** Redshift, DynamoDB, Iceberg + S3 Tables, Glue Catalog, S3 Lifecycle, HNSW/IVF, Aurora pgvector.

**Build this weekend (~6 hrs):**

1. **Bedrock enable** (10 min): request model access for Nova Lite + Titan Embeddings v2 in your region.
2. **Glue Job v2 — with Bedrock enrichment** (2 hr):
   - Reads Iceberg silver table `events_flat`.
   - Filters to `type IN ('IssuesEvent', 'PullRequestEvent', 'DiscussionEvent')`.
   - Batches rows in groups of 20 per Spark task; calls Bedrock Nova Lite with a few-shot classification prompt returning JSON `{category, severity, sentiment, summary}`.
   - Calls Bedrock Titan Embeddings v2 on the issue/PR body → 1024-dim vector.
   - Writes to `s3://.../silver/events_enriched/` as Iceberg.
   - Uses Glue bookmarks so re-runs are incremental.
3. **RDS pgvector setup** (1 hr): install pgvector extension, create `event_vectors(event_id text pk, repo text, type text, embedding vector(1024))`, `CREATE INDEX ... USING hnsw`.
4. **Glue step to sync embeddings to RDS** (1 hr): from the enrichment job, batch-insert vectors into RDS. Use RDS Data API (no VPC networking headache) if possible.
5. **DynamoDB hot table** (30 min): `event_hot(event_id PK, ttl, category, severity, embedding_ref)` with 24-hour TTL. Populated by the streaming Lambda path.
6. **Redshift Serverless workgroup + external schema** (45 min): connect via Glue Catalog + Lake Formation, `CREATE EXTERNAL SCHEMA silver ...`, run a weekly aggregation query.
7. **S3 Lifecycle policies** (30 min): raw → IA at 30 days → Deep Archive at 90 days.
8. **Athena** (30 min): validate enriched data — `SELECT category, severity, COUNT(*) FROM silver.events_enriched WHERE event_date = current_date - 1 GROUP BY 1, 2`.

**End of Weekend 2 test:** enriched Iceberg table has category/sentiment/embedding for real GitHub issues. pgvector-based semantic search works via direct SQL.

**Cost this weekend:** ~$5–8 (Bedrock is the biggest single line).

### Weekend 3 — Domain 3 (Operations + Analysis)

**You've just studied:** Athena, QuickSight, DataBrew, CloudWatch, CloudTrail, Config, Glue Data Quality, sampling, skew handling.

**Build this weekend (~6 hrs):**

1. **Athena partition projection** (30 min): rewrite the raw event table with projection so no crawler needed for daily partitions.
2. **Glue Data Quality DQDL** (1 hr):
   ```
   Rules = [
     IsComplete "event_id",
     Uniqueness "event_id" > 0.99,
     ColumnValues "category" in ["bug","feature","question","docs","ci","perf","other"],
     ColumnValues "severity" in ["low","medium","high","critical"],
     RowCount between 100 and 10000000
   ]
   ```
   Attach to the enrichment job; fail the job on DQ violation.
3. **Search API** (2 hr): API Gateway → Lambda that embeds the query with Bedrock → queries pgvector → returns top-10 similar events with LLM-generated one-line explanations.
4. **Streamlit dashboard on Fargate** (or your local machine to save $10/mo) (1.5 hr):
   - Trending issues by category over time (chart from Athena).
   - Sentiment breakdown per repo.
   - Word cloud of common phrases in `bug` category.
   - Semantic search box → hits the API from step 3.
5. **CloudWatch monitoring** (45 min):
   - Dashboard: Glue job duration, Bedrock invocation count, Kinesis IteratorAge, API latency p99.
   - Alarms: Glue failures > 0, Bedrock throttling > 10/min, API p99 > 2s.
   - SNS topic → your email.
6. **CloudWatch Logs Insights saved queries** (15 min): e.g., `fields @timestamp, @message | filter @message like /ERROR/ | stats count() by bin(5m)`.
7. **CloudTrail** (15 min): keep management events only (free). Skip data events unless demoing.

**End of Weekend 3 test:** dashboard renders; search API returns semantically-relevant events; alarms trigger correctly when you manually fail a job.

**Cost this weekend:** ~$4–6.

### Weekend 4 — Domain 4 (Security + Governance)

**You've just studied:** IAM policy evaluation, KMS envelope encryption, Lake Formation LF-tags, Macie, VPC endpoints, cross-account.

**Build this weekend (~6 hrs):**

1. **IAM least-privilege pass** (1.5 hr): audit every Lambda/Glue role. Replace `s3:*` with specific actions on specific ARNs. Add `Condition` blocks (source VPC, encryption required, TLS enforced).
2. **KMS refinement** (30 min): one CMK per major service (S3, RDS, Kinesis) with per-service key policies. Encrypt Kinesis + DynamoDB + S3 with customer-managed CMKs, not AWS-managed.
3. **Lake Formation setup** (1.5 hr):
   - Register S3 lake locations with LF.
   - Define two roles: `AnalystRole` (masked user emails in event payloads), `AdminRole` (full).
   - Create LF-tags: `pii=high|low`, `domain=oss|internal`.
   - Tag the `actor.email` column with `pii=high`.
   - Grant `AnalystRole` all tables with `pii=low` visibility.
   - Test: query silver as Analyst → email column redacted.
4. **Macie scan** (30 min): enable Macie on the raw bucket. Manual job for one-time PII discovery. EventBridge rule that catches findings → Lambda that adds LF-tag `pii=high` to the offending column.
5. **VPC endpoints** (1 hr): gateway endpoint for S3, interface endpoint for Kinesis + Bedrock + DynamoDB. Now the Lambdas in your VPC never egress to public internet. **Massive win: you can delete the NAT gateway ($32/mo saved).**
6. **Secrets Manager** (30 min): move RDS master password from CDK env vars to Secrets Manager with 30-day rotation.
7. **Bucket policies** (30 min): deny `aws:SecureTransport=false`, deny non-KMS uploads.
8. **CI/CD** (optional if you have time): CodePipeline that triggers on git push, runs `cdk synth` + tests + `cdk deploy`.

**End of Weekend 4 test:** analyst role sees masked emails, admin role sees raw. Macie flags PII. `cdk destroy` cleanly tears down everything.

**Cost this weekend:** ~$3–5 (mostly the endpoints during test).

### Post-project: README + demo video (1 evening)

- Architecture diagram (draw.io or Excalidraw).
- 5-min README (template below).
- 90-second Loom demo video.
- Push to GitHub with clean commits.

---

## Data Volume Sanity Check

You do **not** need to ingest the full GitHub Archive (~2 TB/year) — you just need enough for a compelling demo.

Recommendation:
- **Backfill: 3 days** of GHA data (~500 MB, ~15M events, ~$0 in storage).
- **Live streaming: 10 repos**, polled every minute (~1K events/hour).
- **Enrich only issues + PRs + discussions** (~5–10% of total volume, so ~50–500K events).

If you want to look impressive on the resume: process **30 days** (~5 GB, ~150M events, ~$1 in storage). Total Bedrock cost even at 30 days with Nova Lite embeddings would be $10–15.

---

## The GitHub Repos to Track (curated list)

Start with these 10 — representative of the ML/data-infra space:

```python
TRACKED_REPOS = [
    "pytorch/pytorch",
    "tensorflow/tensorflow",
    "apache/spark",
    "apache/iceberg",
    "duckdb/duckdb",
    "langchain-ai/langchain",
    "huggingface/transformers",
    "ray-project/ray",
    "vllm-project/vllm",
    "delta-io/delta",
]
```

Extend to 50 or 100 for a more impressive demo. The event volume scales linearly.

---

## Common Pitfalls (specific to this design)

1. **GitHub API rate limit** — 5,000 requests/hour with a personal access token. Store the token in Secrets Manager. Track remaining rate via response headers.
2. **GHA files are big-ish** — 5–20 MB per hour. Lambda has 6 MB response limit for synchronous invocations but that doesn't apply to writing S3. Just don't try to return the whole file from Lambda.
3. **Bedrock rate limits** — Nova Lite defaults are generous but Glue can burst. Add exponential-backoff retries. For the demo, one Glue DPU making sequential calls is fine (~20 rows/sec × 60 min = 72K events per DPU-hour).
4. **pgvector index tuning** — for < 1M vectors, defaults are fine (`m=16, ef_construction=64`). Don't over-tune.
5. **RDS in a VPC** — Lambda needs VPC config to reach it. Adds cold start. Use **RDS Data API** or an interface endpoint to keep Lambdas out of VPC.
6. **DMS ↔ Kinesis** for a t4g.micro-scale RDS is overkill — feel free to skip DMS on Weekend 1 and just have Lambda query RDS directly. It's there to check the Domain 1 skill box.
7. **NAT Gateway costs** — $32/mo just for existing. Design your VPC with VPC endpoints from Day 1 to avoid the NAT.

---

## The README Template (repo landing page)

```markdown
# OSS Community Intelligence Platform

A serverless data-engineering platform on AWS that ingests **GitHub Archive** and live GitHub events for the top ML/data-infrastructure OSS projects, enriches issues/PRs with Amazon Bedrock LLMs, and provides semantic search + trend analytics.

Deployed 100% via AWS CDK. Runs at ~$5/month idle, ~$20 total to build. Covers all four domains of the AWS DEA-C01 certification.

![architecture](docs/architecture.png)

## What it does

- **Ingests** GitHub Archive (batch) + GitHub Events API (real-time) + a Postgres CDC feed for tracked-repo metadata.
- **Enriches** every issue / PR / discussion with Bedrock (Nova Lite classification + Titan embeddings): category, severity, sentiment, summary, 1024-dim vector.
- **Stores** in a governed Iceberg lakehouse (S3 Tables + Glue Catalog + Lake Formation), with RDS pgvector (HNSW) for semantic search and Redshift Serverless for BI.
- **Serves** a semantic-search API (API Gateway + Lambda), a Streamlit dashboard, and Bedrock-Agent-powered NL Q&A over the lake.
- **Governs** with Lake Formation LF-tags, Macie PII discovery, KMS CMKs, VPC endpoints, IAM least-privilege.
- **Observes** via CloudWatch dashboards, alarms, Logs Insights, and CloudTrail.

## Services used

S3 · Kinesis Data Streams · Kinesis Firehose · DMS · RDS PostgreSQL + pgvector · DynamoDB · Redshift Serverless · Glue (Jobs, Catalog, Crawler, Data Quality) · Athena · Lambda · Step Functions · EventBridge · Bedrock (Nova Lite + Titan Embeddings) · API Gateway · Lake Formation · Macie · IAM · KMS · Secrets Manager · VPC endpoints · CloudWatch · CloudTrail · SNS · CDK (Python)

## Interesting technical choices

- **S3 Tables** (managed Iceberg) instead of self-managed — automatic compaction saves ~15% of eng time on maintenance.
- **RDS pgvector over Aurora Serverless v2** — 3× cheaper for the demo scale, same HNSW recall.
- **Bedrock Nova Lite** for classification — 50× cheaper than Claude Opus, sufficient accuracy on structured extraction.
- **VPC endpoints, no NAT gateway** — saves $32/month and forces private-only egress.
- **Athena partition projection** instead of Glue crawler — no crawler cost, sub-second query latency.
- **Batching 20 rows per Bedrock call** in Glue — 15× cheaper than row-by-row invocation.

## Try it

git clone …
cd oss-intel && pip install -r requirements.txt
cdk bootstrap && cdk deploy --all

## Demo

[90-second video](https://loom.com/…)

## Cost

~$5/month idle. ~$20 total to build across 4 weekends. `cdk destroy` cleans everything.

## What I learned

- Bedrock rate-limit patterns at Glue-job scale.
- Iceberg vs S3 Tables trade-off (managed compaction vs cost predictability).
- pgvector HNSW index tuning for ~500k vectors.
- Lake Formation LF-tag design that scales to 100+ tables.
- Cost engineering — cut $60/month of default AWS charges by choosing VPC endpoints, Nova Lite, and Redshift Serverless.

## Author

Manas — [LinkedIn](…) — [Portfolio](…)
```

---

## The Interview Story (60-second pitch)

> "I built a serverless GenAI data platform on AWS that ingests the GitHub Archive event stream — every public GitHub push, PR, issue — for the top 10 ML and data-infrastructure open-source projects like PyTorch, Spark, and LangChain. Every issue and PR gets enriched by Bedrock — Nova Lite for classification and severity scoring, Titan for 1024-dimensional embeddings — and lands in an Iceberg lakehouse on S3 Tables with Lake Formation column-level PII masking. The embeddings power a semantic-search API on RDS pgvector with HNSW indexing, and Redshift Serverless feeds a Streamlit dashboard for trend analysis. It's 100% AWS CDK, uses VPC endpoints instead of a NAT gateway to cut cost, and runs at about $5 a month idle. The whole thing covers every domain of the AWS Data Engineer certification. My favorite piece was cost engineering — I cut the naïve architecture from about $80/month down to $5 by choosing Nova Lite over Claude for classification, RDS over Aurora Serverless v2 for pgvector, and Athena partition projection instead of Glue crawlers."

That's the story. Practice it aloud until it flows.

---

## Optional Extensions (weekends 5+)

- **Bedrock Agent** with tool use — "show me the biggest PyTorch issues from the last 24 hours" in natural language.
- **Trend detection** — anomaly-detection Lambda flags spikes in `category=bug severity=critical` per repo → SNS alert.
- **Multi-region** — Global Table for DynamoDB hot cache; S3 CRR for lake.
- **Vector index comparison** — build the same corpus in OpenSearch Serverless and benchmark cost/recall vs pgvector. Blog post material.
- **A/B Bedrock models** — compare Nova Lite vs Claude Haiku on classification accuracy. Log results.
- **dbt + Redshift** — analyst-friendly transformations layer.
- **Grafana dashboard** on top of CloudWatch metrics — "operations mode" view for the pipeline itself.

---

## References

- [GitHub Archive project](https://www.gharchive.org/) — the data source.
- [GitHub Events API docs](https://docs.github.com/en/rest/activity/events) — for real-time polling.
- [AWS CDK Python API](https://docs.aws.amazon.com/cdk/api/v2/python/) — infra as code.
- [Bedrock Runtime API](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_runtime_InvokeModel.html) — enrichment.
- [S3 Tables docs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables.html) — managed Iceberg.
- [pgvector docs](https://github.com/pgvector/pgvector) — vector search on Postgres.
- [Streamlit docs](https://docs.streamlit.io/) — for the dashboard.
- Companion: [System Design for DE](../15-system-design-de.md) — the DE framework this project follows.

---

*This is a project you can actually be proud of and that recruiters at NVIDIA / Databricks / HuggingFace / Snowflake will recognize instantly as "the shape of platform we build internally." Start with Weekend 1 alongside your Domain 1 study.*
