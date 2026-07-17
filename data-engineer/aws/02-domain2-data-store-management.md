# DEA-C01 Domain 2 — Data Store Management (26%)

**Weight:** ~26% of the exam.
**Prereqs:** [Warehouses & Lakehouses](../03-warehouses-lakes-lakehouse.md) · [Table Formats](../08-table-formats.md) · [Partitioning & Performance](../13-partitioning-performance.md)
**Related tasks:** T2.1 Choose a data store · T2.2 Cataloging · T2.3 Lifecycle · T2.4 Data models + schema evolution

---

## §1 — Storage Service Picker (Task 2.1)

The exam constantly asks "which service for pattern X?" Memorize this decision tree:

| Pattern | Service | Why |
|---|---|---|
| Point KV lookups, ms latency, huge scale | **DynamoDB** | Serverless NoSQL, single-digit ms, global tables |
| Sub-ms KV, ultra-low latency, in-memory | **MemoryDB for Redis / ElastiCache** | RAM-resident, MemoryDB is durable Redis |
| Relational transactional, standard SQL | **RDS (Postgres, MySQL, MariaDB, SQL Server, Oracle)** | Managed OLTP |
| High-availability relational, auto-scale | **Aurora** | RDS-compatible, 6-way replication, storage auto-scales |
| Petabyte-scale analytical warehouse | **Redshift** | Columnar MPP |
| Query S3 with SQL, ad hoc, no infra | **Athena** | Serverless Presto/Trino |
| Full-text + vector search | **OpenSearch Service** | Managed OpenSearch/Elasticsearch |
| Vector search with HNSW alongside SQL | **Aurora PostgreSQL with pgvector** | Skill 2.1.3 explicitly mentions this |
| Graph queries, multi-hop | **Neptune** | Property + RDF graphs |
| Wide-column NoSQL, Cassandra-compatible | **Amazon Keyspaces** | Managed Cassandra |
| Document store, MongoDB-compatible | **DocumentDB** | Managed MongoDB API |
| Data lake on S3 with governance | **Lake Formation over S3** | Fine-grained access, Iceberg tables |
| Streaming state (KV) | **DynamoDB or MemoryDB** | Fast + durable |

---

## §2 — Amazon Redshift (deep dive)

Redshift shows up in ~5–10 questions. Know it cold.

### Deployment options

- **Provisioned (RA3 nodes)** — you pick node type + count. Storage separated from compute (RA3 uses managed storage in S3).
- **Redshift Serverless** — auto-scales, pay per RPU-hour. Pick for spiky or unknown workloads. Newer default.
- **Legacy DC2 / DS2 nodes** — still on the exam but AWS is moving people off; RA3 is the modern default.

### Table design — the 4 knobs

**1. Distribution style (DISTKEY / DISTSTYLE):**

| Style | Behavior | Use for |
|---|---|---|
| `KEY` | Rows with the same distribution key land on the same node | Large fact tables that join on the same key |
| `ALL` | Full copy on every node | Small dimension tables (<3M rows) |
| `EVEN` | Round-robin | No obvious join key |
| `AUTO` | Redshift picks | Default for new tables; usually fine |

The classic trap: two large tables joined on a common key → give both the same DISTKEY to co-locate rows → avoid shuffle.

**2. Sort key (SORTKEY):**

- **Compound** — data sorted by column1 then column2. Skips blocks when filter matches leading column. Default choice.
- **Interleaved** — equally weights multiple columns. Rarely used; complex maintenance (VACUUM REINDEX).
- **AUTO** — Redshift picks.

**Zone maps** cache each block's min/max — the query engine skips blocks that can't match the filter. **This is why sort keys speed up queries so much.**

**3. Encoding (compression):**
- `AZ64` — best for numeric types (default).
- `ZSTD` — best general-purpose.
- `LZO`, `DELTA`, `BYTEDICT`, `TEXT255` — situational.
- `ANALYZE COMPRESSION` recommends encodings.

**4. VACUUM & ANALYZE:**
- `VACUUM` — reclaims space, re-sorts.
- `ANALYZE` — updates statistics for the query planner.
- **Auto-VACUUM and Auto-ANALYZE** run in background on RA3 and Serverless.

### Redshift Spectrum

Query S3 data directly from Redshift using external tables (Glue Catalog). Same SQL. Pay per TB scanned.

- **When to use:** joining massive S3 datasets with Redshift-resident dimensions; querying cold data without loading.
- **Trap:** if you scan everything, you pay everything. Partition + Parquet + predicate pushdown.

### Redshift Data Sharing

Share live data between Redshift clusters (producer → consumer) without copying. Cross-account, cross-region. Enables data-mesh patterns.

### Redshift Materialized Views + Auto-refresh

Precompute expensive aggregates. Redshift can auto-refresh incrementally on base-table changes.

### Redshift Streaming Ingestion

Materialized view directly on top of Kinesis or MSK stream. Micro-batch every ~10 seconds. Low-latency dashboards.

### Federated Queries

Query RDS / Aurora directly from Redshift SQL — no ETL. Great for real-time joins between OLTP and warehouse.

### Redshift ML

`CREATE MODEL` statement trains SageMaker Autopilot model, exposes as SQL function for inline predictions.

### Zero-ETL

Point RDS/Aurora/DynamoDB at Redshift; changes replicated automatically. Kills a whole category of ETL pipelines.

### Concurrency Scaling

Automatic transient clusters when concurrency spikes. First hour of usage per day is free per cluster.

### Workload Management (WLM)

Queue-based resource allocation. **Auto WLM** is default and usually enough.

### Redshift AQUA (Advanced Query Accelerator)

Hardware acceleration for RA3 clusters. Automatic; no config.

---

## §3 — DynamoDB (deep dive)

### Table basics

- **Partition key (PK)** — hashed to determine partition. Must be well-distributed to avoid hot partitions.
- **Sort key (SK)** — optional. `(PK, SK)` is the composite primary key. Enables range queries.
- **Item** — a row. Max 400 KB.
- **Attribute** — a column. Schemaless per-item.

### Capacity modes

- **On-demand** — pay per request. Bursts to any level.
- **Provisioned** — you pick RCU / WCU. Auto-scaling optional. Cheaper for predictable steady load.
- **1 RCU** = 1 strongly consistent read/sec (up to 4 KB) or 2 eventually consistent.
- **1 WCU** = 1 write/sec (up to 1 KB).

### GSI vs LSI

**Local Secondary Index (LSI):**
- Same partition key as base, different sort key.
- Must be created at table creation.
- Shares WCU/RCU with base table.
- Max 5 per table.
- Strong consistency available.

**Global Secondary Index (GSI):**
- Different partition key (and optionally sort key).
- Can be added anytime.
- Own WCU/RCU (or on-demand).
- Max 20 per table (soft limit).
- **Eventually consistent only.**

**Exam picker:** need a new PK dimension → GSI. Need a second SK on the same PK → LSI. LSI decision must be made at table creation.

### DynamoDB Streams

Change log per table. Retention 24 hours. Trigger Lambda or connect to Kinesis Data Streams (via Kinesis adapter).

Used for: CDC, cross-region replication, denormalization to search indexes, cache invalidation.

### Global Tables

Multi-region, multi-active replication. Sub-second replication. Conflict resolution: last-writer-wins by timestamp.

### DAX (DynamoDB Accelerator)

In-memory write-through cache for DynamoDB. Microsecond latency for reads. Sits in your VPC.

### TTL (Time To Live)

Auto-delete items after epoch timestamp. Great for session data, ephemeral events. Deletions **do** flow through DynamoDB Streams (as a `REMOVE` with `userIdentity = 'dynamodb.amazonaws.com'`).

### DynamoDB partition-key design principles

Avoid hot partitions:
- **High-cardinality PK** — random UUIDs, hashed values.
- **Composite PK** — e.g., `user_id#YYYY-MM-DD`.
- **Sharding hot keys** — append random suffix `_0` to `_9`, distribute across shards.
- **Adaptive capacity** — DynamoDB automatically rebalances hot partitions, but only up to a point.

### PartiQL

SQL-like query language over DynamoDB. Nice for exploration; less efficient than native API.

---

## §4 — Open Table Formats + S3 Tables (Skill 2.1.7)

Iceberg is the AWS-recommended open table format. Big new area on the exam.

Cross-read: [Table Formats](../08-table-formats.md).

### Apache Iceberg on AWS — the three flavors

**1. Iceberg on S3 with Glue Catalog** — the classic. You write Iceberg tables to S3, register in the Glue Catalog, query with Athena / EMR / Redshift Spectrum. **You manage compaction, snapshot expiration, and file management.**

**2. Amazon S3 Tables (new, in-scope)** — managed Iceberg. AWS runs compaction, snapshot expiration, and orphan file cleanup automatically. Ships with a per-bucket **table bucket** type separate from general-purpose S3 buckets. **Highly recommended by AWS on the exam for new Iceberg workloads.**

**3. Iceberg in Redshift** — Redshift can read Iceberg tables via Spectrum and write Iceberg via `CREATE EXTERNAL TABLE ... TBLPROPERTIES ('table_type'='ICEBERG')`.

### Why Iceberg wins on AWS

- **ACID** transactions over S3.
- **Schema evolution** — add / rename / drop columns safely.
- **Partition evolution** — change partitioning without rewriting history.
- **Time travel** — query as of any snapshot.
- **Hidden partitioning** — `PARTITIONED BY days(ts)` — users don't need to know the partition column.
- **Multiple engines** read/write the same table (Athena, EMR, Redshift, Glue, Spark).

### Common Iceberg operations

```sql
-- Athena Iceberg
CREATE TABLE mydb.orders (
  order_id string,
  ts timestamp,
  amount double
)
PARTITIONED BY (day(ts))
LOCATION 's3://.../orders/'
TBLPROPERTIES ('table_type'='ICEBERG', 'format'='parquet');

-- Time travel
SELECT * FROM mydb.orders FOR TIMESTAMP AS OF TIMESTAMP '2026-06-01 00:00:00';
SELECT * FROM mydb.orders FOR VERSION AS OF 12345;

-- Merge (upsert)
MERGE INTO mydb.orders o
USING new_orders n ON o.order_id = n.order_id
WHEN MATCHED THEN UPDATE SET amount = n.amount
WHEN NOT MATCHED THEN INSERT (order_id, ts, amount) VALUES (n.order_id, n.ts, n.amount);
```

### Compaction & snapshot management

- **`OPTIMIZE` (Athena)** — rewrites small files into bigger ones.
- **`VACUUM`** — expires old snapshots (careful — destroys time-travel history past retention).
- **S3 Tables** — handles all this automatically. **This is why S3 Tables is the recommended answer for new workloads.**

---

## §5 — Data Cataloging (Task 2.2)

### AWS Glue Data Catalog

Hive-compatible metastore. Central metadata for:
- Athena queries
- EMR jobs
- Redshift Spectrum
- Glue ETL
- Lake Formation permissions

Objects: **databases, tables, partitions, table versions**. Backed by DynamoDB internally.

### Glue Crawlers

Scan S3 (or JDBC sources) → infer schema → create/update Glue table.

- **Schedule** — cron-based.
- **Custom classifiers** — override built-in schema inference for weird formats.
- **Partition detection** — auto-adds new partitions.
- **Table update behavior** — `LOG`, `UPDATE_IN_DATABASE`, `MERGE_NEW_COLUMNS`.

**Trap:** crawlers cost money and take time on huge datasets. For known-partition-layout data, use **Athena partition projection** (Domain 3) instead — no crawler needed.

### AWS Glue schema registry

Central schema store for **Kafka / Kinesis / MSK Connect** payloads. Supports Avro, JSON Schema, Protobuf. Versioned + compatibility rules (backward, forward, full).

### Amazon SageMaker Catalog (business data catalog)

Newer, part of SageMaker Unified Studio. Business-friendly catalog with governance, lineage, glossary. Distinct from Glue Data Catalog (technical). Expect a question distinguishing them.

### Apache Hive Metastore

Alternative to Glue Data Catalog. Used on EMR by default; you can point EMR at Glue Catalog instead.

---

## §6 — Data Lifecycle (Task 2.3)

### S3 Lifecycle policies

Rules per bucket/prefix that transition or expire objects:

```yaml
Transition:
  - After 30 days: Standard-IA
  - After 90 days: Glacier Instant Retrieval
  - After 365 days: Glacier Deep Archive
Expiration:
  - After 7 years: delete
```

Cost-optimization patterns:
- **Hot → Cold ladder** based on last-access date (use Intelligent-Tiering if you don't want to write policy).
- **Deep Archive for compliance** — 7+ year retention with rare access.
- **Incomplete multipart upload cleanup** — free savings (Lifecycle rule).

### S3 versioning + MFA delete

- **Versioning** — every overwrite keeps the old version. Enables rollback.
- **MFA delete** — requires MFA to permanently delete versioned objects. Rarely used but on exam.

### S3 Object Lock

WORM (write-once-read-many). Two modes:
- **Governance** — special IAM permission can override.
- **Compliance** — nobody can override, not even root, until retention expires.

For regulated data (SEC, HIPAA compliance archives) → Compliance mode.

### DynamoDB TTL

Auto-delete items after epoch timestamp. Async, no cost. Deletions flow through Streams.

### Redshift storage lifecycle

- **UNLOAD to S3** — offload cold data.
- **Redshift Spectrum** — query it back from S3 without loading.

### Backup patterns

- **AWS Backup** — centralized policy-based backup across RDS, DynamoDB, EFS, EBS, S3.
- **Point-in-time recovery (PITR)** — DynamoDB (35 days), Aurora (35 days), RDS (up to 35 days).
- **Cross-region backup** — for DR.

---

## §7 — Schema Design and Evolution (Task 2.4)

### Redshift schema patterns

Cross-read: [Data Modeling](../02-data-modeling.md).

- **Star schema** — denormalized dimensions + fact tables. Standard for analytics.
- **SORTKEY on the fact table's most-filtered column** (usually date).
- **DISTKEY on the highest-cardinality join column** (usually `user_id` or `order_id`).
- **`ALL` distribution for small dimensions** (< a few million rows).

### DynamoDB schema — single-table design

DynamoDB's single-table design pattern: pack multiple entity types into one table using **PK / SK overloading**.

Example: users + orders + order_items in one table.

| PK | SK | Attributes |
|---|---|---|
| `USER#123` | `PROFILE` | name, email, ... |
| `USER#123` | `ORDER#2026-01-15#001` | amount, status |
| `USER#123` | `ORDER#2026-01-15#001#ITEM#SKU-A` | qty, price |

Query patterns:
- Get user + all orders: `PK=USER#123` (SK starts-with any).
- Get user's orders in date range: `PK=USER#123 AND SK BETWEEN ORDER#2026-01-01 AND ORDER#2026-01-31`.

This is a deeper DynamoDB pattern — Alex DeBrie's *The DynamoDB Book* is the reference.

### Schema conversion

- **AWS DMS Schema Conversion** — replaces deprecated AWS SCT. Converts source-DB DDL to target-DB DDL. Support: Oracle / SQL Server / MySQL / Postgres / DB2 → various targets including Aurora / Redshift.

### Data lineage

- **SageMaker ML Lineage Tracking** — for model + data lineage in ML pipelines.
- **SageMaker Catalog** (Unified Studio) — business lineage tracking.
- **Marquez / OpenLineage** — open source, common in industry. Not AWS-native but interoperable.

Cross-read: [Governance & Lineage](../14-governance-lineage-observability.md).

### Vectorization (Skill 2.4.6)

Turn text → embeddings for RAG. AWS answer:
- **Bedrock Titan Embeddings** or Cohere Embeddings via Bedrock.
- Store vectors in **Aurora PostgreSQL with pgvector** (HNSW index) or **OpenSearch Serverless** (k-NN with HNSW).

Cross-read: [RAG Fundamentals](../../ai/rag/01-rag-fundamentals.md) · [Storage & Vector Search Structures](../../python/13-storage-and-vector-structures.md).

### Vector index types (Skill 2.1.8)

**HNSW (Hierarchical Navigable Small Worlds)** — graph-based ANN. Expected O(log n × d) query. High recall (95%+). Default in Aurora pgvector, OpenSearch k-NN.

**IVF (Inverted File)** — cluster then search top-`nprobe` centroids. Trades recall for speed at scale. Often combined with PQ for compression (IVF-PQ).

**Flat (brute-force KNN)** — exact, O(n × d). Only viable for small n.

**DiskANN** — SSD-resident graph for billion-scale.

**Exam patterns:**
- "Sub-100ms vector search alongside relational data" → Aurora PostgreSQL + pgvector + HNSW.
- "Ultra-fast key-value + vector" → MemoryDB (limited vector support) or OpenSearch Serverless.
- "Managed RAG with vector store" → Bedrock Knowledge Base (chooses OpenSearch / Aurora / Pinecone as vector store).

---

## Common Exam Traps

**Q: "OLTP with strict ACID + JSON columns + 100k TPS"**
A: Aurora (Postgres or MySQL). Not DynamoDB (weaker consistency choices), not Redshift (analytical).

**Q: "Analytical warehouse with unpredictable spiky query load"**
A: Redshift Serverless. (Not provisioned RA3 — that's for predictable.)

**Q: "Join massive S3 data with Redshift dims without loading it"**
A: Redshift Spectrum.

**Q: "Sub-ms KV cache in front of DynamoDB"**
A: DAX. (ElastiCache/MemoryDB also fast but require app-managed caching.)

**Q: "Vector search integrated with SQL data in RDS-family"**
A: Aurora PostgreSQL + pgvector.

**Q: "Cheapest storage for 7-year compliance archive, rare access"**
A: S3 Glacier Deep Archive.

**Q: "New Iceberg table with automatic compaction"**
A: S3 Tables.

**Q: "Graph queries across social connections"**
A: Neptune.

**Q: "MongoDB-compatible for existing MongoDB app"**
A: DocumentDB.

**Q: "Managed Cassandra"**
A: Amazon Keyspaces.

**Q: "Full-text + vector hybrid search"**
A: OpenSearch Service (or Aurora + pgvector if simpler).

**Q: "Business catalog with glossary + governance"**
A: SageMaker Catalog. (Glue Data Catalog is technical.)

---

## Concept-Check (self-quiz)

1. Redshift DISTKEY `KEY` vs `ALL` — when each?
2. Which SORTKEY variant is default and why?
3. RA3 vs Serverless Redshift — key differences?
4. What does Redshift Spectrum let you do that native Redshift can't?
5. DynamoDB GSI vs LSI — three differences.
6. What's a hot partition and how do you prevent it?
7. DynamoDB TTL — does deletion flow through Streams?
8. Iceberg on S3 vs S3 Tables — what's different?
9. Glue Data Catalog vs Hive Metastore — one difference.
10. Glue Data Catalog vs SageMaker Catalog — one difference.
11. S3 Object Lock — Governance vs Compliance mode?
12. HNSW vs IVF — one advantage each.
13. Aurora + pgvector — when do you pick this over OpenSearch?
14. AWS Backup — what does it centralize?
15. DMS Schema Conversion — what did it replace?

---

## Resources

- [Redshift Best Practices](https://docs.aws.amazon.com/redshift/latest/dg/best-practices.html)
- [DynamoDB Best Practices](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/best-practices.html)
- [Iceberg on AWS](https://docs.aws.amazon.com/athena/latest/ug/querying-iceberg.html)
- [S3 Tables](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables.html)
- [Aurora pgvector guide](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.VectorDB.html)
- [Alex DeBrie — The DynamoDB Book](https://www.dynamodbbook.com/) (paid but definitive).

---

*Next: [Domain 3 — Data Operations and Support](./03-domain3-operations-and-support.md).*
