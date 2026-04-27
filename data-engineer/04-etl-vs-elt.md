# ETL vs ELT — Pipeline Patterns, Idempotency, and Backfills

**Phase:** 1 (Foundations)
**Difficulty progression:** Beginner → Intermediate → Advanced
**Last updated:** April 24, 2026
**Related:** [SQL Deep Dive](./01-sql-deep-dive.md) · [Orchestration & Airflow](./05-orchestration-airflow.md) · [dbt — Modular SQL Transforms](./10-dbt.md) · [CDC](./09-cdc.md)

---

## Why This Topic Matters

"Tell me about a pipeline you built" is the most common open-ended interview question for DEs. To answer it well, you need a vocabulary for *pipeline patterns* — and that means knowing the trade-offs between ETL and ELT, plus the cross-cutting concerns of [idempotency](#idempotency), incremental processing, and backfills.

Most production pipeline failures come from violating one of these principles. The same is true of most failed interview answers.

---

## BEGINNER — The Core Patterns

### ETL vs ELT

| | ETL | ELT |
|---|---|---|
| Order | Extract → Transform → Load | Extract → Load → Transform |
| Where transforms happen | On a separate compute layer (Spark, Python, SSIS) | Inside the warehouse (SQL) |
| Era | Historical (Hadoop, on-prem warehouses) | Modern (cloud warehouses with cheap compute) |
| Pros | Lighter warehouse, can pre-shape sensitive data before landing | Use SQL, leverage warehouse scale, easier to iterate |
| Cons | Two systems to maintain, transforms tied to engine choice | Loads raw (sometimes sensitive) data into warehouse first |

**The modern default is ELT.** Cloud warehouses are cheap and fast enough that loading raw data and transforming with SQL/dbt is simpler and faster than building a separate transformation layer.

You'll still see ETL in:
- Streaming pipelines (transforms in Flink/Spark before landing)
- Sensitive data (PII tokenization before warehouse load)
- Legacy stacks
- Cross-system transforms where the warehouse isn't the destination

### A Real Pipeline Has More Than Three Steps

```
[Sources] → [Ingest] → [Raw zone] → [Transform] → [Marts] → [Consumers]
                          │             │
                          └── tests, contracts, observability throughout
```

Each arrow is a place pipelines fail. A good interviewer probes each of them.

### Batch vs Micro-Batch vs Streaming

| | Batch | Micro-batch | Streaming |
|---|---|---|---|
| Cadence | Hourly / daily | Every few minutes | Continuous |
| Latency | Hours | Minutes | Seconds or sub-second |
| Complexity | Low | Medium | High |
| Ordering | Easy | Easy | Hard (event time vs processing time) |
| Examples | Daily nightly aggregates | dbt every 5 min | Fraud detection, real-time CDC |

See [06-batch-vs-streaming.md](./06-batch-vs-streaming.md) for the full streaming story.

**Honest mentor advice:** at your level, you're expected to be solid on batch + micro-batch and conversant on streaming. Most DE jobs are 80% batch.

---

## INTERMEDIATE — The Patterns That Make Pipelines Reliable {#idempotency}

### Idempotency — The Most Important Concept in DE

**Definition:** A pipeline is idempotent if running it N times has the same effect as running it once.

Why it matters: pipelines fail. Re-running them shouldn't double-count, miss data, or corrupt state. Every interviewer probes this — usually with "what happens if your job runs twice?"

**Idempotent patterns:**

| Pattern | How |
|---|---|
| Truncate-and-load | `DELETE` (or partition replace) then `INSERT`. Each run produces the same final state. |
| Upsert / `MERGE` | `MERGE INTO target USING source ON pk WHEN MATCHED ... WHEN NOT MATCHED ...`. Re-runs converge. |
| Insert-only with dedup | Append, then dedup with `ROW_NUMBER` window function on later read or transform. |
| Partition replace | Compute one partition's worth of data, atomically swap it. The norm in lakehouse formats. |
| Idempotent insert by hash | Generate a deterministic event_id from payload; primary key conflict → silent skip. |

**Anti-patterns to avoid (and to call out in interviews if you spot them):**

- `INSERT INTO target SELECT ... WHERE created_at > now() - interval '1 day'` — different result every run; not idempotent.
- Auto-increment IDs that "advance" each run — re-runs produce different surrogate keys.
- Side effects (sending emails, calling APIs) inside a transformation step — re-runs duplicate side effects.

### Incremental Loads

Reprocessing all history every run is slow and expensive. Incremental processing only handles new/changed data — but it's where bugs hide.

**Watermark / cursor-based:**
```sql
INSERT INTO daily_agg
SELECT date_trunc('day', ts) AS day, COUNT(*) AS events
FROM events
WHERE ts >= (SELECT COALESCE(MAX(day), '1970-01-01') FROM daily_agg)
  AND ts <  date_trunc('day', current_timestamp);  -- exclude in-flight day
```

Advantages: simple, follows event-time semantics.
Failure mode: late-arriving data (event with `ts = yesterday` arriving today). Common fixes: rolling window (re-process last N days), change-data-capture from source.

**Hash / change detection:**
```sql
MERGE INTO target t
USING (
  SELECT *, MD5(CONCAT_WS('|', col1, col2, col3)) AS row_hash
  FROM source
) s
ON t.id = s.id
WHEN MATCHED AND t.row_hash <> s.row_hash THEN UPDATE SET ...
WHEN NOT MATCHED THEN INSERT ...;
```

Useful when source has no reliable timestamp/version column.

**Process-time vs event-time:**
- *Event time* — when the thing happened in the real world
- *Process time* — when your pipeline saw the data
- For analytics that match the business, use event time
- For SLA tracking, use process time

### Backfills — The Other Half of Idempotency

A backfill is a re-run of historical data — when you discover a bug, change a transformation, or onboard new data into an old period.

**Requirements for safe backfills:**
1. **Idempotent transforms** (above)
2. **Parameterized by date** — pipeline takes a date range, not "now"
3. **Partitioned writes** — backfill replaces specific partitions atomically
4. **Bounded inputs** — read only data needed for the date range
5. **Observable** — easy to compare backfilled output to existing

**Backfill strategies:**

| Strategy | When |
|---|---|
| Replace single partition | Bug found in one day's data |
| Re-run last N days rolling | Recurring late-data issue |
| Full re-run from raw | Big logic change, schema migration |
| Dual-run / shadow | Big change to a critical pipeline; compare new vs old before cutover |

In Airflow: `clear` a task instance for a date range and re-run; assumes your DAG is parameterized properly. dbt: `--full-refresh` for incremental models, or run with specific date variables.

### Dead Letter Queues / Quarantine

When a record fails (bad schema, parse error, FK violation), don't fail the whole batch — quarantine it.

```
events_raw → transform → events_clean
                    │
                    └─→ events_dlq  (with error reason, source, timestamp)
```

This pattern means:
- Pipelines stay running (one bad row doesn't take down the whole job)
- You can investigate / fix / replay quarantined data later
- You have observable error rates (dashboard the DLQ growth)

### Exactly-Once vs At-Least-Once vs At-Most-Once

| Semantic | Behavior | Trade-off |
|---|---|---|
| At-most-once | Each record processed 0 or 1 times | Cheap, can lose data |
| At-least-once | Each record processed 1+ times | Reliable, may double-count |
| Exactly-once | Each record processed exactly once | Hard; needs idempotent sink or transactional writes |

**Most production pipelines target "at-least-once with idempotent sinks," which is functionally exactly-once.** Streaming systems offer real exactly-once via transactional commits (Kafka transactions, Flink two-phase commit) at the cost of latency. See [06-batch-vs-streaming.md](./06-batch-vs-streaming.md).

### Schema Evolution and Late-Arriving Schemas

Source schemas change. Your pipeline shouldn't break the moment a column is added.

| Change | Risk | Mitigation |
|---|---|---|
| Column added | Low (ignore safely) | Default `NULL`; dbt picks it up if you re-source |
| Column removed | High (downstream breaks) | Schema contract + alert; never silently drop |
| Type changed | High | Schema contract + freeze types in stage models |
| Column renamed | Catastrophic | Treat as remove + add; require source notification |

The pattern that helps: **explicit schemas at the boundary**. Cast columns to expected types in the staging model, alert on cast failures. See [11-data-quality-contracts.md](./11-data-quality-contracts.md).

---

## ADVANCED — Production Pipeline Patterns

### The Anatomy of a Production Pipeline

```
1. Trigger
   ├── Time-based (cron) — daily 2am
   ├── Event-based (S3 file landed, message on queue)
   ├── Sensor (poll for upstream readiness)
   └── Manual (backfill, ad hoc)

2. Extract
   ├── API → handle pagination, rate limits, auth refresh
   ├── DB → use replication, snapshot, or CDC (see [09-cdc.md])
   ├── Files → S3 listing, manifest files, idempotent moves
   └── Stream → checkpoint, replay support

3. Validation (early)
   ├── Schema check
   ├── Row count sanity (compared to historical baseline)
   ├── Required columns present
   └── Quarantine bad rows

4. Transform
   ├── Read deterministic input slice (date range, watermark)
   ├── Apply business logic
   ├── Generate stable surrogate keys
   └── Write to staging area

5. Load (atomic)
   ├── Partition replace (lakehouse)
   ├── MERGE upsert (warehouse)
   └── Symlink swap (legacy)

6. Validate (late)
   ├── Row count matches expectations
   ├── No NULLs in required cols
   ├── Aggregates within tolerance vs upstream
   └── Critical metrics within bounds

7. Notify
   ├── Success — log
   ├── Failure — page (real failure, not flake)
   └── Partial / quarantine alerts
```

A junior pipeline does steps 2, 4, 5. A mid-level pipeline does all seven. **You will be evaluated on whether your answers cover the full lifecycle.**

### Idempotent Pipeline Templates

**Template 1 — daily partitioned table:**
```python
# pseudocode
def run(execution_date: date):
    target_partition = execution_date  # YYYY-MM-DD

    # 1. Read deterministic slice
    source = read_source(
        where=f"event_date = '{execution_date}'"
    )

    # 2. Transform
    transformed = transform(source)

    # 3. Atomic partition replace
    write_partition(
        table="silver.events",
        partition={"event_date": target_partition},
        df=transformed,
        mode="overwrite"
    )

    # 4. Validate
    assert_row_count(table="silver.events", partition=target_partition,
                    min_rows=expected_min(execution_date))
```

This is rerunnable for any historical date. Backfilling 2 weeks = call `run` 14 times. Parallel-safe by partition.

**Template 2 — incremental with merge:**
```sql
-- dbt incremental model
{{ config(materialized='incremental', unique_key='event_id') }}

select
  event_id,
  user_id,
  event_ts,
  ...
from {{ source('raw', 'events') }}
{% if is_incremental() %}
where event_ts >= (select max(event_ts) from {{ this }}) - interval '3 days'
{% endif %}
```

The 3-day lookback handles late-arriving events. Re-runs converge.

### Slowly Changing State and Reprocessing

If a downstream model uses an SCD Type 2 dim, reprocessing is more involved than just re-running facts. You may need to:
1. Replay dim changes in the original order
2. Recompute the fact joins for the affected window

This is one of the most under-discussed pipeline failure modes. The interview-ready answer: **separate "current state snapshots" from "historical facts," reprocess each independently, and document which models depend on which dim history.**

### Optimizing for Cost: Don't Recompute What Hasn't Changed

| Pattern | Win |
|---|---|
| Incremental dbt models | Skip full table rebuilds |
| Partition-level state | Track watermarks per source/table; only process new partitions |
| Hash-based change detection | Skip transforms when input hash matches last successful run |
| Materialized aggregates | Compute once, query many |
| Caching layer (Snowflake result cache, BigQuery cache) | Free in some warehouses; explicit elsewhere |

### Pipeline Orchestration Across Failure Modes

| Failure | Right answer |
|---|---|
| Source unreachable | Sensor + retry with backoff; do not silently skip |
| Schema drift | Quarantine + alert; do not silently coerce |
| Bad row | Quarantine to DLQ; continue |
| Downstream slow | Decouple via queue; back-pressure rather than crashing |
| Job timeout | Investigate; add resources or split |
| Transient flake (network, S3) | Retry; alert only after N retries |
| Data anomaly (3x normal volume) | Alert, don't auto-block; let humans decide |

The skill is knowing which failure deserves a retry, which deserves a page, and which deserves a quarantine. See [Orchestration & Airflow](./05-orchestration-airflow.md) for the operational machinery.

### Observability Through Pipelines

Every pipeline should emit (at minimum):
- Start / success / failure events
- Rows in, rows out (for each step)
- Duration
- Cost (if your engine reports it)
- Schema fingerprint (for drift detection)

Tools: OpenLineage standard, Marquez, DataHub, Monte Carlo, Great Expectations, dbt's `--profile`. See [14-governance-lineage-observability.md](./14-governance-lineage-observability.md).

### Costs of "Just Reprocess from Scratch"

Junior reflex: "When in doubt, re-run from raw." That works on a small system. At scale, "reprocess everything" can be:
- 10+ hours of compute
- Tens of thousands of dollars
- Blocking other pipelines
- Re-emitting events to downstream systems (analytics, ML serving) that *don't* expect them

Mid-level engineers know: **be precise about what to backfill, why, and over what window.**

---

## Worked Pipeline Walk — Stripe Invoices into Warehouse

You're asked to stand up an hourly pipeline from Stripe to your warehouse. Walkthrough an interviewer would expect:

1. **Trigger:** Hourly cron in Airflow / Step Functions.
2. **Extract:** Stripe API has cursor-based pagination and a `created` filter. Pull events `created >= last_watermark`. Store the new max watermark only after successful load.
3. **Land:** Write raw JSON to S3 partitioned by `event_date`. Bronze.
4. **Validate:** schema check; row count nonzero (zero rows on what should be a busy hour → alert).
5. **Stage/Transform:** Cast columns, deduplicate by `event.id` (Stripe sends events more than once during retries), unpack `data.object`. Silver.
6. **Mart:** Build `dim_customer`, `fact_invoice`, `fact_charge`. Gold.
7. **Idempotency:** Each step is partition-replaceable by `event_date`. Re-running for a date overwrites cleanly.
8. **Backfill:** I parameterize by `start_date / end_date`, run 24×N times for N days; partitions process in parallel.
9. **Failure modes:** API rate limit (exponential backoff), Stripe outage (retry with circuit breaker; surface as alert), schema change (alert; don't auto-coerce — Stripe versions APIs anyway).
10. **Observability:** OpenLineage events emitted; rows in / out / duration / cost dashboards; Slack alert on failures or DLQ growth.

That walkthrough hits idempotency, backfills, validation, observability, and ops — exactly what mid-level interviews probe.

---

## Interview-Ready Cheat Sheet

**"What's the difference between ETL and ELT?"** ETL transforms before loading (separate engine); ELT loads raw, transforms inside the warehouse with SQL. Modern default is ELT because warehouses are fast and cheap.

**"What does idempotent mean and why does it matter?"** Running N times = running once. Pipelines fail. Re-runs shouldn't corrupt state. Achieved via partition-replace, MERGE, or hash-based dedup.

**"How do you handle late-arriving data?"** Rolling lookback window in incremental loads (e.g., re-process last 3 days every run), or treat late events as updates via MERGE keyed on event_id.

**"What's exactly-once?"** Each record processed exactly once. In practice: at-least-once with idempotent sinks. True exactly-once requires transactional commits (Kafka transactions, Flink 2PC) and adds latency.

**"How do you back-fill a 30-day bug?"** Parameterize the pipeline by date; clear / replace each affected partition; run in parallel where safe; validate against expected row counts; watch for downstream dependents (SCD2 dims, aggregates).

**"A row-level error breaks your job. What do you do?"** Quarantine the row to a DLQ with the error reason, alert if DLQ rate exceeds threshold, continue processing the rest. Investigate offline.

**Quick trade-off pairs:**
- ETL vs ELT: separate engine vs warehouse-native
- Watermark vs hash-based incremental: time-based vs change-based
- Truncate-load vs MERGE: simpler vs preserves identity
- Reprocess all vs partition replace: slow/cheap-thinking vs fast/precise

---

## Resources & Links

- [Designing Data-Intensive Applications — Kleppmann (Ch. 11 streams, Ch. 12 batch)](https://dataintensive.net/)
- [The Data Engineer's Guide to Apache Spark](https://www.databricks.com/resources/ebook/the-data-engineers-guide-to-apache-spark) — covers idempotent partitioning patterns
- [dbt — Incremental models docs](https://docs.getdbt.com/docs/build/incremental-models)
- [Airflow Best Practices for idempotency](https://airflow.apache.org/docs/apache-airflow/stable/best-practices.html)
- [Functional Data Engineering — Maxime Beauchemin](https://maximebeauchemin.medium.com/functional-data-engineering-a-modern-paradigm-for-batch-data-processing-2327ec32c42a) — the seminal essay on idempotent batch DE

*Next: [Orchestration & Airflow](./05-orchestration-airflow.md) for the operational machinery.*
