# Spark Fundamentals — RDD/DF, Lazy Eval, Partitioning, Shuffles, AQE

**Phase:** 1 (Foundations)
**Difficulty progression:** Beginner → Intermediate → Advanced
**Last updated:** April 24, 2026
**Related:** [SQL Deep Dive](./01-sql-deep-dive.md) · [Partitioning & Performance](./13-partitioning-performance.md) · [Open Table Formats](./08-table-formats.md) · [Batch vs Streaming](./06-batch-vs-streaming.md)

---

## Why Spark Matters

Spark is *the* distributed processing engine for batch DE. It's mentioned in roughly 70% of DE postings, and PySpark is the most-tested distributed-compute interview topic after SQL. Even if your shop uses dbt + warehouse SQL for most work, Spark shows up the moment you need to: process raw lake data, transform at scale before warehouse load, run Spark Streaming, or use Databricks/EMR.

You don't need to be a Spark internals wizard at your level — you need to be solidly intermediate, with a clear mental model of partitions, shuffles, and the Catalyst optimizer.

---

## BEGINNER — The Mental Model

### What Spark Actually Is

Spark is a distributed compute engine that lets you write code as if you're operating on one giant table, while the engine splits the work across many machines. You write transformations (filter, join, groupBy); Spark figures out how to execute them in parallel across a cluster.

### The Cluster Architecture

```
[Driver]                           ← runs your code, builds query plan
   │
   ▼
[Cluster manager: YARN / K8s / EMR / Databricks]
   │   ├── Executor 1 (cores, memory)
   │   ├── Executor 2 (cores, memory)
   │   ├── Executor 3 (cores, memory)
   │   └── ...
```

- **Driver** — your program. Builds a logical plan, asks the cluster manager for resources, sends tasks to executors, collects results.
- **Executors** — JVMs running on worker nodes that hold partitions of data and run tasks.
- **Tasks** — individual units of work (one task per partition per stage).
- **Cluster manager** — the resource broker (YARN, Kubernetes, EMR's YARN, Databricks).

A common cause of "my Spark job is slow" is misunderstanding this layout — e.g., calling `collect()` and OOM'ing the driver, or having one giant partition that pins one executor at 100%.

### Three APIs (Pick One: DataFrame)

| API | When |
|---|---|
| **RDD** (low-level) | Almost never. Pre-2.0 era, type-safe but no Catalyst optimization. |
| **DataFrame** (default) | The right choice. Untyped (Row), Catalyst-optimized, supports Python/Scala/SQL. |
| **Dataset** (Scala/Java) | Type-safe wrapper around DataFrame. Not available in PySpark. |
| **Spark SQL** | Same engine as DataFrame, you write SQL strings. |

For PySpark interviews, write DataFrame code (`df.filter(...).groupBy(...).agg(...)`) or Spark SQL. Mentioning RDDs unprompted signals you're stuck in 2016.

### Lazy Evaluation

Spark transformations (`filter`, `select`, `join`, `groupBy`) are *lazy* — they build a logical plan but don't execute. Only **actions** (`collect`, `count`, `show`, `write`) trigger execution.

```python
df = spark.read.parquet("s3://...")
clean = df.filter(df.amount > 0)            # lazy — nothing runs
agg = clean.groupBy("user_id").sum("amount") # still lazy
agg.write.parquet("s3://out/")              # NOW it runs
```

**Why this matters:**
- Spark optimizes the *whole pipeline*, not individual steps. Filters get pushed down, columns get pruned, joins get reordered.
- Calling `df.count()` mid-pipeline triggers a full execution — debugging by inserting `count()` calls is expensive.
- Caching matters when a DF is used in multiple actions; otherwise it's recomputed.

### Transformations vs Actions

| Transformations (lazy) | Actions (trigger execution) |
|---|---|
| `select`, `filter`, `withColumn`, `drop` | `collect`, `count`, `show`, `take` |
| `join`, `groupBy`, `agg`, `union` | `write`, `save`, `toPandas` |
| `repartition`, `coalesce`, `sortWithinPartitions` | `foreach`, `foreachPartition` |
| `cache`, `persist` | `collect_list` (sometimes triggers) |

### Partitions — The Unit of Parallelism

A DataFrame is split into **partitions**; each partition is processed by one task on one executor. Number of partitions = parallelism.

- Reading Parquet: ~one partition per file (often coalesced if files are small)
- After a shuffle: defaults to `spark.sql.shuffle.partitions` (200 by default — usually wrong)
- After `repartition(n)`: exactly `n` partitions
- After `coalesce(n)`: ≤ `n` partitions (no shuffle, can be skewed)

**Rule of thumb:** target 100MB–1GB per partition for batch work. Way more = OOM and slow tasks. Way fewer = no parallelism.

---

## INTERMEDIATE — Joins, Shuffles, and Partitioning

### Wide vs Narrow Transformations (the most-tested concept)

| | Narrow | Wide |
|---|---|---|
| Definition | Each output partition depends on one input partition | Each output partition depends on many input partitions |
| Examples | `filter`, `select`, `withColumn`, `union` | `groupBy`, `join`, `distinct`, `repartition` |
| Cost | Cheap (no network) | Expensive (shuffle across network) |
| Stage boundary | Stays in same stage | Triggers a new stage |

**A "stage" is a sequence of narrow transformations between two shuffles.** When you read the Spark UI, stages map to shuffles.

### Shuffles — Where Performance Goes to Die

A shuffle redistributes data across partitions by some key (the join key, the groupBy key, the repartition key). It writes intermediate data to disk on each executor and reads it across the network.

**Why shuffles hurt:**
- Disk I/O on every executor (writes shuffle files)
- Network I/O (executors fetch shuffle blocks from each other)
- Memory pressure (sorting / aggregating before write)
- Skew amplification (one partition with 90% of data → one slow task)

**Reducing shuffles:**
- Filter early — push filters before joins/aggregates so less data shuffles
- Pre-partition data by the join key (write Parquet bucketed by key, or upstream Spark partitions by key)
- Broadcast small tables instead of shuffling
- Combine multiple aggregates into one pass instead of shuffling repeatedly

### Join Strategies

Spark picks among several join strategies; you should know each.

| Strategy | When Spark picks it | Cost |
|---|---|---|
| **Broadcast Hash Join** | One side ≤ `spark.sql.autoBroadcastJoinThreshold` (10MB default) | Cheap — small side replicated to every executor |
| **Shuffle Hash Join** | Build side fits in memory, both sides not sorted | Medium — shuffle by key |
| **Sort Merge Join** | Both sides large; default for big × big | Expensive — shuffle + sort both sides |
| **Broadcast Nested Loop / Cartesian** | Non-equi joins, no key | Avoid at all costs (O(n × m)) |

**Force a broadcast when you know it's safe:**
```python
from pyspark.sql.functions import broadcast
df_big.join(broadcast(df_small), "key")
```

This is the single highest-ROI Spark optimization most pipelines miss. Confirm it landed by inspecting the physical plan: `df.explain()` should show `BroadcastHashJoin`.

**AQE will sometimes upgrade Sort Merge to Broadcast at runtime** (see below), but explicit hints are still useful when stats are missing.

### Skew — The Silent Killer

If your join key has hot values (e.g., 80% of events from one user_id), all those rows land in one partition during shuffle. One task processes most of the data, others sit idle. Total runtime is dominated by that one task.

**Symptoms in Spark UI:**
- One stage's task duration: median 30s, max 45 minutes
- Spilling to disk on the long task
- Executor with one partition holding gigabytes

**Fixes:**
- **Salting** — append a random suffix to the hot key to distribute, then aggregate twice
- **Broadcast** the small side (skips shuffle entirely)
- **AQE skew join** (Spark 3+) — automatic skew detection and split (set `spark.sql.adaptive.skewJoin.enabled=true`)
- **Pre-aggregate** before joining when possible

### Repartition vs Coalesce

| | Repartition | Coalesce |
|---|---|---|
| Triggers shuffle | ✅ | ❌ |
| Can increase partitions | ✅ | ❌ (only reduces) |
| Result balance | Even | Can be skewed |
| Use case | Re-balance for downstream parallelism | Reduce partitions before write (to fewer files) |

**Common pattern — write fewer files at the end:**
```python
df.coalesce(10).write.parquet("s3://...")  # but watch for skew
```
Better:
```python
df.repartition(10).write.parquet("s3://...")  # shuffle for balance
```
Or, if writing to a partitioned table, repartition by the partition column to write one file per partition:
```python
df.repartition("date").write.partitionBy("date").parquet("s3://...")
```

### Caching and Persistence

```python
df.cache()         # default: MEMORY_AND_DISK
df.persist(StorageLevel.MEMORY_ONLY)
df.unpersist()
```

Cache only when:
- A DF is used in multiple actions
- The cost of recomputing is high

Anti-patterns:
- Caching a DF that's only used once (waste)
- Caching a huge DF that doesn't fit in memory (constant evictions, worse than recomputing)
- Forgetting to `unpersist` (memory leak across jobs)

### Partition Pruning, Predicate Pushdown, Column Pruning

Catalyst (the optimizer) does these automatically — but only if you let it.

| Optimization | What it does | What breaks it |
|---|---|---|
| **Partition pruning** | Only read partitions matching the filter | Filter on a column transformation: `WHERE year(ts)=2025` instead of `WHERE ts >= '2025-01-01'` |
| **Predicate pushdown** | Push `WHERE` to the Parquet reader (skip row groups by min/max stats) | UDFs in the predicate, complex expressions |
| **Column pruning** | Read only needed columns from Parquet | `SELECT *`, schema-changing UDFs |

**Mid-level interview question:** "Your job is reading 1TB but you only need a small slice. Why?" Answer: check whether your predicates and column references are pushdown-friendly.

---

## ADVANCED — Catalyst, AQE, and Production Patterns

### Catalyst Optimizer

Spark's query optimizer. Goes through phases:

```
Logical plan (your code → tree)
   ↓
Analyzed logical plan (resolve names, types)
   ↓
Optimized logical plan (rule-based: predicate pushdown, constant folding, ...)
   ↓
Physical plans (multiple candidates)
   ↓
Cost model picks one
   ↓
Tungsten code generation (whole-stage code gen → fast JVM bytecode)
```

**Tungsten** is the off-heap memory + code generation layer that makes DataFrames fast — it generates bytecode for whole stages of operations, avoiding interpretation overhead.

### Adaptive Query Execution (AQE) — Default On in Spark 3+

Catalyst plans queries upfront based on table stats. **AQE re-optimizes mid-query** based on what actually happens at runtime.

| Feature | What it does |
|---|---|
| **Coalesce shuffle partitions** | Merge tiny shuffle partitions into bigger ones (no need to manually tune `shuffle.partitions`) |
| **Switch join strategy** | Upgrade SortMerge → Broadcast when actual size is small |
| **Skew join handling** | Detect skewed partitions and split them automatically |

Enable (defaults true in Spark 3.2+):
```
spark.sql.adaptive.enabled = true
spark.sql.adaptive.skewJoin.enabled = true
spark.sql.adaptive.coalescePartitions.enabled = true
```

**Mid-level interview answer to "how would you tune Spark today?":** "Mostly let AQE do it. Set sensible defaults for shuffle partitions, broadcast threshold, and AQE skew handling. Profile only when AQE doesn't solve it."

### Reading the Spark UI (Required Skill)

The Spark UI is your debugger. The four tabs to know:

| Tab | What you find |
|---|---|
| **Jobs** | List of jobs (one per action), their stages, duration |
| **Stages** | Per-stage breakdown: tasks, shuffle read/write, durations, GC time |
| **SQL / DataFrame** | Per-query physical plan with metrics annotated |
| **Executors** | Per-executor memory, GC, shuffle, dead/alive |

**Key things to look at:**
- Stage with the highest duration → bottleneck
- Task duration distribution within a stage → skew
- Shuffle read/write GB → cost
- Spill to disk → memory pressure
- Whole-stage code gen IDs in SQL plan → confirm Catalyst optimized

### File Sizes and the Small-Files Problem

Spark works best when each task processes 100MB–1GB. Real-world lakes accumulate millions of small files (10 KB each), each requiring metadata listing and one-task-per-file processing.

**Symptoms:** A 10GB job takes hours because it's reading 1M files.

**Fixes:**
- Use lakehouse formats (Iceberg/Delta) with file compaction (`OPTIMIZE`)
- Repartition before writes
- Use Spark's `coalesce` to write fewer, larger files
- Periodic compaction jobs

See [13-partitioning-performance.md](./13-partitioning-performance.md).

### Spark Configurations Worth Knowing

| Config | What |
|---|---|
| `spark.sql.shuffle.partitions` | Default 200; set higher for big jobs (1000–4000) or let AQE coalesce |
| `spark.sql.autoBroadcastJoinThreshold` | Default 10MB; bump to 50–200MB if mem allows |
| `spark.executor.memory` / `spark.executor.cores` | Per-executor sizing |
| `spark.executor.memoryOverhead` | Off-heap memory; bump on Python-heavy jobs |
| `spark.sql.adaptive.enabled` | Always true |
| `spark.sql.files.maxPartitionBytes` | Default 128MB — the read-side partition target size |
| `spark.dynamicAllocation.enabled` | Auto-scale executors based on load |

You don't need to memorize all of these — but knowing the half-dozen that matter and *why* signals real Spark experience.

### PySpark Specifics

PySpark runs Python on the driver, JVM on the executors, with serialization between them.

**The expensive thing in PySpark:** Python UDFs. Each row crosses the JVM↔Python boundary, serializing/deserializing.

| API | Speed |
|---|---|
| Pure DataFrame ops (no UDFs) | Fast — runs in JVM |
| **Pandas UDFs / vectorized UDFs** (Arrow) | Fast-ish — batches across boundary |
| Plain Python UDFs | Slow — row-by-row |

**Always prefer DataFrame functions (`F.col`, `F.when`, `F.regexp_extract`) over UDFs.** When you must write logic that doesn't exist as a built-in, use a Pandas UDF.

### Memory Tuning Mental Model

Spark executor memory has zones:

```
Total executor memory
├── Reserved (300MB)
├── Spark memory (60%)
│   ├── Storage (caching)         ← spark.memory.storageFraction
│   └── Execution (joins, sorts)  ← rest of Spark memory
└── User memory (40%)             ← Python objects, UDF state
```

**OOM patterns:**
- "Out of memory in shuffle" → execution memory; reduce shuffle partitions, increase memory, fix skew
- "Out of memory while caching" → storage memory; reduce cache, increase memory
- "Out of memory in driver" → you called `collect()` on a big DF; use `take(n)` or write to storage

### Spark Structured Streaming (Briefly)

Same DataFrame API, but `readStream` instead of `read`. Conceptually micro-batch (default ≤ 100ms), with continuous mode available but rarely used.

```python
events = spark.readStream.format("kafka").option("subscribe", "events").load()
agg = events.withWatermark("ts", "10 minutes") \
            .groupBy(window("ts", "5 minutes"), "user_id") \
            .count()
agg.writeStream.format("delta").outputMode("update").start()
```

Trade-offs vs Flink: easier if your team already knows Spark, weaker for true sub-second latency, simpler stateful semantics. See [06-batch-vs-streaming.md](./06-batch-vs-streaming.md).

### Spark on AWS — EMR vs Glue vs Databricks vs EMR Serverless

| Platform | Strengths |
|---|---|
| **EMR (cluster)** | Full Spark with all the knobs; cheaper at sustained workloads; ops burden of running clusters |
| **EMR Serverless** | No clusters; pay-per-job; great for spiky workloads; less knob control |
| **AWS Glue** | Fully managed; auto-provisioning; great for ETL; some Spark features absent or behind |
| **Databricks** | Best Spark experience; Delta Lake native; expensive but powerful |

See [12-aws-data-stack.md](./12-aws-data-stack.md).

---

## Worked Example — Optimizing a Slow Spark Job

You inherit a job that takes 4 hours. You're asked to debug live. Walkthrough:

1. **Open Spark UI → Jobs.** Identify the longest job.
2. **Drill to Stages.** Identify the dominant stage by duration.
3. **Look at task duration distribution.** If max ≫ median → skew. If all similar but slow → not skew, just cost.
4. **Check shuffle metrics.** If shuffle write is huge (50GB+ for a small final output), look upstream — too much data shuffling that could be filtered first.
5. **Check the SQL/DataFrame physical plan.** Look for: `SortMergeJoin` where one side is small → should be broadcast. `Filter` after `Join` → should be before. `Project *` → could prune columns.
6. **Look at executor metrics.** GC time > 10% of task time → memory pressure. Spilling to disk → memory not enough.
7. **Hypothesize and test.**
   - Skew: enable AQE skew join; or salt key
   - Missing broadcast: hint or raise threshold
   - Bad partitioning: repartition by key before join
   - Too many small files: coalesce on write or compact upstream
   - Wrong partition column for downstream filters: rewrite with partition strategy

For a mid-level interview, walking through the UI like this is the answer. "I'd Google it" is not.

---

## Interview-Ready Cheat Sheet

**"Wide vs narrow transformations?"** Narrow: each output partition depends on one input partition (filter, select). Wide: each depends on many (groupBy, join). Wide triggers shuffles and stage boundaries.

**"What are the join strategies?"** Broadcast (small side replicated), Shuffle Hash (medium), Sort Merge (default for big × big), Cartesian (avoid). Force broadcast for known small sides.

**"What's a shuffle and why is it expensive?"** Repartitioning data across the cluster by a key. Writes shuffle files to disk, fetches across network. The biggest single cost in most Spark jobs.

**"What is AQE?"** Adaptive Query Execution — re-optimizes the plan at runtime using actual stats. Coalesces shuffle partitions, upgrades SortMerge → Broadcast, splits skewed partitions. Default on in Spark 3+.

**"How would you handle skew?"** AQE skew join first. If insufficient: salt the hot key (concat with a random small int), aggregate twice. Or broadcast the small side to skip shuffle.

**"Why is my Spark job slow?"** Open the UI: identify the dominant stage, look at task duration distribution, check shuffle/spill metrics, examine the physical plan, hypothesize. Common culprits: missing broadcast, skew, small-files problem, unfortunate partitioning.

**"DataFrame vs RDD?"** DataFrames go through Catalyst optimization and Tungsten code generation; RDDs don't. Use DataFrames or Spark SQL except in very narrow edge cases.

**"PySpark UDF performance?"** Plain Python UDFs are slow (row-by-row serialization). Use DataFrame built-ins or Pandas UDFs (Arrow-batched).

**Quick trade-off pairs:**
- Repartition vs coalesce: balanced shuffle vs no-shuffle merge
- Broadcast vs shuffle join: cheap-if-small vs scales to big × big
- Cache vs recompute: memory cost vs compute cost
- AQE vs manual tuning: dynamic vs predictable

---

## Resources & Links

- [Spark — The Definitive Guide (Bill Chambers, Matei Zaharia)](https://www.oreilly.com/library/view/spark-the-definitive/9781491912201/)
- [Apache Spark official docs — SQL and DataFrame guide](https://spark.apache.org/docs/latest/sql-programming-guide.html)
- [Databricks "Mastering the Spark UI"](https://www.databricks.com/learn/training/lp/databricks-academy)
- [Spark Performance Tuning — Cloudera](https://blog.cloudera.com/how-to-tune-your-apache-spark-jobs-part-1/) — older but the fundamentals haven't changed
- [The Internals of Apache Spark (Jacek Laskowski)](https://books.japila.pl/apache-spark-internals/) — deep, free

*Next: [Open Table Formats](./08-table-formats.md) — Iceberg/Delta/Hudi sit on top of Parquet and turn lakes into lakehouses.*
