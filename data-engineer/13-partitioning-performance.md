# Partitioning & Performance — Files, Pruning, and the Cost of Carelessness

**Phase:** 1 (Foundations)
**Difficulty progression:** Beginner → Intermediate → Advanced
**Last updated:** April 24, 2026
**Related:** [Spark Fundamentals](./07-spark-fundamentals.md) · [Open Table Formats](./08-table-formats.md) · [Warehouses & Lakehouses](./03-warehouses-lakes-lakehouse.md) · [SQL Deep Dive](./01-sql-deep-dive.md)

---

## Why This Topic Matters

90% of "my pipeline is slow / expensive" complaints trace back to bad file layout, bad partitioning, or queries that defeat pruning. Mid-level interviewers expect you to *think about partitioning* before they ask — and to articulate the trade-offs.

This file complements [Spark Fundamentals](./07-spark-fundamentals.md) with the storage-side optimization story.

---

## BEGINNER — File Sizes, Layout, and Pruning

### The Three Knobs You Actually Control

| Knob | What it controls |
|---|---|
| **File format** | Compression, encoding, predicate pushdown, columnar vs row |
| **File sizes** | Parallelism, metadata overhead |
| **Partition layout** | What can be pruned at planning time |

A mid-level engineer reasons about all three; a junior engineer reasons about none. This is the difference interviewers are testing for.

### File Format Choice — Always Parquet (For Analytics)

| Format | Use |
|---|---|
| **Parquet** | Default for analytics — columnar, compressed, predicate pushdown |
| **ORC** | Hadoop-era columnar; comparable to Parquet, less momentum |
| **Avro** | Row-oriented, schema-embedded; good for streams (Kafka) and ingest |
| **JSON / CSV** | Don't use for analytics. Slow, no schema, awful compression |
| **Iceberg/Delta/Hudi** | Not formats — table formats *over* Parquet (see [08-table-formats.md](./08-table-formats.md)) |

For Parquet specifically:

| Compression | Trade-off |
|---|---|
| **Snappy** | Fast decompress, decent ratio — the default |
| **Zstd** | Better ratio, slightly slower decompress — increasingly the new default |
| **Gzip** | Best ratio, slow decompress — legacy |
| **None** | Don't |

**Rule:** Parquet + Snappy or Zstd, end of discussion for new builds.

### File Size — The Goldilocks Rule

| Size | Problem |
|---|---|
| **Too small** (<10 MB) | Metadata overhead, one-task-per-file overwhelms scheduling, poor compression |
| **Too big** (>1 GB) | No parallelism, OOM risk, can't predicate-prune within the file |
| **Just right** (128 MB–1 GB) | Good parallelism + good compression + good metadata |

**Default targets in 2026:** 128–512 MB Parquet files. Spark's default `spark.sql.files.maxPartitionBytes = 128MB` aligns with this.

### The Small-Files Problem

The most common pipeline pathology. Symptoms:

- Streaming sink emits one file per micro-batch → millions of tiny files in S3
- Untuned `df.write.partitionBy("user_id")` → one file per user, often KB-sized
- Athena scans take 10× longer than expected
- Spark jobs spend more time on scheduling than computing
- S3 LIST operations alone become a cost line item

**Fixes:**
- Compaction jobs (`OPTIMIZE` in Delta, `rewrite_data_files` in Iceberg)
- `coalesce` / `repartition` before write
- Right-size streaming triggers (Spark Structured Streaming's `trigger=ProcessingTime('5 minutes')`)
- Avoid partitioning on high-cardinality columns

### Partitioning Basics

A **partition** in storage terms is a physical sub-directory or file group identified by a column value:

```
s3://bucket/table/event_date=2025-04-15/...
              event_date=2025-04-16/...
              event_date=2025-04-17/...
```

When a query filters `WHERE event_date = '2025-04-15'`, the engine reads only that partition's files. This is **partition pruning** — the foundation of warehouse/lake performance.

**Choosing a partition column:**
- High enough cardinality to be useful, low enough to avoid millions of partitions
- Filtered on by 80%+ of queries
- Stable (a column whose values don't change after write)
- Reasonable distribution (avoid hot partitions if downstream readers parallelize)

**Common good choices:** `event_date`, `event_year_month`, `region`, `tenant_id` (with care).

**Common bad choices:** `user_id` (too high cardinality), `id` (random hash, defeats clustering), `is_active` (too low cardinality).

### When Partition Pruning Breaks Silently

```sql
-- ❌ pruning lost: function on partition column
WHERE date_trunc('month', event_date) = '2025-04-01'

-- ✅ pruning works
WHERE event_date >= '2025-04-01' AND event_date < '2025-05-01'

-- ❌ pruning lost: implicit cast
WHERE event_date = '2025-04-15 00:00:00'  -- if event_date is DATE

-- ❌ pruning lost: filtering on the wrong column
WHERE ts = '2025-04-15'  -- if partition is event_date, not ts
```

This is interview gold. "Why did my query scan 10 TB?" → "I bet you wrapped the partition column in a function" or "you filtered on a non-partition column."

---

## INTERMEDIATE — Layout Strategies

### Time-Based Partitioning (the default)

```
s3://lake/events/event_date=2025-04-15/
                event_date=2025-04-16/
                ...
```

Pros:
- Most analytics filters by time
- Easy backfill (replace one partition)
- Easy retention (drop old partitions)

Cons:
- Single dimension; doesn't help non-time queries

For higher granularity (high-volume streams), partition by `event_date + hour`:

```
s3://lake/events/event_date=2025-04-15/hour=12/
```

But beware: partition by hour for low-volume streams → small-files problem.

### Multi-Level Partitioning

```
s3://lake/events/event_date=2025-04-15/region=us-east-1/...
```

Use only when:
- Both columns are filtered on most queries
- The combination doesn't explode partition count

**Anti-pattern:** partitioning by `event_date / hour / region / tenant_id / source` — millions of partitions, kills metadata operations, exhausts memory in catalog tools.

### Hidden Partitioning (Iceberg)

Iceberg lets you declare partition transforms:

```sql
CREATE TABLE events (
  event_id BIGINT,
  ts TIMESTAMP,
  user_id BIGINT,
  ...
) PARTITIONED BY (days(ts), bucket(16, user_id));
```

**Available transforms:**
- `years(ts)`, `months(ts)`, `days(ts)`, `hours(ts)` — time bucket
- `bucket(N, col)` — hash modulo N (uniform distribution)
- `truncate(N, col)` — string truncation

Queries filter on raw columns; pruning happens transparently. Big win over Hive-style explicit partition columns. See [08-table-formats.md](./08-table-formats.md).

### Bucketing

Bucketing distributes rows by hashed column value into a fixed number of buckets:

```sql
-- Spark / Iceberg
CLUSTERED BY (user_id) INTO 16 BUCKETS

-- Iceberg hidden partition
PARTITIONED BY (bucket(16, user_id))
```

Benefits:
- Hash-distributed, even with high cardinality
- Joins on the bucketing column avoid shuffle (if both sides are bucketed the same way)
- Avoids the "millions of partitions" problem when high-cardinality is needed for performance

Bucketing on `user_id` lets you query "all events for user 42" without scanning the whole table — the hash narrows it to one bucket.

### Within-File Sorting

After partitioning into a file group, *sort within the file* on the most-filtered column. This populates Parquet's row-group min/max statistics with tight ranges → predicate pushdown works.

```python
df.repartition("event_date").sortWithinPartitions("user_id").write...
```

Without sorting, min/max stats are wide ranges and pushdown can't skip row groups.

### Z-Ordering / Liquid Clustering (Delta)

For multi-dimensional pruning (filter on either `user_id` or `event_id`), single-column sort is suboptimal. Z-order interleaves bits of multiple columns to give partial pruning on each:

```sql
OPTIMIZE table_x ZORDER BY (user_id, event_id);
```

**Liquid clustering** (Delta 3+) is incremental Z-order — clustering happens as data is added, no need for the whole-table OPTIMIZE rewrite.

Iceberg has analogous "sort orders" (linear or Z-style), with engine-specific support.

### Bloom Filters and Other Indexes

Some formats support secondary indexes:

| Index | Use |
|---|---|
| **Min/max stats** (Parquet row group) | Range pruning — built-in, free |
| **Bloom filters** (Parquet row group) | Equality pruning on high-cardinality columns ("does any row have user_id=42?") |
| **Hudi record-level indexes** | Fast PK lookups for upserts |
| **Iceberg manifest stats** | File-level pruning before opening files |

For **point lookups** on a column not in the partition key, bloom filters can dramatically speed up "needle in haystack" queries.

### Compaction Strategy

Compaction merges small files into target-size ones. Production schedule:

| Cadence | What runs |
|---|---|
| **On every batch write** | Right-size files in this batch (often built-in) |
| **Hourly** | Compact streaming sink output for the last hour |
| **Daily** | Compact yesterday's partition (full rewrite) |
| **Weekly** | Z-order / liquid cluster for hot tables |
| **Monthly** | Re-evaluate partition strategy; expire old snapshots |

Without compaction, lakes accumulate small files until performance dies. With compaction, performance stays steady — but compaction itself uses compute, so it's a budget item.

---

## ADVANCED — Performance Investigation

### "My Query Is Slow" — A Mental Algorithm

1. **Compute resources or data layout?** Check engine-side metrics first (Spark UI, Athena query stats). If compute is throttled but layout is fine, scale compute.
2. **Pruning effective?** What percent of the table did the query scan? If it's 50% of a 10TB table for a query that should hit one day's data, partitioning is broken.
3. **File sizes reasonable?** If thousands of tiny files, it's small-files problem. Compact.
4. **Joins broadcasting where they could?** Look at the physical plan — small dim should be `BroadcastHashJoin`.
5. **Skew?** Check task duration distribution. One task taking 10× others is a red flag.
6. **Columns being pruned?** `SELECT *` reads all columns; explicit column list is much better on columnar.
7. **Statistics fresh?** Outdated stats lead to bad join order. Refresh table statistics.

### Cost Per Query — Reasoning Aloud

For mid-level interviews, be able to estimate query cost on the fly:

> "This is a 1 TB table with 365 daily partitions. The query filters to 7 days, so 7 partitions = ~20 GB. With Snappy Parquet, ~5 GB compressed scanned. On Athena at $5/TB, that's ~$0.025 per query. If 500 analysts run this 10× a day → $125/day."

This kind of order-of-magnitude reasoning is what separates mid-level from junior. You don't need to be exact — you need to *know how to think about it*.

### Common Performance Anti-Patterns

| Anti-pattern | Cost |
|---|---|
| `SELECT *` on columnar tables | Reads all columns; bills for all columns scanned |
| Partition by high-cardinality column | Millions of partitions, metadata explosion |
| Partition + small batches → tiny files | Small-files; metadata overhead |
| Function on filter column | Defeats pruning |
| Joining small dim by shuffling | Should be broadcast |
| No ORDER BY clustering for scan-pattern queries | Worse data skipping |
| Cache that isn't reused | Wasted memory |
| `count(distinct user_id)` over years of data | Expensive; consider HyperLogLog approximation |
| Recomputing the same expensive subquery | Materialize as table or CTE |

### Query Hints (Use Sparingly)

Most modern engines have decent optimizers. But sometimes hints are necessary:

```sql
-- Spark
SELECT /*+ BROADCAST(small) */ * FROM big JOIN small ON ...
SELECT /*+ MERGE(big1, big2) */ * FROM big1 JOIN big2 ON ...
SELECT /*+ COALESCE(8) */ * FROM ...
SELECT /*+ REPARTITION(100, key) */ * FROM ...

-- Snowflake (clustering hint)
ALTER TABLE t CLUSTER BY (col)
```

Use hints when:
- Stats are missing or wrong (bug or stale)
- You know something the optimizer doesn't
- You've measured and the hinted plan is faster

Don't litter every query with hints; it's brittle.

### Statistics Maintenance

Engines use statistics to plan queries. Missing or stale stats → bad plans.

| Engine | Stats command |
|---|---|
| Spark | `ANALYZE TABLE t COMPUTE STATISTICS [FOR COLUMNS ...]` |
| Snowflake | Mostly automatic |
| BigQuery | Mostly automatic |
| Redshift | `ANALYZE` (auto-runs in modern Redshift) |
| Iceberg | Computes stats on write |

**For dbt Spark / EMR jobs:** include `ANALYZE TABLE` in post-hooks for tables hit by complex joins.

### Auto-Scaling Compute

| Engine | Auto-scaling |
|---|---|
| Snowflake | Multi-cluster warehouses; auto-scale based on queue |
| BigQuery | Slot reservations with auto-scale, or on-demand |
| Redshift Serverless | Auto-scaling compute (RPUs) |
| EMR Auto Scaling | Add/remove nodes based on YARN load |
| EMR Serverless | Per-job auto-scaling |
| Glue | DPU range (`G.1X` to `G.4X`); dynamic allocation |

Auto-scaling is a cost-and-performance lever; default-on for most modern services.

---

## Worked Example — Designing the File Layout

**Scenario:** 100M events/day landing in your lake. Common queries: "give me last 7 days for user X," "yesterday's hourly counts by region," "anomalies in last hour."

**My layout:**

- **Format:** Parquet with Zstd
- **Table format:** Iceberg (because [08-table-formats.md](./08-table-formats.md))
- **Partitioning:** `days(ts), bucket(32, user_id)` — Iceberg hidden partitioning
  - `days(ts)` — most queries filter by date range
  - `bucket(32, user_id)` — point lookups by user; avoids millions of user-partitions
- **Sort within file:** `ts, user_id` — covers both common predicates with min/max stats
- **File size target:** 256 MB
- **Streaming sink:** writes 5-min micro-batches; compaction job hourly to merge into target sizes
- **Daily compaction:** rewrite previous day's partition with sorted files
- **Snapshot retention:** 7 days for time-travel; expire older
- **Metadata:** Glue Data Catalog (since AWS-first); Iceberg metadata in S3

**Predicted query patterns:**

| Query | Pruning |
|---|---|
| `WHERE ts >= '2025-04-15' AND ts < '2025-04-22' AND user_id = 42` | Days(ts) prunes 7 days; bucket(user_id) narrows to 1 of 32 buckets per day |
| `WHERE ts >= '2025-04-22 00:00' AND ts < '2025-04-22 01:00'` | One day's partitions read; within-file sort on ts narrows row groups |
| `WHERE user_id IN (1,2,3,4)` | Days are not pruned (no time filter — bad query); bucket(user_id) helps if predicates are evaluable |

**Cost reasoning:** 100M events/day × 1KB compressed = ~100 GB/day. 7 days = 700 GB. With user_id filter through 1/32 bucket = ~22 GB scanned. On Athena ≈ $0.10–$0.15 per query. Cheap.

This kind of fully-reasoned layout walkthrough is what mid-level interviewers want.

---

## Interview-Ready Cheat Sheet

**"What file size should you target?"** 128 MB–1 GB. Smaller = metadata overhead and small-files problem. Bigger = no parallelism and OOM risk.

**"How do you choose a partition column?"** High-enough cardinality to be useful, low-enough to avoid partition explosion. Should be filtered on by most queries. Should be stable. Time/date is the default.

**"What's the small-files problem?"** Streaming or fine-grained writes produce many tiny files. Each adds metadata + scheduling overhead. Fix: compaction (`OPTIMIZE` / `rewrite_data_files`) and `coalesce` before write.

**"What is hidden partitioning?"** Iceberg feature: declare partition transforms on raw columns (`days(ts)`); queries filter on the raw column and pruning happens automatically. Eliminates "I forgot to add a partition column filter" bugs.

**"What is Z-order / liquid clustering?"** Multi-column data clustering. Single-column sort gives pruning on one column; Z-order interleaves so multiple columns get partial pruning. Liquid clustering does this incrementally.

**"How do you debug a slow query?"** Check the query plan / Spark UI: pruning effective? File sizes reasonable? Joins broadcasting? Skew present? Statistics fresh? Then propose targeted fix.

**"Why does my query scan more than expected?"** Filter wraps the partition column in a function; or filter is on a non-partition column; or stats are missing; or you used `SELECT *`.

**Quick trade-off pairs:**
- Parquet vs Avro: columnar (analytics) vs row (streaming)
- Snappy vs Zstd: fast vs better-compressed
- Partition by date vs (date + hour): coarse-prune vs fine-prune
- Bucket by id vs partition by id: bounded count vs explosion
- Compact often vs rarely: query speed vs write throughput

---

## Resources & Links

- [Apache Parquet documentation](https://parquet.apache.org/docs/)
- [Iceberg partitioning + sorting](https://iceberg.apache.org/docs/latest/partitioning/)
- [Delta Lake — OPTIMIZE and Z-order](https://docs.delta.io/latest/optimizations-oss.html)
- [Databricks blog — Liquid Clustering](https://www.databricks.com/blog/announcing-general-availability-liquid-clustering)
- [Snowflake — clustering keys & micro-partitions](https://docs.snowflake.com/en/user-guide/tables-clustering-keys)
- [AWS Athena best practices](https://docs.aws.amazon.com/athena/latest/ug/best-practices.html)

*Next: [Governance, Lineage, Observability](./14-governance-lineage-observability.md) — the layer above the layout.*
