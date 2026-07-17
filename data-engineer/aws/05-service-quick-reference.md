# DEA-C01 — In-Scope Service Quick Reference

**Purpose:** one-page-per-service summary of every DEA-C01 in-scope AWS service. If you can produce these one-liners cold, service-recognition questions become instant.

Grouped by category (matches the exam guide).

---

## Analytics

### Amazon Athena
- Serverless Presto/Trino over S3. Pay per TB scanned.
- Uses Glue Data Catalog for metadata.
- Best cost win: Parquet + partition + partition projection.
- Athena for Apache Spark = interactive PySpark notebooks.
- CTAS = `CREATE TABLE AS SELECT` — reformat data in place.
- Federated queries via Lambda connectors (RDS, DynamoDB, etc.).

### Amazon EMR
- Managed Hadoop / Spark / Presto / Hive / HBase / Flink.
- Three flavors: **EMR on EC2**, **EMR on EKS**, **EMR Serverless**.
- Pick EMR Serverless for exam scenarios by default.
- Spot for task nodes only. Never for master/core.
- Multi-master cluster = high-availability master.

### AWS Glue
- Serverless Spark ETL + catalog.
- Components: Data Catalog, Crawlers, Jobs (Spark/Python Shell/Ray), Studio (visual), DataBrew (visual prep), Workflows, Data Quality, Schema Registry.
- DynamicFrame for messy schemas; DataFrame for efficient joins.
- Bookmarks = incremental processing.
- Worker types: G.1X (default), G.2X, G.4X, G.8X.
- Glue for streaming = micro-batch Spark Streaming.

### AWS Glue DataBrew
- Visual data prep (no code). 250+ transformations.
- Recipes → Jobs. Built-in PII detection.
- Sources: S3, RDS, Redshift, Snowflake.

### AWS Lake Formation
- Governance layer over S3 data lakes.
- Fine-grained access: column, row, cell, LF-tags.
- Enforces on Athena, Redshift Spectrum, EMR, Glue, QuickSight, SageMaker.
- LF-tags = ABAC for the data lake.

### Amazon Kinesis Data Firehose
- Fully managed delivery to S3 / Redshift / OpenSearch / Splunk / HTTP.
- Buffers 1–128 MB or 60–900s.
- Format conversion JSON → Parquet on the fly.
- No replay. No custom consumers.

### Amazon Kinesis Data Streams (KDS)
- Real-time event streaming.
- Shard: 1 MB/s in, 2 MB/s out; 1000 records/s write; 5 reads/s.
- Enhanced fan-out for dedicated consumer throughput.
- Retention 24h–365d.
- On-demand vs provisioned mode.

### Amazon Managed Service for Apache Flink
- Managed Flink jobs. Windowing, watermarks, joins.
- Replaces older "Kinesis Data Analytics."

### Amazon Managed Streaming for Apache Kafka (MSK)
- Managed Kafka. Provisioned or Serverless.
- MSK Connect for Kafka Connect (Debezium, S3 sink, etc.).
- Pick over Kinesis when you already have Kafka expertise / need Connect.

### Amazon OpenSearch Service
- Managed OpenSearch / Elasticsearch.
- Serverless option available.
- Use cases: log analytics, full-text search, vector k-NN (HNSW).

### Amazon QuickSight
- BI dashboards. SPICE = in-memory columnar cache.
- Enterprise edition adds ML insights, embedded, row-level security, VPC.
- Q = natural-language querying.

### Amazon SageMaker AI
- ML platform (out of exam depth). Recognize its role.
- SageMaker Data Wrangler for feature engineering.
- SageMaker Catalog for business data catalog.
- ML Lineage Tracking.

---

## Application Integration

### Amazon AppFlow
- No-code SaaS-to-AWS ingestion. Salesforce, ServiceNow, Zendesk, Slack, Google Analytics.
- Sources → S3 / Redshift / Snowflake.

### Amazon EventBridge
- Event bus with pattern-based routing.
- Sub-services: **default bus**, **Scheduler** (cron), **Pipes** (source→filter→enrich→target).
- Routes to Lambda / Step Functions / SQS / SNS / Kinesis / MSK.

### Amazon MWAA (Managed Workflows for Apache Airflow)
- Managed Airflow 2.x.
- Pick for complex Python DAGs + Airflow provider ecosystem.

### Amazon SNS
- Pub/sub. Fan-out to many subscribers. Push-based.
- Destinations: SQS, Lambda, HTTP, email, SMS, mobile push.
- FIFO SNS for ordered pub/sub.

### Amazon SQS
- Message queue. Pull-based.
- Standard (at-least-once) or FIFO (exactly-once, ordered, 300 msg/s).
- DLQ for retries exhausted.
- Long polling avoids empty-poll charges.

### AWS Step Functions
- Serverless state machine.
- Standard (1yr, exactly-once, per state transition pricing) vs Express (5min, per request pricing).
- **Distributed Map** = 10K concurrent iterations.
- Integrates with 220+ AWS services.

---

## Cloud Financial Management

### AWS Budgets
- Set alerts on spend thresholds.

### AWS Cost Explorer
- Analyze cost + usage over time.

---

## Compute

### AWS Batch
- Managed batch jobs on EC2 or Fargate.
- Job queues + compute environments.
- Auto-scaling based on queue depth.

### Amazon EC2
- VMs. Understand instance families at high level.
- Spot / On-Demand / Reserved / Savings Plans.

### AWS Lambda
- Serverless functions. 15-min max timeout.
- Memory 128 MB – 10240 MB (CPU scales linearly).
- Ephemeral `/tmp` up to 10 GB.
- Package: 50 MB zip, 250 MB unzip, 10 GB container.
- Reserved concurrency (guarantee) vs Provisioned concurrency (warm).

### AWS SAM (Serverless Application Model)
- CloudFormation dialect for Lambda + API Gateway + DynamoDB.
- `sam local` for local testing.

---

## Containers

### Amazon ECR
- Container registry. Vulnerability scanning built in.

### Amazon ECS
- Managed Docker orchestration.
- EC2 launch type or Fargate (serverless).

### Amazon EKS
- Managed Kubernetes.
- IRSA (IAM Roles for Service Accounts) for pod-level IAM.
- Fargate profile for serverless pods.

---

## Database

### Amazon DocumentDB
- MongoDB-compatible managed database.

### Amazon DynamoDB
- Serverless NoSQL. Single-digit ms.
- Modes: on-demand or provisioned. Auto-scaling for provisioned.
- Indexes: GSI (different PK, own capacity, eventually consistent) vs LSI (same PK, shared capacity, at creation only).
- Streams (24h retention).
- TTL for auto-expire (flows through Streams).
- DAX for microsecond reads. Global Tables for multi-region multi-active. PITR up to 35 days.

### Amazon Keyspaces
- Managed Cassandra.

### Amazon MemoryDB for Redis
- Durable Redis. Sub-ms KV.
- Multi-AZ transaction log.

### Amazon Neptune
- Graph DB (property + RDF). Gremlin, SPARQL, openCypher.

### Amazon RDS
- Managed relational: Postgres / MySQL / MariaDB / SQL Server / Oracle.
- Read replicas, Multi-AZ, backups.

### Amazon Aurora
- MySQL/Postgres-compatible, cloud-native.
- 6-way replicated storage across 3 AZs.
- Serverless v2 auto-scales.
- Read replicas (up to 15).
- Global Database for cross-region.
- **Aurora PostgreSQL + pgvector** = HNSW vector search alongside SQL.
- Aurora Zero-ETL to Redshift.

### Amazon Redshift
- MPP columnar warehouse.
- Deployment: RA3 (provisioned) or Serverless.
- Node design: DISTKEY + SORTKEY + encoding.
- Features: Spectrum, Streaming ingestion, Federated queries, Data Sharing, ML, Zero-ETL, Materialized views, AQUA, Concurrency Scaling.

---

## Developer Tools

### AWS CLI / AWS CDK / AWS CloudFormation
- CLI for API access. CDK for IaC in TS/Python/Java. CloudFormation for template-based infra.

### AWS CodeBuild / CodeDeploy / CodePipeline
- Build / Deploy / Orchestrate CI/CD.

### Amazon Q
- AWS's assistant for code and questions.

---

## Web and Mobile

### Amazon API Gateway
- REST / HTTP / WebSocket APIs.
- Backed by Lambda / HTTP / VPC Link.
- Throttling, caching, auth (Cognito, IAM, JWT).

---

## Machine Learning

### Amazon Bedrock
- Managed foundation models (Anthropic, Meta, Mistral, Nova, Cohere).
- Features: **InvokeModel** API, **Knowledge Bases** (managed RAG), **Agents**, **Guardrails**.
- On-demand or Provisioned Throughput.

### Amazon Kendra
- Intelligent enterprise search. Semantic + keyword hybrid.
- Pre-Bedrock era but still valid.

### Amazon SageMaker AI
- ML platform. Autopilot, Studio, Ground Truth, Feature Store, Pipelines.
- Unified Studio = new integrated experience (Skill 4.1.7).

---

## Management and Governance

### AWS CloudTrail
- Every API call. Management events (default) vs data events (paid, S3/Lambda).
- CloudTrail Lake for centralized SQL queries.

### Amazon CloudWatch
- Metrics + Logs + Alarms + Dashboards + Events (moved to EventBridge).
- Standard 1min / High-res 1s.
- Logs Insights for SQL-lite log queries.

### AWS Config
- Continuous compliance monitoring. Config Rules with auto-remediation.

### Amazon Managed Grafana
- Managed Grafana dashboards.

### AWS Systems Manager
- Parameter Store for non-secret config.
- Session Manager for shell into EC2.
- Automation for runbooks.

### AWS Well-Architected Tool
- Framework review tool. Six pillars.

### AWS Data Exchange
- Marketplace for third-party datasets. Deliver into S3 / Redshift.

---

## Migration and Transfer

### AWS DMS
- Continuous DB replication. Full-load + CDC.
- Serverless DMS auto-scales.
- Schema Conversion (replaces AWS SCT).

### AWS DataSync
- Move data between S3 / EFS / FSx / on-prem NFS/SMB. Incremental.

### AWS Snow Family
- Physical devices for TB–PB migrations. Snowcone / Snowball / Snowmobile.

### AWS Transfer Family
- Managed SFTP / FTPS / FTP / AS2 endpoints. Backed by S3 or EFS.

### AWS Application Migration Service (MGN)
- Lift-and-shift server migration.

---

## Networking and Content Delivery

### Amazon CloudFront
- CDN. Edge locations.

### AWS PrivateLink
- Private access to AWS services via ENIs in your VPC.

### Amazon Route 53
- DNS + health checks + routing policies.

### Amazon VPC
- Isolated network. Subnets, route tables, security groups, NACLs.
- Endpoints: gateway (S3, DynamoDB) vs interface (everything else, via PrivateLink).

---

## Security, Identity, and Compliance

### AWS IAM
- Identity and access. Users, groups, roles, policies.
- Managed vs inline policies. Resource policies.
- IAM Identity Center (SSO).

### AWS KMS
- Managed encryption keys. CMKs. Envelope encryption. Key policies.

### Amazon Macie
- PII / PHI discovery in S3.

### AWS Secrets Manager
- Rotating secrets. Native integration with RDS.

### AWS Shield / AWS WAF
- DDoS (Shield) and L7 firewall (WAF).

---

## Storage

### AWS Backup
- Centralized backups across services.

### Amazon EBS
- Block storage attached to EC2. Snapshots to S3.

### Amazon EFS
- Managed NFS. Multi-AZ. IA / One Zone tiers.

### Amazon S3
- Object store. Storage classes: Standard, IA, One-Zone-IA, Intelligent-Tiering, Glacier IR / Flexible / Deep Archive.
- Features: Versioning, Object Lock, Lifecycle, Event Notifications, Replication (CRR / SRR), Access Points, VPC endpoints, Access Analyzer.
- Encryption: SSE-S3 (default), SSE-KMS, SSE-C, DSSE-KMS.

### Amazon S3 Tables (new)
- Managed Iceberg tables on S3. Auto-compaction, snapshot expiration. Table buckets.

### Amazon S3 Glacier / Deep Archive
- Archival storage classes. Compliance mode Object Lock for WORM.

---

## The "Which Service" Decision Cheat Sheet

| Need | Answer |
|---|---|
| Real-time streaming with replay | KDS |
| Managed streaming delivery to S3 | Firehose |
| Kafka | MSK |
| Streaming windowing / joins | MSAF (Flink) |
| Serverless SQL over S3 | Athena |
| MPP columnar warehouse | Redshift |
| Query S3 from Redshift | Redshift Spectrum |
| Sub-ms KV | DynamoDB (+ DAX) or MemoryDB |
| OLTP relational | RDS or Aurora |
| Vector + relational | Aurora PostgreSQL + pgvector |
| Graph DB | Neptune |
| Document DB (Mongo API) | DocumentDB |
| Cassandra API | Keyspaces |
| Full-text search | OpenSearch |
| No-code SaaS ingestion | AppFlow |
| DB migration + CDC | DMS |
| File transfer SFTP/FTPS | Transfer Family |
| Serverless orchestration | Step Functions |
| Complex Python DAG orchestration | MWAA |
| Event routing | EventBridge |
| Managed Iceberg | S3 Tables |
| No-code data prep | Glue DataBrew |
| Visual ETL | Glue Studio |
| BI dashboards | QuickSight |
| PII discovery in S3 | Macie |
| Fine-grained data lake access | Lake Formation |
| Rotating DB password | Secrets Manager |
| Non-secret config | Parameter Store |
| LLM API access | Bedrock InvokeModel |
| Managed RAG | Bedrock Knowledge Bases |
| Enterprise semantic search | Kendra |
| Serverless compute | Lambda / Fargate |
| Kubernetes | EKS |
| Docker (non-K8s) | ECS Fargate |
| CDN | CloudFront |
| IaC | CloudFormation / CDK / SAM |
| CI/CD | CodePipeline (+ CodeBuild, CodeDeploy) |

---

*If you can answer 90% of these one-liners without hesitation, service-recognition MCQs become free points.*
