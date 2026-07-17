# DEA-C01 Domain 1 — Data Ingestion and Transformation (34%)

**Weight:** ~34% of the exam — biggest domain.
**Prereqs:** [ETL vs ELT](../04-etl-vs-elt.md) · [Batch vs Streaming](../06-batch-vs-streaming.md) · [Airflow Orchestration](../05-orchestration-airflow.md) · [Spark Fundamentals](../07-spark-fundamentals.md)
**Related tasks:** T1.1 Ingest · T1.2 Transform · T1.3 Orchestrate · T1.4 Programming

---

## Why This Domain Matters

A third of the exam. This is where the "which service?" questions live. AWS wants you to know the *right* pick for each pattern — not just what a service *does*.

---

## §1 — Batch Ingestion (Task 1.1)

### Amazon S3 — the landing zone

Everything on AWS lands in S3 first. Memorize the storage classes and lifecycle:

| Class | Retrieval | Cost | Use case |
|---|---|---|---|
| **S3 Standard** | Instant | High | Hot data, active analytics |
| **S3 Intelligent-Tiering** | Instant (auto-moves) | Medium | Unknown access patterns |
| **S3 Standard-IA** | Instant | Lower storage, higher retrieval | Infrequent access, still fast when needed |
| **S3 One Zone-IA** | Instant | ~20% cheaper than Standard-IA | Reproducible data (raw logs you can re-ingest) |
| **S3 Glacier Instant Retrieval** | Instant | Very low | Archive with occasional millisecond retrieval |
| **S3 Glacier Flexible Retrieval** | Minutes–hours | Even lower | Compliance archives |
| **S3 Glacier Deep Archive** | 12+ hours | Lowest | 7–10 year retention |

**Lifecycle policies** transition between classes automatically. The exam loves questions like "compliance requires 7 years retention with rare access — cheapest option?" → Deep Archive.

**S3 Event Notifications** trigger on `ObjectCreated`, `ObjectRemoved`, etc. Destinations: SNS, SQS, Lambda, EventBridge (preferred for filtering). Common pattern: `S3 PUT → EventBridge → Lambda → Glue Job`.

**S3 Tables (new, in-scope)** — managed Iceberg tables on S3. See §4 of Domain 2.

### Amazon AppFlow

Managed **SaaS-to-AWS integration**. Point-and-click ingestion from Salesforce / ServiceNow / Zendesk / Slack / Google Analytics → S3 / Redshift / SnowFlake. When you see "no-code SaaS ingestion," think AppFlow.

### AWS DMS (Database Migration Service)

Continuous replication from a source DB (Postgres / MySQL / Oracle / SQL Server) to a target. Two modes almost always used together:

- **Full load** — initial snapshot.
- **CDC (Change Data Capture)** — ongoing changes via source DB's transaction log.

DMS targets: RDS, Aurora, Redshift, S3, Kinesis, DocumentDB, DynamoDB, OpenSearch.

**Serverless DMS** (newer) — auto-scales replication instances.

**DMS Schema Conversion** replaces the deprecated AWS SCT. Converts source-DB DDL to target-DB DDL (Oracle → PostgreSQL etc.).

Exam trap: DMS ≠ Database Migration *Assessment* — the assessment tool is separate.

### AWS Glue (batch)

Glue is many things — for batch ingestion the key ones:
- **Glue Crawlers** — scan S3, infer schema, populate the Glue Data Catalog.
- **Glue Jobs (Spark ETL)** — batch transformation code.
- **Glue Jobs (Python Shell)** — lightweight Python; smaller data volumes.

### Amazon EMR

Managed Hadoop / Spark / Presto / Hive / HBase. Three flavors:
- **EMR on EC2** — traditional. Full control, hourly billing per instance.
- **EMR on EKS** — run EMR workloads on your existing Kubernetes cluster.
- **EMR Serverless** — pay per job second, no cluster management. **Prefer this on the exam** for batch Spark unless the question specifies persistent cluster / interactive.

### AWS Lambda for ingestion

Trigger on: S3 events, Kinesis, DynamoDB Streams, EventBridge, API Gateway, SQS, MSK.

Limits to memorize:
- **Timeout:** 15 minutes max.
- **Memory:** 128 MB to 10240 MB.
- **Ephemeral storage (`/tmp`):** 512 MB default, up to 10 GB.
- **Package size:** 50 MB zipped, 250 MB unzipped; up to 10 GB via container image.
- **Concurrency:** default 1000 per region, requestable higher.
- **Reserved concurrency** guarantees a slot; **provisioned concurrency** keeps instances warm (kills cold start).

Common trap: Lambda has a 15-min timeout — for longer jobs use Step Functions, Glue, or ECS/Fargate.

### Amazon Redshift as ingestion target

Two key ingestion paths:
- **COPY command** — bulk load from S3 (most common, MPP-parallel).
- **Redshift streaming ingestion** — ingest directly from Kinesis or MSK into a materialized view. Newer, low-latency.
- **Redshift Zero-ETL** integrations with Aurora / RDS / DynamoDB — no pipeline needed.

---

## §2 — Streaming Ingestion (Task 1.1)

### Kinesis Data Streams (KDS)

Managed **event streaming**. Concepts:

- **Shard** — parallelism unit. 1000 records/sec write, 2 MB/sec write, 5 reads/sec, 2 MB/sec read.
- **Enhanced fan-out** — dedicated 2 MB/sec/consumer, sub-200ms latency. Costs extra per consumer per shard.
- **Retention:** 24 hours default, up to 365 days.
- **Partition key** — hashed → shard assignment. Bad key → hot shards.

**On-demand** vs **provisioned** mode:
- **On-demand** — auto-scales; pay per GB. Pick for spiky/unknown loads.
- **Provisioned** — you pick shard count. Pick for predictable steady load.

### Kinesis Data Firehose

Fully managed **delivery-to-destination** (S3, Redshift, OpenSearch, Splunk, HTTP endpoints). Zero code.

- **Buffering:** by size (1–128 MB) or time (60–900 seconds).
- **Format conversion:** JSON → Parquet/ORC on the fly.
- **Compression:** GZIP, Snappy, ZIP.
- **Lambda transformation:** invoke Lambda per micro-batch for enrichment.

**KDS vs Firehose (exam favorite):**
- KDS: real-time consumers, replay, multiple readers, sub-second custom processing.
- Firehose: near-real-time (min 60s buffer), managed delivery, no code, S3/Redshift/OS destination.

### Amazon MSK (Managed Kafka)

Managed Apache Kafka. Two flavors:
- **MSK Provisioned** — you pick broker instance type + count.
- **MSK Serverless** — auto-scaled, per-throughput billing.

**MSK Connect** — managed Kafka Connect for source/sink connectors (Debezium, S3 sink, JDBC, etc.).

**Pick MSK over Kinesis when:** you already run Kafka on-prem, need Kafka's exactly-once semantics, need MirrorMaker for multi-cluster, or need Connect ecosystem.

### DynamoDB Streams

Time-ordered log of item-level changes in a DynamoDB table. Retention: 24 hours. Consumed by Lambda triggers or Kinesis (via Kinesis adapter). Used for CDC, cross-region replication, cache invalidation.

### Amazon Managed Service for Apache Flink (MSAF)

Formerly "Kinesis Data Analytics for Apache Flink." Runs Flink jobs against streaming sources. Use for windowing, joins, event-time watermarks, exactly-once semantics.

### Fan-in / fan-out patterns

- **Fan-in:** many producers → one stream. Kinesis handles this natively (any producer can write to any shard).
- **Fan-out:** one stream → many consumers. Standard iterator = shared 2 MB/sec across consumers; **enhanced fan-out** = dedicated 2 MB/sec/consumer.

### Replayability

- **KDS:** replay by shard iterator type (`TRIM_HORIZON`, `AT_TIMESTAMP`, `AT_SEQUENCE_NUMBER`). Retention up to 365 days.
- **Firehose:** no replay — it's just delivery.
- **MSK:** replay from any offset within topic retention.
- **S3 raw zone:** the ultimate replay source; keep raw JSON/Avro for 30+ days.

---

## §3 — Transformation with Glue (Task 1.2)

### Glue architecture

Serverless Spark under the hood. Key components:

- **Glue Data Catalog** — Hive-compatible metastore for S3 tables (used by Athena, EMR, Redshift Spectrum, Iceberg).
- **Glue Crawlers** — discover schema, populate catalog.
- **Glue Jobs** — Spark or Python Shell ETL scripts.
- **Glue Studio** — visual job authoring.
- **Glue DataBrew** — visual data prep (250+ built-in transformations, no code).
- **Glue Workflows** — DAG orchestration of crawlers + jobs (Domain 1 §5).
- **Glue Data Quality** — DQDL rules on datasets (Domain 3).

### Glue Job types

| Type | Language | Use for |
|---|---|---|
| Spark ETL (Glue 4.0 / 5.0) | PySpark or Scala | Standard ETL, ML data prep |
| Spark Streaming | PySpark or Scala | Streaming ETL from Kinesis / Kafka |
| Python Shell | Python | Small jobs (< 4 vCPUs, no distributed compute needed) |
| Ray | Ray | Distributed Python beyond Spark |

**Worker types** (Spark ETL):
- `G.1X` — 1 DPU (4 vCPU, 16 GB RAM). Default.
- `G.2X` — 2 DPU.
- `G.4X`, `G.8X` — larger, for heavy workloads.
- **DPU** = Data Processing Unit = 4 vCPU + 16 GB.

### Glue DynamicFrames vs DataFrames

- **DynamicFrame** — Glue's schema-flexible abstraction. Better for messy/inconsistent input (choice types, `resolveChoice`, `dropNullFields`).
- **DataFrame** — standard Spark. Convert with `.toDF()` and back with `DynamicFrame.fromDF()`.

Exam trap: Use DynamicFrame for the "messy JSON" question, DataFrame for the "efficient joins" question.

### Format conversion — the exam classic

`CSV / JSON → Parquet` (or ORC) reduces storage by 5–10× and speeds analytics queries 10–100× (columnar + compression + predicate pushdown). This shows up constantly. Also common:
- **Parquet with Snappy** compression — best speed.
- **Parquet with GZIP** — better compression, slower.
- **ORC** — similar to Parquet; more common in the Hive ecosystem.

### Job bookmarks

Track processed data so re-runs skip it. Options: `Enable`, `Disable`, `Pause`. Critical for **incremental ETL**.

---

## §3.5 — EMR (deeper)

EMR is Spark/Hadoop-heavy. When to pick over Glue:
- Complex Spark customization (custom JARs, specific Spark version).
- Non-Spark workloads (Hive, HBase, Presto, Flink).
- Very long-running jobs where Glue billing is more expensive.
- Interactive notebooks (EMR Studio).

**EMR pricing patterns:**
- **On-Demand** — most expensive, no interruption.
- **Reserved Instances** — commit for 1–3 years.
- **Spot** for task nodes — up to 90% cheaper. **Never for master or core** (data loss).
- **Instance fleets** — mix on-demand + spot with capacity targets.

**EMR Serverless** — best default for exam scenarios that don't specify persistent cluster.

### Container-based compute for data engineering

**ECS** — orchestrates Docker containers. Fargate = serverless mode (no EC2 to manage).
**EKS** — managed Kubernetes. Use for portable, K8s-native pipelines or when you already have K8s expertise.
**ECR** — container registry.

Exam: when a question mentions "container-based ETL with existing Docker images," think ECS/Fargate for simplicity, EKS for K8s-native.

---

## §4 — Integrating LLMs for Data Processing (Skill 1.2.10)

New in Version 1.1 of the exam. AWS's answer is **Amazon Bedrock**.

### Amazon Bedrock

Managed access to foundation models (Anthropic Claude, Meta Llama, Mistral, Amazon Nova, Cohere embeddings). Two consumption modes:
- **On-demand** — pay per token.
- **Provisioned throughput** — dedicated capacity for consistent low latency.

**Bedrock features to recognize:**
- **Model invocation** via API — used in Lambda or Glue for enrichment.
- **Bedrock Agents** — orchestrates tool use around an FM.
- **Bedrock Knowledge Bases** — managed RAG. Ingests S3 → chunks → embeds → stores in a vector DB (OpenSearch Serverless, Aurora pgvector, Pinecone, MongoDB Atlas, Redis) → serves retrieval.
- **Bedrock Guardrails** — content filtering, PII masking, denied topics.

### Common LLM-in-pipeline patterns

- **Batch enrichment:** Glue job reads records → calls Bedrock InvokeModel → writes structured output (classification, extraction, summary). Rate-limit with SDK retries + provisioned throughput.
- **Streaming enrichment:** Kinesis → Lambda → Bedrock → destination. Watch Lambda timeout (15 min) and Bedrock throttling.
- **Embeddings pipeline:** documents → chunk in Lambda/Glue → Bedrock Titan Embeddings → store in Aurora pgvector or OpenSearch Serverless with HNSW index.
- **RAG serving:** query → Bedrock Knowledge Base → LLM answer with citations.

### Amazon Kendra vs Bedrock KB

- **Kendra** — intelligent search over enterprise docs, pre-Bedrock but still valid. Keyword + semantic hybrid.
- **Bedrock Knowledge Bases** — the newer, LLM-centric alternative. Recommend on the exam for "LLM-powered document search."

---

## §5 — Orchestration (Task 1.3)

### AWS Step Functions

Serverless state machine. Two workflow types:

| | Standard | Express |
|---|---|---|
| Duration | Up to 1 year | Up to 5 minutes |
| Pricing | Per state transition | Per request + duration |
| Rate | 2000 exec/sec/account | 100,000+ exec/sec |
| At-least-once vs exactly-once | Exactly-once | At-least-once (Async) / Sync options |
| Use for | Long-running ETL, human-in-the-loop, orchestration | High-volume event processing, IoT, streaming |

**Task states** integrate with 220+ AWS services (SDK integration or optimized integration). Common: Lambda, Glue, EMR, ECS, Batch, SageMaker.

**Distributed Map** — process up to 10,000 concurrent iterations over S3 objects. Perfect for parallel ETL of many files.

**Catch / Retry** — built-in error handling per state.

### Amazon MWAA (Managed Workflows for Apache Airflow)

Managed Airflow 2.x. Pick when:
- Team already knows Airflow.
- Complex Python-based DAGs with heavy dependencies.
- Rich ecosystem of Airflow providers (Snowflake, dbt, Databricks, etc.).

Cross-read: [Airflow Orchestration](../05-orchestration-airflow.md).

**MWAA vs Step Functions (exam favorite):**
- MWAA: complex Python DAGs, Airflow ecosystem, long history of scheduled jobs.
- Step Functions: native AWS integration, serverless, simpler state machines, cheaper for lightweight orchestration.

### AWS Glue Workflows

Chain Glue crawlers + jobs into a DAG. Lightweight; only orchestrates Glue tasks. Use when everything is Glue.

### Amazon EventBridge

Event bus for AWS-native and SaaS events. Two key features:
- **Event patterns** — filter by JSON structure ("only S3 PUT events with `.parquet` suffix").
- **Rules** — route matched events to Lambda / Step Functions / SQS / SNS / MSK.
- **Scheduler** — cron-style scheduling for tasks (replaces CloudWatch Events schedules).
- **Pipes** — point-to-point event pipes with filtering + enrichment (source → filter → enrich → target).

### AWS Lambda in orchestration

Common orchestration patterns:
- Lambda as the "glue" between events (S3 → Lambda → invoke Glue job).
- Lambda + Step Functions for stateful workflows.
- Lambda + SQS for queue-based decoupling.

### SNS vs SQS

- **SNS** — pub/sub, fan-out to many subscribers. Push-based.
- **SQS** — queue, one message → one consumer. Pull-based.
- **SNS + SQS fanout pattern** — SNS topic with multiple SQS queue subscribers = reliable fan-out with per-consumer buffering.

**SQS types:**
- **Standard** — at-least-once, unordered.
- **FIFO** — exactly-once, ordered (300 msg/sec, 3000 with batching).

**DLQ (Dead Letter Queue)** — SNS/SQS/Lambda failures go here after N retries.

---

## §6 — Programming Concepts (Task 1.4)

### Lambda concurrency & performance

- **Cold start** — first invocation after idle; provisioned concurrency eliminates it.
- **Reserved concurrency** — cap and guarantee.
- **Provisioned concurrency** — always-warm instances (paid).
- **SnapStart** (Java) — snapshot-based fast cold starts.
- **Memory tuning** — CPU scales with memory. `AWS Lambda Power Tuning` (open-source tool) finds cost/perf optimum.

### CI/CD for data pipelines

- **CodeCommit** — was Git. **Deprecated in 2024–2025**; not on current exam (removed from in-scope in v1.1).
- **CodeBuild** — build service.
- **CodeDeploy** — deployment to EC2/Lambda/ECS.
- **CodePipeline** — end-to-end orchestration.

Common data-engineering CI/CD flow: git push → CodeBuild runs `sam build` / `cdk synth` → CloudFormation deploys.

### Infrastructure as Code (IaC)

- **CloudFormation** — AWS-native. YAML/JSON templates. Stack sets across regions/accounts.
- **AWS CDK (Cloud Development Kit)** — TypeScript / Python / Java / Go / C#. Synthesizes to CloudFormation. Best for complex infra.
- **AWS SAM (Serverless Application Model)** — CloudFormation dialect optimized for serverless (Lambda, API Gateway, DynamoDB). `sam local` for local testing.
- **Terraform** — third-party, out of scope for the exam but common in industry.

**AWS CDK is the current AWS-recommended way** for new IaC projects. Learn it — it's on the exam.

### Data structures & algorithms (Skill 1.4.11)

The exam asks light-touch questions about **graph** and **tree** structures. Don't panic — they're conceptual:
- **Graph DB (Neptune)** — traversal queries, "friends-of-friends."
- **Tree structures** — hierarchical data (org charts, category taxonomies), stored as adjacency list or nested set.

### Distributed computing (Skill 1.4.10)

Recognize the terms: MapReduce, shuffle, partitioning, data locality, fault tolerance via lineage (Spark RDD), master-slave, coordination (ZooKeeper). Cross-read: [Spark Fundamentals](../07-spark-fundamentals.md).

---

## Common Exam Traps

**Q: "Lowest-latency streaming ingestion with replay across multiple consumers"**
A: Kinesis Data Streams with enhanced fan-out. (Not Firehose — no replay. Not MSK unless the question specifies Kafka.)

**Q: "Near-real-time delivery to S3 in Parquet with format conversion, no code"**
A: Kinesis Data Firehose (converts JSON → Parquet in-flight).

**Q: "Batch ETL that must run daily, output to Iceberg on S3"**
A: Glue Job (Spark) with Iceberg connector, orchestrated by Step Functions or Glue Workflow.

**Q: "Migrate an on-prem Oracle DB to Aurora Postgres with ongoing sync"**
A: DMS (full-load + CDC) + DMS Schema Conversion for DDL.

**Q: "Ingest Salesforce data daily with zero code"**
A: Amazon AppFlow.

**Q: "LLM-powered document Q&A"**
A: Amazon Bedrock Knowledge Bases (or Kendra for older answers, but prefer KB).

**Q: "Distributed parallel processing of 10,000 S3 files"**
A: Step Functions **Distributed Map** state.

**Q: "Event bus that routes S3 events to different Lambdas based on prefix"**
A: EventBridge with event patterns (not raw S3 event notifications — those don't filter).

**Q: "Exactly-once message processing, ordered"**
A: SQS FIFO.

**Q: "Real-time Kafka processing with existing Kafka expertise"**
A: MSK (not Kinesis).

---

## Concept-Check (self-quiz)

1. Kinesis Data Streams shard write and read limits per second?
2. Firehose buffering minimums and maximums?
3. When do you use enhanced fan-out?
4. Glue DynamicFrame vs DataFrame — one advantage of each.
5. EMR Spot instances — safe on which node types?
6. Step Functions Standard vs Express — three differences.
7. DMS CDC — what does it read from the source DB?
8. Lambda maximum timeout? Max memory?
9. AppFlow — what problem does it solve?
10. Bedrock Knowledge Base — what does it manage for you?
11. S3 event → EventBridge vs directly to Lambda — why prefer EventBridge?
12. Iceberg connector in Glue — what does it enable?
13. SQS Standard vs FIFO — pick one for "ordered exactly-once."
14. CDK vs CloudFormation — which is recommended for new IaC?
15. Distributed Map in Step Functions — max concurrency?

If any is fuzzy, that's the section to re-read.

---

## Resources

- [AWS Kinesis docs](https://docs.aws.amazon.com/kinesis/)
- [AWS Glue docs](https://docs.aws.amazon.com/glue/)
- [AWS Step Functions docs](https://docs.aws.amazon.com/step-functions/)
- [Amazon Bedrock docs](https://docs.aws.amazon.com/bedrock/)
- [MWAA docs](https://docs.aws.amazon.com/mwaa/)
- [AWS Big Data Blog — ingestion patterns](https://aws.amazon.com/blogs/big-data/)

Cross-reference: [12-aws-data-stack.md](../12-aws-data-stack.md) has the wider AWS overview.

---

*Next: [Domain 2 — Data Store Management](./02-domain2-data-store-management.md).*
