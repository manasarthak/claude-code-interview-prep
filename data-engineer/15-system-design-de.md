# System Design for Data Engineers — How to Pass the Hardest Round

**Phase:** 3 (Differentiator)
**Difficulty progression:** Beginner → Intermediate → Advanced
**Last updated:** April 24, 2026
**Related:** [Warehouses & Lakehouses](./03-warehouses-lakes-lakehouse.md) · [ETL vs ELT](./04-etl-vs-elt.md) · [Streaming](./06-batch-vs-streaming.md) · [AWS Data Stack](./12-aws-data-stack.md) · [Governance](./14-governance-lineage-observability.md)

---

## Why This File Exists

Senior and staff DE interviews always include a system design round, and this is where most candidates with strong technical fundamentals fail. The reason isn't lack of knowledge — it's lack of a *framework*. People who know everything about [Spark](./07-spark-fundamentals.md), [Airflow](./05-orchestration-airflow.md), and [Iceberg](./08-table-formats.md) still bomb the round because they jump straight to "I'd use Kafka and Spark" without ever asking what the requirements are.

This file gives you the framework, the trade-off vocabulary, and four worked examples of the canonical DE system design questions.

---

## BEGINNER — The Framework Every Senior DE Should Internalize

### The Six-Step Framework (use this in every round)

```
1. Clarify requirements   — what, why, who, how much
2. Define interfaces      — sources, sinks, SLAs
3. Sketch the high-level  — boxes & arrows, no specifics
4. Drill into components  — each box becomes a real system
5. Discuss trade-offs     — what breaks, what scales, what costs
6. Operate it             — monitoring, recovery, on-call
```

**Most candidates skip steps 1, 2, 5, 6.** Those are the steps that separate senior from staff.

### Step 1: Clarify Requirements (spend 5–10 minutes here)

Always ask these questions explicitly. Interviewers grade you on whether you ask, not on whether you guess right.

**Functional questions:**
- What's the use case? (analytics dashboard, ML feature, ops alerting, regulatory report)
- Who's the consumer? (analyst running ad-hoc SQL, app reading via API, ML model)
- What does "done" look like? (a fact table, a Kafka topic, a Looker dashboard)

**Non-functional questions:**
- **Volume**: rows/day, GB/day, peak QPS
- **Velocity**: real-time (seconds), near-real-time (minutes), batch (hours/daily)
- **Variety**: structured, semi-structured (JSON), unstructured
- **Veracity**: how clean? How critical that it's correct? (financial vs. clickstream)
- **SLA / SLO**: freshness target, availability target, accuracy tolerance
- **Cost ceiling**: do we have a budget?
- **Compliance**: PII? GDPR? HIPAA? See [Governance](./14-governance-lineage-observability.md).

**Trick:** if the interviewer is vague, write your assumed numbers on the board explicitly. "I'm assuming 10M events/day, 1KB each = 10GB/day = 3.65TB/year." This is *not* showing off; it's how seniors derisk a design.

### Step 2: Define Interfaces and SLAs

Before drawing arrows, define the contract:

| Boundary | Question | Why it matters |
|---|---|---|
| Source | Push or pull? | Drives whether you need a queue |
| Source | Schema stable? | Drives [contracts](./11-data-quality-contracts.md) and [CDC](./09-cdc.md) approach |
| Sink | Read pattern? | Point lookups → KV store; analytics → columnar |
| Sink | Latency tolerance? | Streaming vs. batch decision |
| Sink | Mutation semantics? | At-least-once vs. exactly-once vs. idempotent |

### Step 3: Sketch the High-Level Architecture

The five-box default starting point for any DE system:

```
[ Source ] → [ Ingest ] → [ Storage ] → [ Compute ] → [ Serve ]
                              ↓
                        [ Catalog / Governance ]
                              ↓
                        [ Observability / Lineage ]
```

Almost every DE system maps to this. Then you specialize each box.

### Step 4: Specialize Each Box (the choices that win the interview)

| Box | Common choices | Decision driver |
|---|---|---|
| Source | App DB, SaaS API, files, IoT, clickstream | Already given by problem |
| Ingest | [Kafka](./06-batch-vs-streaming.md), Kinesis, Fivetran, [Debezium](./09-cdc.md), S3 drop zones | Velocity + back-pressure tolerance |
| Storage | [S3 + Iceberg/Delta](./08-table-formats.md), [Redshift](./12-aws-data-stack.md), Snowflake, BigQuery | Read pattern + cost |
| Compute | [Spark](./07-spark-fundamentals.md), Flink, [dbt](./10-dbt.md), Athena, Redshift | Volume + latency |
| Serve | OLAP DB, KV store, API, BI tool | Consumer profile |
| Orchestration | [Airflow](./05-orchestration-airflow.md), Dagster, Step Functions | Scheduling complexity |
| Catalog | Glue, Unity, DataHub, Atlan | Discovery + lineage needs |

### Step 5: Discuss Trade-offs (this is the senior signal)

Every choice you make should be paired with the trade-off. Don't just say "I'll use Kafka." Say:

> *"I'll use Kafka because we need replay and multiple consumers, accepting the operational cost of ZooKeeper/KRaft and the complexity of consumer offset management. If the team didn't have Kafka expertise, I'd use Kinesis to trade tunable retention for managed simplicity."*

That sentence — choice + reason + acknowledged cost + alternative — is what staff-level looks like.

### Step 6: Operate It (the round most people forget)

Always end by walking through:
- **How does this fail?** (broker dies, S3 throttles, schema drift, late data)
- **How do you detect failure?** (alerts on freshness SLO, lag metrics, [data quality checks](./11-data-quality-contracts.md))
- **How do you recover?** (replay from offset, reprocess partition, [backfill](./04-etl-vs-elt.md#backfills))
- **Who's on-call and what do they look at?** (runbook, dashboards, [observability](./14-governance-lineage-observability.md))

---

## INTERMEDIATE — Trade-off Vocabulary You Must Speak Fluently

### The Eight Big Trade-offs in DE System Design

These come up in every senior round. Memorize the framing.

| Trade-off | Lean A | Lean B | Driver |
|---|---|---|---|
| Batch vs Streaming | Daily batch | Streaming | Latency requirement |
| Push vs Pull ingestion | Pull (polling) | Push (CDC, webhooks) | Source ownership + freshness |
| ETL vs ELT | ETL | ELT | Compute location + governance |
| Lambda vs Kappa | Lambda (two paths) | Kappa (one stream path) | Reprocessing simplicity vs latency |
| Lakehouse vs Warehouse | Warehouse | Lakehouse | Cost at scale + flexibility |
| Strong vs Eventual consistency | Strong | Eventual | Use case (financial vs analytics) |
| At-least-once vs Exactly-once | At-least-once + idempotent | Exactly-once | Cost + downstream complexity |
| Schema-on-read vs Schema-on-write | Read | Write | Source variety + contract maturity |

### Capacity Estimation Cheat Sheet

You'll need this in every round. Practice estimating without a calculator.

```
1KB event ×   1M/day    =   1GB/day   = ~365GB/yr
1KB event ×  10M/day    =  10GB/day   = ~3.6TB/yr
1KB event × 100M/day    = 100GB/day   = ~36TB/yr
1KB event ×   1B/day    =   1TB/day   = ~365TB/yr
```

Common multipliers:
- **Compression**: Parquet+Snappy ≈ 5–10× smaller than raw JSON
- **Replication**: 3× for distributed file systems (S3 already handles)
- **Indexes / sort keys / Z-order**: assume +20–30% storage
- **Iceberg/Delta metadata + history**: +5–15% with default retention

### Latency Budgets (back-of-envelope)

| Layer | Typical latency |
|---|---|
| Network in same AZ | <1ms |
| S3 PUT/GET | 50–200ms |
| Kafka produce-consume | 5–20ms |
| Kinesis put-get | 70ms–1s |
| Redshift / Snowflake simple query | 1–5s |
| Athena over Iceberg | 3–30s |
| Spark batch job | minutes |
| Airflow DAG end-to-end | 10s of minutes to hours |

Map the user requirement to these and you can justify your latency choices.

### Reliability Patterns You'll Be Asked About

- **Idempotency** — every write keyed so re-runs don't duplicate. See [ETL vs ELT](./04-etl-vs-elt.md#idempotency).
- **Dead Letter Queues** — bad messages go aside, don't kill the pipeline.
- **Backfills** — partitioned writes so you can reprocess one day in isolation.
- **Replay** — Kafka retention or S3 raw-zone preserves ability to rebuild.
- **Circuit breakers** — auto-pause downstream if a [data quality](./11-data-quality-contracts.md) check fails.
- **Bulkheads** — separate pools/queues so one bad tenant can't starve others.
- **Outbox pattern** — for [transactional CDC](./09-cdc.md) without dual-write risk.

---

## ADVANCED — Worked Examples (4 Canonical DE Designs)

These four cover ~80% of what you'll be asked. Practice them out loud.

---

### Worked Example 1: Design a Daily Analytics Pipeline (Stripe-like Payments → Dashboard)

> *"Design a pipeline that ingests payment events from a Postgres OLTP database into a warehouse so analysts can run revenue dashboards by morning."*

#### Step 1: Clarify
- Volume: 5M payments/day → ~5GB/day raw
- Latency SLA: refresh by 7am — 1-day lag acceptable
- Consumers: analysts (SQL), exec dashboard
- PII: customer email, last-4 of card → masked in analytics layer
- History: 5 years of historical data

#### Step 2: Architecture

```
Postgres (OLTP)
    │
    │   [DMS / Debezium full-load + CDC] ← see ./09-cdc.md
    ▼
S3 raw zone (Parquet, partitioned by ingest_date)
    │
    │   [Spark/Glue: dedupe, mask PII, conform schema]
    ▼
S3 silver zone (Iceberg, partitioned by event_date)
    │
    │   [dbt Core on Redshift / Athena]
    ▼
Gold marts (fct_payment, dim_customer, dim_merchant)
    │
    ▼
Looker / QuickSight dashboards
```

Orchestration: [Airflow](./05-orchestration-airflow.md) DAG running 2am UTC daily.

#### Step 3: Trade-offs to Surface
- **Why DMS over Debezium?** Managed, simpler. Trade-off: less flexible than Debezium for custom transformations.
- **Why daily batch vs streaming?** SLA is daily; streaming would burn cost without business value.
- **Why Iceberg, not just Parquet on S3?** ACID merges from CDC, schema evolution, time travel for [audits](./14-governance-lineage-observability.md).
- **Why Redshift over BigQuery?** AWS-only stack; data already on S3; Redshift Spectrum reads Iceberg directly.
- **Why dbt?** Versioned SQL, testable, easy for analysts. See [dbt](./10-dbt.md).

#### Step 4: Operate It
- Freshness SLO: silver layer ready by 4am, gold by 6am
- Alerts: dbt test failures page on-call, freshness checks every hour
- Backfill plan: re-run a single date partition without touching others
- Cost: ~$2k/mo on Redshift ra3.4xlarge ×2 + S3 + Glue

---

### Worked Example 2: Design a Real-Time Fraud Detection Pipeline

> *"Detect fraudulent transactions within 1 second so we can decline them before authorization completes."*

#### Step 1: Clarify
- Volume: 10K TPS peak, 200M txns/day
- Latency SLA: p99 < 500ms decision
- Consumers: payment service (synchronous decision endpoint), fraud analysts (post-hoc review)
- Model: pre-trained, sub-100ms inference

#### Step 2: Architecture

```
Payment service ──HTTP──▶ Fraud API ──┐
                                       │
                                       ▼
                          Feature service (Redis cluster)
                              ▲           ▲
                              │           │
                  ┌───────────┘           └─────────────┐
                  │                                      │
       Online features (last 5min)              Offline features
       Flink streaming (Kinesis)                Spark daily (S3 + Iceberg)
                  ▲                                      ▲
                  │                                      │
       Kinesis (transactions topic) ◀─── Payment service produces here
                  │
                  ▼
       S3 raw archive (replay + training)
```

#### Step 3: Trade-offs
- **Online + offline feature store**: short-window features need streaming; long-history features live in batch. Lambda-style.
- **Why Flink over Spark Streaming?** Lower latency, better state mgmt for windowed features.
- **Why Redis as serving layer?** Sub-ms reads. Trade-off: cost + memory ceiling.
- **Exactly-once?** No — we accept at-least-once with idempotent writes (transaction_id key) because the model decision is read-only.
- **Cold start**: when Redis is empty (e.g. after restart), serve a default risk score and warn ops.

#### Step 4: Operate It
- p99 latency dashboard at every hop
- Replay from Kinesis on Flink job restart
- Shadow mode for new model versions before swap
- Audit trail: every decision logged with feature snapshot for [compliance](./14-governance-lineage-observability.md)

---

### Worked Example 3: Design a CDC Pipeline from 100 Microservice Databases to a Lakehouse

> *"We have 100 Postgres databases across 100 microservices. Land everything into a unified analytics lakehouse with under 5-minute lag."*

#### Step 1: Clarify
- Volume: ~50GB/day write across all 100 DBs
- Latency: 5 min freshness
- Schema drift: yes — services deploy independently
- Consumers: cross-service analytics, fraud team, ML
- Each service team owns its source schema; central platform team owns the lake

#### Step 2: Architecture

```
100× Postgres
    │  (logical replication, one slot per DB)
    ▼
Debezium Connect cluster (on EKS) ──▶ Schema Registry
    │
    ▼
MSK / Kafka (topic-per-table convention)
    │
    ▼
Kafka Connect S3 sink (or Flink) writing to Iceberg
    │
    ▼
S3 + Iceberg silver tables (MERGE-on-read)
    │
    ▼
[hourly Spark compaction job + GDPR delete job]
    │
    ▼
Athena / Redshift Spectrum for consumers
```

#### Step 3: Trade-offs
- **Why Debezium, not DMS?** 100 DBs is at the limit of DMS scaling; Debezium gives more control + a single Kafka pipeline.
- **Schema Registry mandatory** here — services WILL drift. See [contracts](./11-data-quality-contracts.md).
- **Iceberg MoR vs CoW?** MoR — write amplification matters at this volume; readers tolerate delete files.
- **One topic per table or one per service?** One per table → simpler sink config, more topics.
- **Failure domain**: blast radius. Per-DB connector so one failing service doesn't halt others.

#### Step 4: Operate It
- Per-source dashboard (Debezium connector status, Kafka lag, Iceberg snapshot age)
- Auto-pause sink when [data contract](./11-data-quality-contracts.md) violated; alert source team
- Compaction every 4 hours; expire snapshots > 7 days
- GDPR delete: weekly job that runs `DELETE FROM ... WHERE user_id IN (deletion_list)` on every Iceberg table — see [Governance](./14-governance-lineage-observability.md#gdpr-delete-pattern)

---

### Worked Example 4: Design a Clickstream Pipeline at 1B Events/Day

> *"Ingest 1B clickstream events per day, surface near-real-time funnel metrics, and feed an ML training pipeline."*

#### Step 1: Clarify
- Volume: 1B events/day = ~12K events/sec average, 50K peak
- Avg event size: 2KB → 2TB/day raw, ~250GB/day Parquet+Snappy
- Latency: dashboards within 5 min, ML training daily
- Retention: 2 years hot, 5 years cold

#### Step 2: Architecture

```
SDKs (web, mobile)
    │  HTTPS POST to ingest endpoint
    ▼
Edge collector (NGINX + Lua → Kinesis) ──▶ Schema validation
    │
    ▼
Kinesis Data Streams (sharded by user_id)
    │
    ├──▶ Kinesis Firehose ──▶ S3 raw (compressed JSON, 5-min files)
    │
    ├──▶ Flink (sessionization, funnel events) ──▶ Kinesis "metrics" topic
    │                                                    │
    │                                                    ▼
    │                                              ClickHouse (real-time dashboards)
    │
    └──▶ Spark Streaming (write to Iceberg silver, partitioned by event_date+event_type)

Daily batch:
  S3 raw (ingest_date partition) ──▶ Spark ETL ──▶ Iceberg gold ──▶ Feature store / training
```

#### Step 3: Trade-offs
- **Kinesis vs Kafka?** Managed; we don't have a Kafka team. Trade: 70ms vs 10ms latency, sufficient.
- **Why ClickHouse for dashboards?** Sub-second OLAP on streaming data. Trade-off: ops complexity vs Athena.
- **Why dual-write to Iceberg AND ClickHouse?** ClickHouse for hot ops dashboards, Iceberg for ML + ad-hoc.
- **File sizing**: target 256MB Parquet files; 5-min Firehose files are too small → hourly compaction job. See [Partitioning & Performance](./13-partitioning-performance.md).
- **Sharding by user_id**: preserves session ordering per user. Trade-off: hot users create skew → fall back to user_id+session_id hash if needed.

#### Step 4: Operate It
- Kinesis IteratorAge metric (>5 min = page)
- Flink checkpoint health, late-event watermark dashboard
- Compaction job ratio (small file count) tracked weekly
- Cost: ~$15–20k/mo on this scale (Kinesis dominates)

---

## How to Communicate During the Round

### The "Layered Reveal" Technique

Strong candidates don't dump the full architecture in minute 5. They reveal in layers:

1. **Minute 0–5**: clarify, write requirements
2. **Minute 5–10**: high-level boxes (the five-box default)
3. **Minute 10–25**: drill into each box, justify choices, draw arrows
4. **Minute 25–40**: trade-offs, scaling, failure modes
5. **Minute 40–50**: operations, monitoring, on-call
6. **Minute 50–55**: "what would you do differently with more time?"

### Phrases That Signal Senior

- *"Let me write down the assumptions I'm making."*
- *"There are three reasonable choices here. I'd pick X because of Y, accepting cost Z."*
- *"This will fail when..."*
- *"The blast radius of this failure is..."*
- *"If I were operating this on-call, I'd want to see..."*
- *"I'd start simple with X and only move to Y if [trigger]."*

### Phrases That Signal Junior (avoid)

- "I'd use Kafka." (no reason given)
- "We'd just use Spark." (no operational context)
- "Then we put it in the warehouse." (which one? why?)
- "We'd handle that with a script." (where does the script run? who owns it?)

---

## Worked Example: A Full Round Transcript (Condensed)

> **Interviewer:** Design a system to detect duplicate user sign-ups across our 50 microservices in real time.

**You:** A few clarifying questions before I start. *(writes them on the board)*
1. By "duplicate," do you mean same email, same phone, fuzzy name match, or device fingerprint?
2. What's the action — block at sign-up, alert fraud team, or both?
3. Latency budget — must we decide before sign-up completes, or can we be eventually consistent within minutes?
4. Volume — sign-ups/day across all services?

**Interviewer:** Same email or phone, both block + alert, decide before sign-up completes (≤500ms), 100k sign-ups/day peak 5/sec.

**You:** *(writes)* So 500ms p99, 5/sec peak is small — single-region serving is fine. Let me sketch the high-level.

```
Sign-up service ──▶ Identity Resolution API ──▶ [yes/no, with reason]
                            │
                            │ reads from
                            ▼
                  Identity Index (DynamoDB on email_hash, phone_hash)
                            ▲
                            │ writes
                            │
   Kafka "user_created" topic ── all 50 services produce here
                            ▲
                            │
                  Service DBs (CDC via Debezium, fallback)
```

**You:** Trade-offs I'm making:
- **DynamoDB** for the index because we need point reads at p99 < 50ms; trade-off: harder to do fuzzy matching, but spec says exact email/phone.
- **Kafka pub-sub** so all 50 services can publish without coupling to the resolution API; trade-off: at-least-once → I make the index write idempotent on (email_hash, user_id).
- **Hash, don't store raw email/phone** — see [Governance](./14-governance-lineage-observability.md), keeps PII off the index.
- **Synchronous lookup with async write**: API reads instantly; the new sign-up's row is written async after success.

**You:** Failure modes:
- DynamoDB throttle → degrade to "allow with flag"; alert.
- Kafka lag spike → not blocking; alert when lag > 5min.
- Identity API down → sign-up service falls back to default-allow + flag for review.

**Interviewer:** What if a sign-up arrives before its CDC event from another service has been indexed?

**You:** That's a TOCTOU race. Three options: (a) write-through from the resolution API (we own the source of truth), (b) require services to call resolution API as a precondition rather than rely on CDC, (c) accept eventual consistency and re-check periodically. I'd push for (b) — the indirection of CDC isn't necessary because all 50 services already know the moment they create a user. CDC stays as a backfill / disaster recovery path only.

That kind of layered, traded-off, operationally-grounded answer is what differentiates a senior from a junior.

---

## Interview-Ready Cheat Sheet

### Top 12 system design Q&As

**Q: What's the first thing you do when given a system design problem?**
A: Spend 5–10 minutes clarifying functional and non-functional requirements out loud, writing assumptions on the board.

**Q: How do you decide batch vs streaming?**
A: Driven by SLA. If the consumer's latency tolerance is hours → batch. Minutes → micro-batch. Seconds → streaming. Default to batch for cost reasons unless requirements force streaming.

**Q: When would you choose Kafka over Kinesis?**
A: Kafka when you need <10ms latency, multi-consumer replay over weeks, or you already have ops expertise. Kinesis when you want managed simplicity and your team is AWS-native.

**Q: Lambda vs Kappa architecture?**
A: Kappa (single streaming path, batch reprocess by replay) is simpler for new builds. Lambda (separate batch + speed) lingers when batch is tuned, well-tested, and you don't want to risk re-implementing.

**Q: How do you size compute for a Spark job?**
A: Start from data volume → required parallelism (data ÷ partition size of ~128MB) → executors. Validate with first run, tune with [Spark UI](./07-spark-fundamentals.md).

**Q: What's your strategy for backfills?**
A: Partition all writes by ingest_date or event_date so a single day can be reprocessed in isolation. Make every job idempotent on (key, partition). Have a separate backfill DAG that doesn't fight the daily schedule.

**Q: How do you handle late data in streaming?**
A: Watermarks + allowed lateness. Buffer windows for the lateness budget; events past the budget go to a side-output for batch reconciliation. See [Streaming](./06-batch-vs-streaming.md).

**Q: How do you guarantee exactly-once?**
A: Three layers: source offsets committed only after sink durably writes, sink is idempotent on a key, and the consumer manages transactions atomically. In practice most systems do at-least-once + idempotent sinks.

**Q: How do you size storage cost?**
A: rows/day × bytes/row × compression ratio × replication × retention years × $/GB-month. Add 20–30% for indexes/sort/Z-order. Then double it because reality is messy.

**Q: How do you pick between Iceberg, Delta, and Hudi?**
A: See [Table Formats](./08-table-formats.md). Iceberg if you're cloud-agnostic / multi-engine. Delta if you're Databricks-native. Hudi for record-level upsert-heavy CDC. AWS-default today: Iceberg via Glue.

**Q: How do you design for compliance from day one?**
A: PII tagging in the catalog, column-level masking, audit logs of access, partitioning that lets you do GDPR deletes in O(partition) not O(table). See [Governance](./14-governance-lineage-observability.md).

**Q: What does "operate it" mean in your design?**
A: Freshness/quality SLOs published, alerts on lag/failure, runbook for top-3 failure modes, dashboards every consumer can self-serve, on-call rotation.

### Quick Trade-off Pairs

| If they say... | You say... |
|---|---|
| "make it faster" | "trade cost + complexity, here's how" |
| "make it cheaper" | "trade latency or freshness, here's how" |
| "simpler" | "trade flexibility or feature set" |
| "more reliable" | "trade cost (replication) or latency (replication waits)" |
| "use less infra" | "go managed; trade lock-in for ops cost" |

### Red Flags to Avoid

- Designing without writing requirements first
- Picking tools without naming the trade-off
- Not discussing failure modes
- Forgetting [data quality](./11-data-quality-contracts.md) and [governance](./14-governance-lineage-observability.md)
- Hand-waving capacity ("yeah it'll scale")
- Skipping monitoring + on-call
- Re-using a memorized architecture without adapting to requirements

---

## How to Practice

1. **Pick one of the four worked examples above. Cover the answer. Solve it cold in 45 minutes.** Compare your answer to the worked example.
2. **Re-do the same problem with one constraint changed** (e.g. 100× volume, 1/10 budget, no streaming allowed). Notice which boxes change.
3. **Take a real system you've built and turn it into a 45-min interview answer.** Practice telling its story with the six-step framework.
4. **Read engineering blogs** (Uber, Airbnb, Stripe, Netflix, Lyft) — each post is a worked system design. Map them to the framework.
5. **Pair-practice** — find a peer or use Pramp/Hello Interview. The bar is "does the listener follow the trade-offs?"

---

## Resources & Links

- *Designing Data-Intensive Applications* — Martin Kleppmann ([book](https://dataintensive.net/)) — the bible for this round.
- [Hello Interview — Data Engineering tracks](https://www.hellointerview.com/) — paid but high-quality DE system design walk-throughs.
- [Uber Engineering Blog](https://www.uber.com/en-IN/blog/engineering/data/) — Michelangelo, Uber Data Lake, real worked-out designs.
- [Airbnb Data Engineering posts](https://medium.com/airbnb-engineering/tagged/data-engineering)
- [Netflix TechBlog — Data](https://netflixtechblog.com/tagged/data)
- [Stripe Engineering — Data Infrastructure](https://stripe.com/blog/engineering)
- [The Big Book of Data Engineering — Databricks](https://www.databricks.com/resources/ebook/big-book-of-data-engineering) — free PDF.
- [Data Engineering Wiki](https://dataengineering.wiki/) — community-maintained reference.
- [Awesome Data Engineering](https://github.com/igorbarinov/awesome-data-engineering) — curated tooling list.
- [r/dataengineering interview thread compilation](https://www.reddit.com/r/dataengineering/) — recent real questions.

---

*This is the last topic file in the data-engineer track. From here, work through the [projects/](../projects/) folder to build portfolio pieces, and revisit the [README](./README.md) for the certification roadmap.*
