# AWS Data Stack — The Services That Show Up in Every Posting

**Phase:** 1 (Foundations)
**Difficulty progression:** Beginner → Intermediate → Advanced
**Last updated:** April 24, 2026
**Related:** [Warehouses & Lakehouses](./03-warehouses-lakes-lakehouse.md) · [Spark Fundamentals](./07-spark-fundamentals.md) · [Batch vs Streaming](./06-batch-vs-streaming.md) · [Open Table Formats](./08-table-formats.md) · [Orchestration & Airflow](./05-orchestration-airflow.md)

---

## Why This File

For AWS-primary interviews (and the [DEA-C01 cert](./README.md#certifications-that-actually-move-the-needle)), you need to be conversant in S3, Glue, EMR, Redshift, Athena, Kinesis/MSK, Lake Formation, IAM, and a handful of supporting services. This file is the one-stop tour with mental models for each — not a reproduction of AWS docs.

If you're reading this in interview prep and don't have hands-on with at least 3-4 of these, that's the gap to close before anything else.

---

## BEGINNER — The Mental Map

### The AWS Data Stack at a Glance

```
[Sources]
   │
   ├── Apps / SaaS  → AppFlow / Lambda / custom
   ├── Databases   → DMS / Glue connectors
   └── Streams     → Kinesis Data Streams / MSK / Firehose
                      │
                      ▼
[Storage]                            [Compute]                    [Catalog]
   ├── S3 (lake, all formats)        ├── EMR / EMR Serverless     ├── Glue Data Catalog
   ├── Iceberg / Delta on S3         ├── Glue (managed Spark)     └── Lake Formation
   ├── Redshift Managed Storage      ├── Athena (Trino on S3)        (governance)
   └── DynamoDB / Aurora             ├── Redshift / Spectrum
                                      ├── Lambda
                                      └── Managed Flink (KDA)
                                              │
                                              ▼
                                  [Orchestration]
                                  ├── MWAA (Managed Airflow)
                                  ├── Step Functions
                                  └── EventBridge

                                  [Consumption]
                                  ├── QuickSight
                                  ├── SageMaker (ML)
                                  └── External BI (Tableau, Looker, ...)
```

### The Five You Must Know Cold

| Service | Role | Interview must-know |
|---|---|---|
| **S3** | Object storage, the lake | Storage classes, lifecycle, intelligent-tiering, server-side encryption, prefix design |
| **Glue Data Catalog** | Metadata catalog | Tables, partitions, crawlers, integration with Athena/EMR/Redshift |
| **Athena** | Serverless SQL on S3 | Partition projection, Iceberg, federation, costs (per TB scanned) |
| **Glue (ETL)** | Managed Spark for ETL | Jobs, triggers, bookmarks, DPUs, Glue Studio, Glue Data Quality |
| **Redshift** | Cloud warehouse | RA3 vs DC2, distribution/sort keys, Spectrum, materialized views, AQUA |

You can build a real pipeline with just these five. Everything else extends from there.

### S3 — More Than "A Bucket"

S3 underpins every AWS data architecture. Things that matter for DE:

| Topic | Why it matters |
|---|---|
| **Storage classes** | S3 Standard, Standard-IA, Intelligent-Tiering, Glacier (Instant/Flexible/Deep). Costs differ 10×+. |
| **Lifecycle policies** | Auto-transition cold data to IA/Glacier; expire old versions |
| **Versioning** | Protects against accidental deletes; doubles cost |
| **Server-side encryption** | SSE-S3, SSE-KMS, SSE-C — KMS for managed keys |
| **Prefix design** | Hot prefixes can throttle (3500 PUT / 5500 GET req/sec per prefix); randomize for very high write rates |
| **Intelligent-Tiering** | Often the cost-effective default for unknown access patterns |
| **S3 Tables** (2024+) | Managed Iceberg tables in S3 — AWS's bet on open table formats |

**Common interview probe:** "How would you design S3 prefixes for a 100M event/day stream?" Answer: partition by event_date and (hashed) shard prefix to avoid hot keys; use Iceberg / S3 Tables to manage metadata.

---

## INTERMEDIATE — The Compute Layer

### Glue (the umbrella name, but really three things)

| Component | What it is |
|---|---|
| **Glue Data Catalog** | Metadata service — tables, partitions, schemas. Used by Athena, EMR, Redshift Spectrum, Lake Formation |
| **Glue Crawlers** | Inspect S3 / DBs and infer schemas → write into Catalog |
| **Glue ETL Jobs** | Managed Spark jobs (PySpark, Scala) running on AWS-provided cluster |
| **Glue Studio / DataBrew** | Visual ETL designers (less popular for engineering teams) |
| **Glue Schema Registry** | Avro/JSON schemas for Kafka/Kinesis |
| **Glue Data Quality** | DQ rules engine (DDQDL) integrated into pipelines |

**Glue ETL specifics:**
- Pricing per **DPU-hour** (1 DPU = 4 vCPU + 16 GB)
- **Glue 4.0+** runs Spark 3.3+; supports Iceberg/Delta/Hudi
- **Bookmarks** track which input files were processed (incremental loads)
- **G.1X / G.2X / G.4X** — DPU sizes for different workloads
- **Glue Streaming** — micro-batch streaming (Spark Structured Streaming)
- Slower start-up than EMR (~60-90s) but no cluster ops

**When Glue ETL fits:** ad-hoc ETL, low-medium volume, want zero cluster ops, OK with slightly higher per-DPU price than self-managed EMR.

### EMR — When You Need Real Spark Power

EMR is AWS-managed clusters running Spark, Hadoop, Hive, Presto, Trino, Flink, etc.

| Mode | When |
|---|---|
| **EMR on EC2** | Long-lived cluster you fully control; cheapest for sustained workloads |
| **EMR on EKS** | Spark on your existing K8s cluster |
| **EMR Serverless** | Pay-per-job; great for spiky / unpredictable workloads |

**Mid-level interview points:**
- EMR is more cost-effective than Glue for sustained, large-scale Spark
- EMR has the full Spark feature set (custom configurations, advanced tuning); Glue abstracts some of that
- EMR Serverless removes cluster sizing — pay for what you use
- Spot instances + Auto Scaling on EMR can drop costs 50–70%

### Athena — Serverless SQL on S3

Athena is Trino/Presto fronted by AWS, queries S3 (and federated sources) using the Glue Catalog.

**Strengths:**
- Serverless — no clusters
- Pays per TB scanned
- Reads any format Trino can read (Parquet, Iceberg, Delta)
- Federated queries to RDS, Redshift, DynamoDB, ElastiCache, etc. (Athena Federated Query)

**Cost levers:**
- **Partition projection** on Hive-style partitions → no Glue Catalog calls per query
- Scan only needed columns (Parquet)
- Filter on partition columns
- Query Iceberg tables for metadata-driven pruning
- Use **Athena workgroups** with per-query cost limits

**When it fits:** ad-hoc analytics on S3 lake; lightweight query tier in front of a lakehouse; per-query unpredictable loads. Not for sustained dashboard traffic — Redshift / Snowflake will be cheaper there.

### Redshift — The AWS-Native Warehouse

| Variant | Notes |
|---|---|
| **DC2 / DS2 (legacy)** | Storage tied to compute; resize is painful; avoid for new builds |
| **RA3** | Storage decoupled (Redshift Managed Storage on S3); resize fast; the default for modern Redshift |
| **Redshift Serverless** | No cluster; auto-scaling; pay per RPU-hour; great for variable workloads |

**Distribution and sort keys (must know):**
- `DISTKEY` — column used to distribute rows across nodes; co-locate for joins
- `SORTKEY` — column(s) sorted within each slice; drives zone-map skipping
- `AUTO` — Redshift's automatic mode; usually fine for most workloads

```sql
CREATE TABLE fact_orders (
  order_id BIGINT,
  customer_id BIGINT DISTKEY,    -- co-locate with dim_customers
  order_date DATE SORTKEY,        -- predicates filter on date
  amount DECIMAL(10,2)
);
```

**Spectrum:** query S3 data from Redshift in place — predecessor to lakehouse approaches.

**Materialized views** with auto-refresh in Redshift = pre-aggregated dashboards without recomputing per query.

**AQUA (Advanced Query Accelerator)** — hardware acceleration on RA3; some queries 10× faster.

### Kinesis vs MSK vs Firehose

Recap from [06-batch-vs-streaming.md](./06-batch-vs-streaming.md):

| Service | Best for |
|---|---|
| **Kinesis Data Streams** | Application events you control |
| **Kinesis Data Firehose** | Stream → S3 / Redshift / OpenSearch with batching, no code |
| **Managed Service for Apache Flink** (formerly KDA) | Stateful streaming SQL/Java |
| **MSK** | You want Kafka semantics + ecosystem |
| **MSK Serverless** | Kafka without capacity sizing |
| **MSK Connect** | Managed Kafka Connect (e.g., for Debezium) |

**For DEA-C01 / interviews:** know that Firehose is the simplest "stream → lake" path; KDS is for app-emitted events; MSK is for Kafka-native shops.

### DMS — Database Migration / CDC

AWS Database Migration Service handles full-load + ongoing replication (CDC).

| Use | Notes |
|---|---|
| **One-time migration** | DB to DB or DB to S3 |
| **Ongoing CDC** | Postgres / MySQL / Oracle / SQL Server / Mongo → Kinesis / Kafka / S3 |
| **Schema conversion** | SCT (Schema Conversion Tool) for cross-engine migrations |

For full CDC patterns see [09-cdc.md](./09-cdc.md). DMS is the AWS-native alternative to self-hosted Debezium.

### Lake Formation — Governance on the Lake

Sits on top of Glue Catalog and adds:

- **Fine-grained access control** (column / row / cell-level)
- **Tag-based access** (LF-tags)
- **Cross-account / cross-region sharing**
- **Audit logs** of who-queried-what

**When you'd talk about it:** Compliance-heavy environments (healthcare, finance), multi-team lakes with sensitive data, cross-account data sharing.

You don't need to be an expert at this level, but be able to say: "We'd use Lake Formation for column-level masking on PII so analysts can query without seeing emails."

### IAM for DE (Often Overlooked)

Every interviewer probes IAM literacy:

| Concept | Why it matters |
|---|---|
| **Roles vs Users** | Workloads use roles; humans use users (or SSO → assumed roles) |
| **Trust policy** | Who can assume the role |
| **Permissions policy** | What the role can do |
| **Resource policies** | What's allowed *to* a resource (S3 bucket policy, KMS key policy) |
| **Service roles** | Glue service role, EMR service role, MWAA execution role — AWS expects specific patterns |
| **Cross-account access** | Assume role in another account; common for shared data lakes |
| **Least privilege** | Don't `s3:*` everything; scope to specific buckets/prefixes |

**Common pattern:** Glue job runs under a Glue service role; that role has S3 read on the source bucket, S3 write on the target, Glue catalog read/write on the relevant database, KMS use on the data keys, and CloudWatch Logs write.

---

## ADVANCED — Architectures, Pricing, and Production Patterns

### Common AWS DE Reference Architectures

**1) Lake-first analytics (modern, recommended for new builds)**

```
Sources → Kinesis Firehose / DMS / Lambda / Glue Connectors
       → S3 raw (Parquet)
       → Glue ETL (or EMR / Spark) — produces Iceberg silver
       → Glue ETL (or dbt on Athena) — produces Iceberg gold
       → Athena / Redshift Spectrum / Snowflake for analytics
       Cataloged in Glue, governed by Lake Formation
       Orchestrated by MWAA / Step Functions
```

**2) Warehouse-centric (when ML/analytics primarily lives in Redshift)**

```
Sources → DMS / Firehose → Redshift staging tables
       → dbt running against Redshift → marts
       → QuickSight / Tableau / etc.
       Spectrum used for occasional S3 access
```

**3) Streaming-heavy (for fraud, real-time analytics, IoT)**

```
Producers → Kinesis Data Streams / MSK
        → Managed Flink (KDA) → Iceberg tables on S3
        → Athena / Redshift for analytics
        Real-time consumers read directly from streams
```

You should be able to draw any of these on a whiteboard and defend choices.

### Pricing Mental Models (the cost lever cheat sheet)

| Service | Pays for | Cost lever |
|---|---|---|
| **S3** | Storage GB-months + requests + transfer | Lifecycle policies, intelligent-tiering, prefix design |
| **Glue ETL** | DPU-hours | Right-size DPUs, use bookmarks for incremental, kill long-running jobs |
| **Glue Crawlers** | Per-crawl | Schedule less often, use partition projection instead |
| **Athena** | $5 / TB scanned (varies by region) | Partition + columnar + filter; avoid SELECT * |
| **EMR** | EC2 + EMR fee | Spot instances, auto-scale, EMR Serverless for spiky |
| **Redshift** | Cluster hours (provisioned) or RPU-hours (Serverless) | Right-size, pause/resume, RA3 + concurrency scaling |
| **Kinesis Data Streams** | Shard-hours + PUT requests | Right-size shards, enhanced fan-out only when needed |
| **MSK** | Broker-hours + storage | Tiered storage, MSK Serverless |
| **MWAA** | Per environment + worker | Right-size environment, prefer Step Functions for trivial workflows |
| **Egress** | Data transferred out | Keep data in-region; use VPC endpoints; avoid cross-region |

**Common cost surprise areas:** Egress (forgotten), NAT Gateway costs (unnecessary VPC traffic), Athena scans on un-partitioned tables, oversized Redshift clusters left running 24/7.

### Networking Fundamentals (Frequently Probed)

- **VPC Endpoints (Gateway / Interface)** — keep S3 traffic on the AWS network; avoids NAT and egress charges
- **Security Groups vs NACLs** — security groups are stateful; NACLs are stateless (subnet-level)
- **Cross-account access** — assume role + bucket policies + KMS key policies; sometimes Resource Access Manager
- **PrivateLink** — expose services across VPCs/accounts privately

You're not expected to be a networking expert at mid-level, but knowing "I'd configure a VPC endpoint to keep S3 traffic off the public internet" is the kind of detail that lands well.

### Encryption and KMS

- Encryption at rest: SSE-S3, SSE-KMS (CMK), SSE-C
- Encryption in transit: TLS 1.2+ everywhere
- KMS Key policies vs IAM — both must allow key usage
- Cross-account decrypts require explicit KMS key grants

PII-heavy interviews probe this: "How do you protect customer PII in S3?" — KMS-encrypted bucket with bucket policy denying unencrypted PUTs, IAM least-privilege, Lake Formation for column masking, audit via CloudTrail.

### Secrets

- **AWS Secrets Manager** — rotated DB credentials, third-party API keys
- **Parameter Store** (SSM) — config + lighter-weight secrets
- Never put secrets in environment variables in your DAGs

### Data Sharing Patterns

| Need | Service |
|---|---|
| Share S3 data across accounts | Bucket policy + KMS grant; or Lake Formation cross-account |
| Share Redshift data | Datashares + AWS Glue Data Catalog |
| Share with external partners | S3 access points + bucket policy; Cleanrooms for federated computation |
| Multi-region replication | S3 CRR / cross-region table replication on warehouse |

### Observability on AWS

| Tool | Use |
|---|---|
| **CloudWatch Metrics** | Service health (Glue job duration, Kinesis lag, Redshift CPU) |
| **CloudWatch Logs** | Application logs from Glue / EMR / Lambda / MWAA |
| **CloudTrail** | Audit log — who did what to which resource |
| **EventBridge** | Event routing for "alert me when Glue job fails" patterns |
| **AWS X-Ray** | Distributed tracing |
| **CloudWatch Anomaly Detection** | Auto-baseline + anomaly alerts on metrics |
| **Cost Explorer / AWS Budgets** | Cost tracking and alarms |

For DE specifically, CloudWatch + custom dbt-test failure alerts + maybe Monte Carlo / OpenLineage covers the bases.

### Infrastructure-as-Code

| Tool | Notes |
|---|---|
| **Terraform** | Multi-cloud, broadest community, the de facto standard |
| **AWS CDK** | TypeScript/Python; high-level constructs, AWS-only |
| **CloudFormation** | YAML/JSON; AWS-only; what CDK compiles to |
| **AWS SAM** | Serverless-focused; CloudFormation extension |

**Mid-level interview expectation:** know Terraform basics; have used at least one to provision a Glue / EMR / Redshift / S3 setup.

### Disaster Recovery and Backups

- S3 versioning + cross-region replication for the lake
- Redshift snapshots (manual + automated)
- DynamoDB PITR (point-in-time recovery)
- Test recovery procedures — "we have backups" without "we've restored from them" is fiction

### When to Use Step Functions vs MWAA

| Step Functions | MWAA |
|---|---|
| Serverless, AWS-native | Open-source Airflow ecosystem |
| State-machine model (ASL JSON) | Python DAGs |
| Cheap for low volume; pricey at high event counts | Fixed environment cost; cheaper at scale |
| Limited lineage / observability | Full Airflow UI / OpenLineage |
| Great for short, AWS-only workflows (Glue + Lambda + EMR) | Better for complex, long-running, cross-system pipelines |

Most production AWS DE shops run **MWAA + Step Functions together**: MWAA for pipeline-of-pipelines orchestration, Step Functions for tightly-coupled AWS service workflows.

### Choosing AWS Services — A Framework

When asked "which AWS service would you use for X," structure as:
1. **What's the workload?** (volume, latency, frequency, predictability)
2. **What's the team's existing expertise?**
3. **What's the cost profile?** (sustained vs spiky; compute-heavy vs storage-heavy)
4. **What's the integration story?** (already on Redshift? Already use Spark?)
5. **What's the trade-off you're making?**

A junior answer is "use Glue." A mid-level answer is "I'd use Glue ETL because the workload is low-volume and the team doesn't want cluster ops; I'd consider EMR Serverless if costs become an issue at higher volumes."

---

## Worked Example — End-to-End AWS Pipeline

**Scenario:** Build a pipeline ingesting 50M events/day from a SaaS app via webhook, with hourly aggregates and ad-hoc analytics for 20 analysts.

**My answer:**

```
Webhook receiver (API Gateway → Lambda) → Kinesis Data Streams (events stream)
   ├── Firehose → S3 raw/events/dt=YYYY-MM-DD/hr=HH/  (bronze, Parquet)
   └── Managed Flink job → Iceberg silver.events table
                            (sessionization, dedup, schema validation)

Glue Data Catalog tracks all tables (Iceberg metadata).

dbt running on Athena (or EMR Spark with Iceberg):
  silver.events → silver.sessions → gold.daily_session_metrics

Orchestration: MWAA DAG runs hourly:
  - validate volume of last hour (Anomalo / dbt source freshness)
  - dbt build --select silver+
  - run reconciliation against application's reporting API
  - alert on failure to Slack

Analyst access:
  - Athena workgroup with per-query cost limit
  - QuickSight dashboards over gold tables
  - Lake Formation column masking on PII columns

Cost controls:
  - S3 Intelligent-Tiering on bronze (cold after 30 days)
  - Iceberg compaction nightly
  - Athena partition projection on date/hour
  - MWAA right-sized to mw1.small

Ops:
  - CloudWatch alarms on Kinesis iterator age (consumer lag)
  - dbt test failures → Slack
  - Cost anomaly detection in Cost Explorer
```

This walkthrough covers ingest, processing, lakehouse storage, transformation, orchestration, governance, observability, and cost. It's the shape of a strong mid-level system-design answer. See [15-system-design-de.md](./15-system-design-de.md) for the framework.

---

## Interview-Ready Cheat Sheet

**"Glue ETL vs EMR vs EMR Serverless?"** Glue: managed, low-ops, slightly more expensive per DPU; great for moderate ETL. EMR: full Spark control, cheapest for sustained large jobs, you manage clusters. EMR Serverless: pay-per-job, great for spiky workloads.

**"Athena pricing optimization?"** Partition data, query Iceberg / partition projection so partition pruning works, use Parquet with the right codec, list only needed columns, set workgroup query limits.

**"Redshift distribution and sort keys?"** DISTKEY co-locates rows for joins; SORTKEY orders rows for zone-map skipping. Use AUTO if you don't have specific patterns.

**"Why use Iceberg on S3?"** ACID transactions, schema/partition evolution, time travel, atomic compaction — turns a directory of Parquet into a real table that multiple engines can read/write safely.

**"Step Functions vs MWAA?"** Step Functions: serverless, AWS-native state machines, cheap for low volume, weak observability. MWAA: full Airflow ecosystem, cross-system DAGs, fixed env cost. Often used together.

**"How would you secure a data lake?"** Bucket policy + IAM least-privilege + KMS encryption + Lake Formation for fine-grained access + CloudTrail for audit + VPC endpoints to keep traffic private.

**Quick trade-off pairs:**
- Glue vs EMR: managed convenience vs cluster control
- Athena vs Redshift: serverless ad-hoc vs sustained dashboard tier
- Kinesis vs MSK: AWS-native simplicity vs Kafka ecosystem
- MWAA vs Step Functions: open-source DAG vs serverless state machine

---

## Resources & Links

- [AWS Big Data Blog](https://aws.amazon.com/blogs/big-data/) — best official source for patterns
- [AWS Well-Architected — Data Analytics Lens](https://docs.aws.amazon.com/wellarchitected/latest/analytics-lens/analytics-lens.html)
- [AWS Certified Data Engineer DEA-C01 Exam Guide](https://aws.amazon.com/certification/certified-data-engineer-associate/) — covers most services here
- [Stephane Maarek's DEA-C01 course](https://www.udemy.com/course/aws-data-engineer/) — popular cert prep
- [Tutorials Dojo DEA-C01 practice](https://portal.tutorialsdojo.com/courses/aws-certified-data-engineer-associate-practice-exam-dea-c01/)
- [AWS Athena best practices](https://docs.aws.amazon.com/athena/latest/ug/best-practices.html)
- [AWS Iceberg + Glue](https://docs.aws.amazon.com/glue/latest/dg/aws-glue-programming-etl-format-iceberg.html)

*Next: [Partitioning & Performance](./13-partitioning-performance.md) — the file format / partitioning fundamentals across all of these services.*
