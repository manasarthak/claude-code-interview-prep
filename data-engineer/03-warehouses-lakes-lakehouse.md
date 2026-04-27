# Warehouses, Lakes, and Lakehouses — Storage Architecture

**Phase:** 1 (Foundations)
**Difficulty progression:** Beginner → Intermediate → Advanced
**Last updated:** April 24, 2026
**Related:** [Data Modeling](./02-data-modeling.md) · [Open Table Formats](./08-table-formats.md) · [AWS Data Stack](./12-aws-data-stack.md) · [Partitioning & Performance](./13-partitioning-performance.md)

---

## Why This Topic Matters

Every architecture decision in DE depends on the answer to "where does the data live and in what shape?" Interviewers test this not because they need a textbook recital — but to see whether you understand the *trade-offs* between warehouse, lake, and lakehouse approaches.

---

## BEGINNER — The Three Storage Paradigms

### Data Warehouse (the original)

A managed analytical database optimized for SQL queries over structured data. Storage and compute were historically tightly coupled.

**Examples:** Snowflake, Amazon Redshift, Google BigQuery, Azure Synapse, Teradata, Databricks SQL Warehouse.

**Strengths:**
- Fast SQL on structured data
- ACID transactions
- Mature governance, access control, BI tooling
- Performance tuning is well-understood

**Weaknesses:**
- Historically rigid schemas (less true now)
- Loading semi/unstructured data was painful
- Vendor lock-in: proprietary file formats, vendor-specific SQL dialects
- Costly at scale, especially compute coupled with storage

### Data Lake

Cheap, scalable object storage holding files in open formats. The "schema-on-read" idea — store first, structure later.

**Examples:** S3, GCS, Azure Data Lake Storage (ADLS Gen2), HDFS.

**File formats:** Parquet (columnar, the standard), ORC (columnar, Hadoop-era), Avro (row, streaming-friendly), JSON / CSV (don't do it for analytics).

**Strengths:**
- Stupidly cheap storage
- Any data type — structured, semi, unstructured
- Decoupled from compute — bring any engine (Spark, Athena, Trino, Spark, EMR)
- No vendor lock-in on storage

**Weaknesses (the original "data swamp" problem):**
- No transactions — concurrent writers corrupt files
- No schema enforcement — silent drift
- No upserts/deletes without rewriting whole files
- Slow metadata operations on large catalogs
- Hard to manage updates, GDPR deletes, etc.

### Lakehouse — The Synthesis

Open table formats (Iceberg, Delta Lake, Hudi) bolt warehouse-like guarantees on top of lake storage. You get cheap, open object storage *plus* ACID transactions, schema evolution, time travel, and updates/deletes. Multiple engines can read/write the same tables.

**Examples of the architecture:** Databricks (Delta), Snowflake on Iceberg, AWS Glue + Iceberg, BigQuery + Iceberg.

**Why it took over:** Decoupled compute from storage *without* giving up the ACID guarantees that made warehouses useful. Lets organizations keep one copy of the data and pick the best engine per workload.

See [08-table-formats.md](./08-table-formats.md) for the deep dive on Iceberg/Delta/Hudi.

### One-Table Comparison

| Dimension | Warehouse | Lake | Lakehouse |
|---|---|---|---|
| Storage | Managed (sometimes proprietary) | Object storage, open formats | Object storage + open table format |
| Compute | Tightly bundled (mostly) | BYO engine | BYO engine |
| Transactions | ✅ | ❌ | ✅ |
| Schema enforcement | ✅ | ❌ | ✅ |
| Schema evolution | Limited | N/A | ✅ (varies by format) |
| Updates / deletes | ✅ | ❌ (rewrite) | ✅ |
| Time travel | Limited | ❌ | ✅ |
| ML/AI workloads | Awkward | Native | Native |
| Vendor lock-in | High | Low | Low (mostly) |
| Cost | High | Low | Low–medium |

---

## INTERMEDIATE — Columnar Storage and Why It Matters

### Row vs Columnar Storage

A row store keeps all the values for one row together; a column store keeps all the values for one column together.

```
Row layout:    [r1c1, r1c2, r1c3] [r2c1, r2c2, r2c3] [r3c1, r3c2, r3c3]
Column layout: [r1c1, r2c1, r3c1] [r1c2, r2c2, r3c2] [r1c3, r2c3, r3c3]
```

| | Row | Columnar |
|---|---|---|
| Best for | Many small reads/writes of full rows | Scanning subsets of columns over many rows |
| Compression | Mediocre (mixed types per page) | Excellent (one type per page) |
| `SELECT *` | Cheap | Expensive (must read all column files) |
| `SELECT col1` over millions of rows | Expensive (reads everything) | Cheap (reads only col1) |
| Examples | Postgres, MySQL, MongoDB | Parquet, ORC, Snowflake, Redshift, BigQuery |

**Why analytics loves columnar:**
- Most analytical queries touch a small subset of columns
- Like data compresses well — Parquet typically gets 5–10× compression on real data
- Vectorized execution: SIMD-friendly because column data is contiguous

### Parquet — the De Facto Lake Format

Parquet is a columnar format on disk:

```
File
├── Row group 1 (default ~128MB)
│   ├── Column chunk col_A (with min/max stats, encoding)
│   ├── Column chunk col_B
│   └── ...
├── Row group 2
└── Footer (schema, row group metadata, statistics)
```

Critical features:
- **Column chunks per row group** — read only the columns you need
- **Min/max statistics per row group** — engines skip row groups whose range can't match the filter ("predicate pushdown")
- **Dictionary encoding** for low-cardinality columns
- **Multiple compression codecs** — Snappy (fast, good ratio), Zstd (better ratio, slower), Gzip (legacy)

**Practical Parquet rules of thumb:**
- Aim for files of 128MB–1GB. Tiny files → metadata overhead; huge files → no parallelism.
- Sort within partitions on the column you filter on most (or use Z-order if your engine supports it). Better stats → better skipping.
- Use Snappy unless you're storage-bound, then Zstd.
- See [13-partitioning-performance.md](./13-partitioning-performance.md) for the full performance story.

### MPP — How Warehouses Scale

Modern warehouses are massively parallel (MPP). The architecture:

1. A query planner shards work across many compute nodes.
2. Each node processes a slice of the data.
3. Intermediate results are shuffled across the network as needed.
4. Results aggregated at the coordinator.

**Distribution strategies (Redshift terminology, but concepts apply broadly):**

| Strategy | When |
|---|---|
| `EVEN` | Default — round-robin distribution; fine when no obvious join key |
| `KEY` | Hash on a column — co-locate rows that frequently join |
| `ALL` | Replicate to every node — for small dim tables joined often |
| `AUTO` | Engine decides (Redshift, Snowflake's micro-partitioning) |

**Why this matters:** if your fact table is distributed by `customer_id` and your dim is distributed by `product_id`, joining them by `customer_id` requires shuffling the dim — slow at scale. Distribution choices are the OLAP equivalent of indexes.

### Snowflake's Micro-Partitions (Implicit Partitioning)

Snowflake auto-creates ~16MB compressed micro-partitions and tracks min/max per column per partition. Add a clustering key on the column you filter on, and Snowflake re-clusters in the background to maintain pruning quality. You don't pick distribution — it's managed.

### BigQuery's Slots

BigQuery is a *fully* serverless warehouse — you don't manage clusters. You pay per byte scanned (on-demand) or per slot-hours reserved. Partition + cluster on the columns you filter on; otherwise queries are expensive.

### Redshift's Architecture (Briefly)

- Leader node + compute nodes
- Each compute node has slices (CPU cores)
- DISTKEY controls which node a row lives on; SORTKEY orders rows within each slice
- RA3 nodes separate compute from storage (managed via Redshift Managed Storage), bringing it closer to Snowflake's model
- Spectrum lets you query S3 data in place — predecessor to today's lakehouse approach

---

## ADVANCED — Architecting the Modern Stack

### The Modern Data Stack

The popular reference architecture, ~2026:

```
Sources (apps, SaaS, events)
    │
    ▼
Ingest (Fivetran / Airbyte / custom Kafka / Kinesis Firehose)
    │
    ▼
Raw storage (S3 / Lake — Iceberg or Delta tables)
    │
    ▼
Transform (dbt + Spark/SQL warehouse engine)
    │
    ▼
Marts (warehouse — Snowflake / BigQuery / Redshift / Databricks SQL)
    │
    ▼
BI (Looker, Tableau, Hex, Mode), Reverse-ETL (Hightouch / Census), ML
```

This is the architecture interviewers will expect you to know cold. The key insight: **storage at the lake, compute fluid, transformation in dbt, marts where BI hits.**

### Medallion Architecture (Bronze / Silver / Gold)

Coined by Databricks but generic enough to apply anywhere:

| Layer | Contents | Audience |
|---|---|---|
| **Bronze** | Raw, append-only, source-faithful — JSON dumped exactly as received | DEs only — debugging, reprocessing |
| **Silver** | Cleaned, deduped, conformed types, joined to reference data | DEs + analytics engineers |
| **Gold** | Business-ready, denormalized, aggregated, KPI-friendly | Analysts, BI, ML, executives |

This structure:
- Makes reprocessing easy — you can always re-derive Silver/Gold from Bronze
- Separates the messy reality of source data from the cleanliness of business semantics
- Maps cleanly to dbt staging / intermediate / mart models

### Lambda vs Kappa Architectures

**Lambda** — historical pattern from Hadoop era:
- A *batch layer* computes accurate aggregates from full history (slow but correct)
- A *speed layer* computes approximate aggregates from streaming data (fast but approximate)
- A *serving layer* merges them
- Two codebases doing the same thing → operational nightmare

**Kappa** — modern pattern:
- One streaming pipeline that handles both real-time and reprocessing (replay the log)
- One codebase
- Requires durable, replayable streams (Kafka) and stream processors (Flink, Spark Structured Streaming)

Most modern systems pick Kappa-flavored architectures. See [06-batch-vs-streaming.md](./06-batch-vs-streaming.md).

### Storage Costs and Compute Costs

| Cost type | Driver |
|---|---|
| Storage | Bytes stored, replication, snapshots, time travel retention |
| Compute | Bytes scanned (BigQuery), warehouse uptime (Snowflake), node hours (Redshift) |
| Egress | Cross-region or cross-cloud transfer (S3, BigQuery) |
| Operational | DEs to maintain pipelines, monitoring tools |

**The biggest single cost lever:** scan less. Partition + cluster + SELECT only the columns you need. A poorly partitioned 10TB table scanned per dashboard load can cost more than the engineers who built it.

### Choosing Among Warehouses

| Choice | Lean toward |
|---|---|
| Snowflake | Multi-cloud needs, advanced features (data sharing, clean rooms), strong ecosystem |
| BigQuery | GCP-native, ML integrations (BQML), serverless simplicity, very cheap on-demand for spiky workloads |
| Redshift | AWS-native, tight VPC/IAM integration, predictable pricing on RA3 |
| Databricks SQL | Already using Databricks for ML/Spark, want one platform |
| ClickHouse | Real-time analytics, sub-second queries, can self-host |
| DuckDB | Local/embedded analytics, single-machine workloads, dev experience |

**Honest take:** for an interview, you should be able to defend "we picked X because Y" with concrete trade-offs, not "X is the best." All of these are good choices in their niches.

### When NOT to Use a Warehouse

- Operational point lookups by ID — use a key-value store (DynamoDB, Redis) or OLTP DB
- Sub-100ms analytical lookups for app features — use a serving layer (DuckDB, ClickHouse, materialized views)
- Vector similarity for AI — use a vector DB (see [RAG Fundamentals](../ai/rag/01-rag-fundamentals.md))
- Time series at very high cardinality — consider Druid / Pinot / TimescaleDB

### Format Decisions in 2026

For new lakehouse builds:

| Need | Default |
|---|---|
| Columnar file format | **Parquet** (Snappy compression) |
| Open table format | **Apache Iceberg** for new builds (broad engine support, AWS investment) |
| Real-time CDC into the lake | **Hudi** (built for it) or Iceberg with regular merges |
| Locked into Databricks | **Delta Lake** (better integrations there) |
| Streaming-only formats | **Avro** for Kafka, **Protobuf** if you control producers |

See [08-table-formats.md](./08-table-formats.md) for the comparison.

---

## Worked Mental Model: "Why Doesn't My Warehouse Just Read S3?"

This is a classic interview probe, designed to check that you understand the gap that lakehouse table formats fill.

If you ask Athena to "select count(*) from a thousand-Parquet-file directory," it has to:
1. List every file (S3 LIST is slow at scale).
2. Read each Parquet footer to figure out row counts and column ranges.
3. Plan reads, possibly of all files.

There's no transaction log, so:
- Concurrent writers can produce inconsistent reads (half-written files visible)
- No way to atomically replace a partition
- No "what was the table 3 hours ago" semantics
- Updates require rewriting whole partitions

A table format (Iceberg/Delta/Hudi) adds a manifest of "the canonical set of files at version N, with stats." Reading the manifest is fast; reading the files is parallelizable; replacing files is atomic via writing a new manifest.

That's the whole game.

---

## Interview-Ready Cheat Sheet

**"Difference between a data lake and a data warehouse?"** Lake: cheap object storage, open formats, BYO compute, no transactions natively. Warehouse: managed analytical DB, ACID, governance, tightly bundled compute (historically). Lakehouse bridges them via open table formats.

**"Why columnar storage?"** Most analytical queries touch a few columns over many rows. Columnar gives you (a) read only the columns you need, (b) much better compression, (c) vectorized execution.

**"What is a lakehouse?"** Lake storage + open table format (Iceberg/Delta/Hudi) that adds ACID, schema evolution, time travel, and updates/deletes. You get warehouse semantics on cheap object storage with engine flexibility.

**"Bronze/Silver/Gold?"** Raw / cleaned-conformed / business-ready. Each layer can be reprocessed from the one below. Maps to dbt staging / intermediate / mart.

**"Why does my warehouse query scan 100GB when I asked for one column?"** Probably partition isn't pruning (no filter on partition column or filter using a function), or table isn't clustered/sorted on the predicate column, or you used `SELECT *`. See [13-partitioning-performance.md](./13-partitioning-performance.md).

**Quick trade-off pairs:**
- Warehouse vs lake: managed convenience vs flexibility/cost
- Lake vs lakehouse: cheap+open vs cheap+open+ACID
- Lambda vs Kappa: dual-codebase batch+stream vs single streaming pipeline
- Snowflake vs BigQuery vs Redshift: multi-cloud vs GCP-native serverless vs AWS-native predictable

---

## Resources & Links

- [Apache Parquet documentation](https://parquet.apache.org/docs/) — file format internals
- [Snowflake Architecture overview](https://docs.snowflake.com/en/user-guide/intro-key-concepts) — micro-partitions explained
- [Databricks Lakehouse paper (CIDR 2021)](https://www.cidrdb.org/cidr2021/papers/cidr2021_paper17.pdf) — origin of the lakehouse concept
- [Designing Data-Intensive Applications (Kleppmann)](https://dataintensive.net/) — the canonical book; chapters 3, 10, 11 are gold
- [Modern Data Stack overview — dbt blog](https://www.getdbt.com/blog/future-of-the-modern-data-stack) — current architecture trends

*Next: [Open Table Formats](./08-table-formats.md) for the Iceberg/Delta/Hudi deep dive.*
