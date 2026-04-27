# Open Table Formats — Iceberg, Delta Lake, and Hudi

**Phase:** 1 (Foundations)
**Difficulty progression:** Beginner → Intermediate → Advanced
**Last updated:** April 24, 2026
**Related:** [Warehouses & Lakehouses](./03-warehouses-lakes-lakehouse.md) · [Spark Fundamentals](./07-spark-fundamentals.md) · [Partitioning & Performance](./13-partitioning-performance.md) · [CDC](./09-cdc.md)

---

## Why Table Formats Exist

A bare directory of Parquet files isn't a table. It's a heap of files. Concurrent writes corrupt readers, you can't update or delete a row without rewriting whole partitions, schema changes break consumers, and "what was this table at 9am yesterday" is unanswerable. Open table formats — Apache Iceberg, Delta Lake, Apache Hudi — bolt warehouse-like guarantees onto cheap object storage.

If you only learn one Tier 2 topic deeply for AWS DE interviews in 2026, **make it Iceberg**. AWS has bet heavily on it (Glue, Athena, Redshift Spectrum, S3 Tables).

---

## BEGINNER — What a Table Format Adds Over Plain Parquet

| Capability | Plain Parquet | Open Table Format |
|---|---|---|
| ACID writes | ❌ | ✅ |
| Concurrent readers/writers safely | ❌ | ✅ |
| Schema evolution (add/drop/rename) | Limited | ✅ |
| Partition evolution | ❌ | ✅ (Iceberg, Hudi) |
| Updates / deletes / merges | Rewrite partition | Native (`UPDATE`, `DELETE`, `MERGE`) |
| Time travel (query as of T) | ❌ | ✅ |
| Small files compaction | Manual | Native (`OPTIMIZE` / `compaction`) |
| Multi-engine reads/writes | ❌ | ✅ (open spec) |
| Hidden partitioning | ❌ | ✅ (Iceberg) |

### How They Work Conceptually (All Three Are Similar)

```
Table = (Data files, columnar Parquet) + (Metadata layer that tracks "the canonical set of files at version N")
```

The metadata layer is the magic. Reading a table = read the metadata to get the file list, then read the files. Writing = produce new files, then atomically commit a new metadata version pointing at them. The atomic commit is what gives you ACID semantics on object storage.

Each format implements this with slightly different layouts and trade-offs.

---

## INTERMEDIATE — The Three Formats Compared

### Apache Iceberg

Originally from Netflix; now the broadest adoption.

**Architecture:**
```
Catalog (e.g., AWS Glue, Hive Metastore, REST catalog, Nessie)
  │ points to current table metadata file
  ▼
metadata.json (snapshot v17)
  │ points to manifest list
  ▼
manifest list (one entry per snapshot)
  │ points to manifest files
  ▼
manifest files (each lists data files + per-file stats)
  │
  ▼
data files (Parquet/ORC/Avro)
```

**Strengths:**
- **Hidden partitioning** — declare partition transforms (`days(ts)`, `bucket(16, user_id)`); queries don't need to filter on a separate partition column
- **Partition evolution** — change partitioning over time without rewriting old data
- **Schema evolution by ID** — rename columns safely (column tracked by ID, not name)
- **Branching and tagging** (Iceberg 1.0+) — like git for tables
- Strong support across engines: Spark, Trino, Athena, Snowflake, BigQuery, Flink, Dremio
- AWS native: Glue Iceberg tables, Athena, Redshift, S3 Tables

**Weaknesses:**
- More complex than Delta if you only run Spark
- Streaming/CDC story improving but historically weaker than Hudi

### Delta Lake

Originally from Databricks; now open-sourced under the Linux Foundation.

**Architecture:**
```
Table directory
├── _delta_log/
│   ├── 00000000000000000000.json   ← every commit is a JSON entry
│   ├── 00000000000000000001.json
│   ├── 00000000000000000010.checkpoint.parquet  ← periodic checkpoint
│   └── ...
└── *.parquet (data files)
```

The `_delta_log` is an append-only log of operations (`AddFile`, `RemoveFile`, `Metadata`, etc.). Periodic checkpoints summarize the log for fast reads.

**Strengths:**
- **Best Spark/Databricks experience** by far — first-class integration
- **Liquid clustering** (Delta 3+) — incremental clustering without rewrite-the-world re-partition
- **Change Data Feed** (CDF) — query "what changed in this table between v10 and v15"
- **Z-ordering** for multi-column data skipping
- Strong streaming support (Spark Structured Streaming + Delta is well-trodden)

**Weaknesses:**
- Engine support outside Spark is real but less first-class than Iceberg's
- Historically tied to Databricks, though OSS Delta is broadly adopted
- Some advanced features are Databricks-specific (Photon, Unity Catalog)

### Apache Hudi

Originally from Uber.

**Two table types:**
- **Copy-on-Write (CoW)** — like Iceberg/Delta defaults; updates rewrite affected files
- **Merge-on-Read (MoR)** — base files (Parquet) + delta logs (Avro). Reads merge them on the fly. Faster ingest, slower reads, requires periodic compaction.

**Strengths:**
- **Best in class for streaming CDC ingestion** — built for it
- **Record-level indexes** — fast point lookups by primary key
- **Incremental queries** — "give me everything that changed since commit X"
- **Concurrency control with timestamp ordering** — better than the others for high-concurrency writes

**Weaknesses:**
- More operational complexity (CoW vs MoR; clustering services; compaction services)
- Smaller community than Iceberg / Delta
- AWS support exists but less first-class

### Side-by-Side Comparison Table

| Feature | Iceberg | Delta Lake | Hudi |
|---|---|---|---|
| Origin | Netflix | Databricks | Uber |
| Best engine | Spark, Trino, Athena, Flink | Spark / Databricks | Spark, Flink |
| Hidden partitioning | ✅ | ❌ | ❌ |
| Partition evolution | ✅ | ❌ | Limited |
| Schema evolution | ✅ (by column id) | ✅ | ✅ |
| Time travel | ✅ | ✅ | ✅ |
| MERGE / UPDATE / DELETE | ✅ | ✅ | ✅ |
| Streaming sink | ✅ | ✅ (best) | ✅ (best for CDC) |
| Z-order / clustering | Sort + Z-order | Z-order, Liquid clustering | Clustering |
| Branching/tagging | ✅ | Limited | ❌ |
| AWS native support | ✅ Glue/Athena/Redshift | ✅ EMR/Athena/Glue | ✅ EMR/Glue |
| Multi-engine writes | ✅ | ✅ (improving) | ✅ |
| Default in 2026 | **Recommended for new builds** | **Best on Databricks** | **Best for high-velocity CDC** |

### Common Operations (Spark SQL Examples)

These look nearly identical across formats — that's the point.

```sql
-- create
CREATE TABLE catalog.db.events (
  event_id STRING,
  user_id STRING,
  ts TIMESTAMP,
  amount DECIMAL(10,2)
) USING ICEBERG
PARTITIONED BY (days(ts));

-- insert
INSERT INTO catalog.db.events VALUES (...);

-- update
UPDATE catalog.db.events SET amount = 0 WHERE event_id = 'bad-row';

-- delete
DELETE FROM catalog.db.events WHERE ts < '2020-01-01';

-- merge (the workhorse for upserts and CDC)
MERGE INTO catalog.db.events t
USING staging.events s
ON t.event_id = s.event_id
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *;

-- time travel
SELECT * FROM catalog.db.events FOR VERSION AS OF 42;
SELECT * FROM catalog.db.events FOR TIMESTAMP AS OF '2025-04-01';

-- compaction
CALL system.rewrite_data_files('catalog.db.events');  -- Iceberg
OPTIMIZE catalog.db.events ZORDER BY (user_id);       -- Delta
```

---

## ADVANCED — Production Patterns

### Compaction and the Small-Files Problem

Streaming + frequent appends produce many small files. Each file has metadata overhead and serializes a task. Without compaction, query performance degrades over time.

**Compaction operations:**
- **Iceberg**: `CALL system.rewrite_data_files(...)` — rewrites tiny files into target-size ones
- **Delta**: `OPTIMIZE` — bin-packs files; `OPTIMIZE ZORDER BY (cols)` adds Z-order data skipping
- **Hudi MoR**: scheduled compaction merges delta logs into base files

**Schedule** compaction as a regular job (hourly / daily / per-write threshold). Without it, your beautiful lakehouse becomes a small-files swamp.

### Snapshot Retention and Cleanup

Every commit creates a snapshot. Snapshots reference data files. Old snapshots keep old data files alive (consuming storage), but enable time travel.

| Operation | Iceberg | Delta |
|---|---|---|
| Expire old snapshots | `expire_snapshots` | `VACUUM` |
| Default retention | None (must configure) | 7 days |
| Compute "orphan" data files | `remove_orphan_files` | `VACUUM` (with cleanup) |

**Set retention policies explicitly.** Many production lakes accidentally double their storage cost by never expiring snapshots.

### Hidden Partitioning (Iceberg's Killer Feature)

Without hidden partitioning, every query that wants partition pruning has to filter on the literal partition column:

```sql
-- traditional (Hive-style)
WHERE event_date = '2025-04-15' AND ts >= '...'  -- must filter both
```

With Iceberg hidden partitioning:
```sql
CREATE TABLE events (ts TIMESTAMP, ...) PARTITIONED BY (days(ts));

-- queries don't need a separate partition col
WHERE ts >= '2025-04-15' AND ts < '2025-04-16'  -- prunes automatically
```

This eliminates a whole class of "I forgot to filter on the partition column" bugs.

### Partition Evolution

You partitioned by `days(ts)`. Two years in, daily partitions are too granular. With Iceberg:

```sql
ALTER TABLE events SET PARTITIONING (months(ts));
```

New writes go to monthly partitions; old data stays in daily partitions; queries handle both transparently. Delta and Hudi require rewriting.

### Schema Evolution

All three formats support adding columns. The trickier ops:

| Operation | Iceberg | Delta | Hudi |
|---|---|---|---|
| Add column | ✅ | ✅ | ✅ |
| Drop column | ✅ | ✅ (since 1.x) | ✅ |
| Rename column | ✅ (by ID) | ✅ (since 2.x) | Limited |
| Change column type (compatible) | ✅ | Limited | Limited |
| Reorder columns | ✅ | ✅ | Limited |

### Time Travel and Reproducibility

Time travel makes pipelines reproducible:

```sql
-- "rerun yesterday's job against yesterday's data"
INSERT OVERWRITE TABLE silver.daily_agg
SELECT date_trunc('day', ts), count(*)
FROM bronze.events FOR TIMESTAMP AS OF '2025-04-23 00:00:00'
GROUP BY 1;
```

Use cases:
- Debug data anomalies — "what did the table look like before the bad write"
- Reproducible ML training datasets
- Audit / compliance — "show me the data as of this date"
- Quick rollback — `RESTORE` / re-pin to previous snapshot

### Streaming + Lakehouse — the Modern Pattern

```
Kafka / Kinesis  →  Flink / Spark Streaming  →  Iceberg / Delta / Hudi tables
                                                        │
                                                        ▼
                                                Same tables read by:
                                                  - dbt batch transforms
                                                  - ad-hoc Athena queries
                                                  - BI tools
                                                  - ML feature pipelines
```

This is the architecture that kills the Lambda dual-pipeline problem (see [03-warehouses-lakes-lakehouse.md](./03-warehouses-lakes-lakehouse.md)). One copy of the data, multiple engines.

**Iceberg + Flink** for streaming sink: Flink commits to Iceberg in transactional batches; readers see snapshot-consistent views; no half-written file corruption.

### Concurrency Control

| Format | Concurrency model |
|---|---|
| **Iceberg** | Optimistic concurrency on commit (compare-and-swap on metadata pointer); retries on conflict |
| **Delta** | Optimistic concurrency on `_delta_log`; retries on conflict |
| **Hudi** | Multi-versioned writes with timestamp ordering; can use ZK / lock provider for stricter ordering |

For concurrent writers (rare in practice for warehouse-style workloads, common for streaming + batch), Hudi's mechanisms are the most sophisticated. For typical "one batch writer + many readers," all three are fine.

### Catalog Choice (Iceberg-specific)

Iceberg needs a catalog to track which metadata file is current.

| Catalog | Notes |
|---|---|
| **AWS Glue Data Catalog** | Default for AWS shops; pairs with Athena/Redshift/EMR |
| **REST Catalog** | Open spec, vendor-neutral; multi-engine friendly |
| **Hive Metastore** | Legacy compat; works |
| **Nessie** | Git-like branching for tables |
| **Snowflake / Databricks Unity** | If you're warehouse-led |

**Modern recommendation:** REST catalog (Tabular, Polaris, or self-hosted) for vendor neutrality; Glue if you're committed to AWS.

### CDC into Lakehouse

The most-asked production scenario: keep an analytics copy of an OLTP database fresh, with low latency. See [09-cdc.md](./09-cdc.md).

```
Postgres → Debezium → Kafka → [stream processor] → MERGE INTO iceberg.silver.users
```

**Hudi** is purpose-built for this (CoW with primary-key indexes; MoR for higher ingest throughput). **Iceberg** with `MERGE INTO` works well too, especially with newer V2 row-level deletes. **Delta** with structured streaming + MERGE is canonical on Databricks.

---

## Worked Decision — "Which Format Should We Pick?"

You're asked: "We're building a new lakehouse on AWS. Iceberg, Delta, or Hudi?"

**Default answer: Iceberg.** Reasoning:
- AWS has invested heavily in Iceberg (S3 Tables, Glue, Athena, Redshift integration)
- Broadest engine support (Spark, Trino, Snowflake, BigQuery, Flink)
- Hidden partitioning + partition evolution are real productivity wins
- Most active community; vendor neutrality
- The ecosystem in 2026 is converging on Iceberg as the de facto standard

**Pick Delta when:**
- You're committed to Databricks (Delta is significantly better integrated there)
- You want Photon, Liquid clustering, Unity Catalog features

**Pick Hudi when:**
- High-throughput streaming CDC is your primary workload
- You need record-level indexes for fast point lookups
- You're already operating it successfully

This kind of trade-off-led decision is what mid-level interviews look for. "Always pick X" is a junior answer.

---

## Interview-Ready Cheat Sheet

**"What's a lakehouse / table format?"** A metadata layer (manifests, transaction log) on top of Parquet files that adds ACID, schema evolution, time travel, and updates/deletes. Multiple engines can read/write the same tables.

**"Iceberg vs Delta vs Hudi?"** Iceberg: most engine-agnostic, hidden + evolving partitioning, AWS-blessed. Delta: best on Databricks, mature streaming + Z-order. Hudi: built for high-velocity streaming CDC, record-level indexes.

**"What's hidden partitioning?"** Iceberg's feature: declare partition transforms on raw columns (e.g., `days(ts)`); queries filter on the raw column and pruning happens automatically. Eliminates a whole class of missed-partition-filter bugs.

**"How does time travel work?"** Each commit produces a new snapshot referencing a set of data files. Old snapshots remain queryable until expired. Read at version N or timestamp T resolves to the snapshot's file list.

**"What's the small-files problem?"** Streaming/frequent commits produce many tiny files; metadata overhead and per-task overhead degrade reads. Fix: scheduled compaction (`OPTIMIZE` / `rewrite_data_files`).

**"How do MERGE statements work?"** Logically: match on key, update or insert. Physically: scan target, identify rows to delete or update, write new files for affected rows + insert new ones, commit a new snapshot atomically.

**Quick trade-off pairs:**
- CoW vs MoR: read-fast/write-amplification vs write-fast/read-amplification
- Iceberg vs Delta: portability vs Databricks integration
- Compaction now vs later: query speed vs write throughput
- Time travel retention: queryability vs storage cost

---

## Resources & Links

- [Apache Iceberg docs](https://iceberg.apache.org/) — start here for new builds
- [Delta Lake docs](https://docs.delta.io/)
- [Apache Hudi docs](https://hudi.apache.org/)
- [Tabular blog — "Iceberg vs Delta vs Hudi"](https://tabular.io/blog/) — deep technical comparisons (Tabular was acquired by Databricks)
- [AWS S3 Tables (Iceberg-managed S3)](https://aws.amazon.com/s3/features/tables/) — AWS's bet on Iceberg
- [Onehouse blog](https://www.onehouse.ai/blog) — Hudi creators, balanced comparisons

*Next: [CDC](./09-cdc.md) — the most common reason to use these table formats in the first place.*
