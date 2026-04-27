# Change Data Capture (CDC) — Keeping Analytics Fresh from Source

**Phase:** 1 (Foundations)
**Difficulty progression:** Beginner → Intermediate → Advanced
**Last updated:** April 24, 2026
**Related:** [Batch vs Streaming](./06-batch-vs-streaming.md) · [Open Table Formats](./08-table-formats.md) · [ETL vs ELT & Pipeline Patterns](./04-etl-vs-elt.md) · [AWS Data Stack](./12-aws-data-stack.md)

---

## Why CDC Matters

The classic batch pattern — "every night, full snapshot of the source DB" — breaks at scale. Snapshots take hours, hammer the source DB, and produce hours-stale analytics. CDC replaces snapshots with a continuous stream of inserts, updates, and deletes, giving you near-real-time analytics with minimal source impact.

CDC has gone from "nice to have" to "expected" in mid-level DE interviews. Even if you've never built a CDC pipeline, you should understand the patterns and trade-offs.

---

## BEGINNER — The Big Picture

### What CDC Is

A **CDC pipeline** captures every change (insert / update / delete) to a source database and emits those changes as a stream of events.

```
Source DB (Postgres / MySQL / Mongo / DynamoDB)
    │
    │  insert / update / delete events
    ▼
CDC tool (Debezium / DMS / Kafka Connect / etc.)
    │
    ▼
Stream (Kafka / Kinesis)
    │
    ▼
Sink (Lakehouse table / Warehouse / Search index / Cache)
```

### Three Approaches to "Get Changes Out of a Database"

| Approach | How | Pros | Cons |
|---|---|---|---|
| **Periodic snapshot** | Dump full table on schedule | Dead simple | Slow, expensive, hammers source, hours-stale |
| **Cursor / timestamp polling** | `WHERE updated_at > last_run` | Easy to implement | Misses deletes, requires `updated_at` discipline, not truly real-time |
| **Log-based CDC** | Tail the DB transaction log (WAL / binlog / oplog) | Real-time, captures deletes, low source impact | More setup, requires DB privileges, more moving parts |

**Log-based CDC is the modern default.** Snapshots and polling still happen for simple cases or initial loads.

### Why Log-Based CDC Wins

Every transactional database keeps a write-ahead log (Postgres WAL, MySQL binlog, Mongo oplog) for durability and replication. CDC tools parse this log and emit each change as an event. Benefits:

- Captures inserts, updates, *and* deletes — polling misses deletes
- Near-real-time (sub-second to seconds latency)
- Doesn't query the production tables — reads the log instead, minimal source impact
- Inherently ordered per row (the log is sequential)
- Can include "before" and "after" images of the row for full change visibility

---

## INTERMEDIATE — Tools and Patterns

### The Tool Landscape

| Tool | Source DBs | Notes |
|---|---|---|
| **Debezium** | Postgres, MySQL, Mongo, SQL Server, Oracle, etc. | Most popular open-source CDC; runs as Kafka Connect connectors |
| **AWS DMS** | Many, including Postgres, MySQL, Oracle, SQL Server, Mongo | Managed; targets S3, Kinesis, Kafka, Redshift |
| **Fivetran / Airbyte / Stitch** | SaaS sources + DBs | Managed ELT; often "snapshot + log" hybrid |
| **Maxwell / Canal** | MySQL | Lightweight binlog readers; less popular than Debezium |
| **Estuary Flow / Materialize** | Multiple | Newer streaming-first tools |
| **DynamoDB Streams** | DynamoDB only | Native — every table change available as a stream |
| **MongoDB change streams** | Mongo | Native; convenient |

### A Concrete Debezium Pipeline

```
Postgres
   │ (logical replication slot)
   ▼
Debezium Postgres Connector (running on Kafka Connect)
   │ (one Kafka topic per table, by default)
   ▼
Kafka topics: db.public.users, db.public.orders, ...
   │
   ▼
Stream processor (Flink / Spark Streaming / Kafka Connect sink)
   │
   ▼
Iceberg / Delta tables (silver layer)
```

**Example Debezium event payload (simplified):**
```json
{
  "before": { "id": 42, "name": "Alice", "email": "alice@old.com" },
  "after":  { "id": 42, "name": "Alice", "email": "alice@new.com" },
  "op": "u",                       // c=create, u=update, d=delete, r=read (snapshot)
  "ts_ms": 1714060800123,
  "source": { "name": "..." }
}
```

### Initial Load + Streaming (the "snapshot + log" pattern)

You can't start streaming from now and expect a complete table. Standard pattern:

1. **Snapshot phase** — read the source table in batches, emit each row as a "read" event (op=`r`).
2. **Streaming phase** — switch to tailing the log; emit subsequent changes as `c`/`u`/`d`.
3. The downstream consumer applies all events idempotently — duplicate snapshot rows from a restart are OK.

Debezium handles this transition automatically. AWS DMS calls it "full load + CDC."

### The MERGE Sink Pattern

Once changes land in a stream, you usually want them as an **up-to-date copy** of the source table in your lakehouse:

```sql
MERGE INTO silver.users AS t
USING staging.user_changes AS s
ON t.user_id = s.user_id
WHEN MATCHED AND s.op = 'd' THEN DELETE
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED AND s.op <> 'd' THEN INSERT *;
```

Run this MERGE on a micro-batch cadence (every minute / 5 minutes) against accumulated changes. Iceberg, Delta, and Hudi all support this. Hudi's MoR is particularly optimized for it.

### CDC Patterns (Two Big Choices)

| Pattern | Behavior | Use case |
|---|---|---|
| **Latest-state mirror** | Apply changes via MERGE; downstream sees only current state | Mirror an OLTP table for analytics |
| **Append-only history** | Append every change to a log table (with op type, timestamp, before/after) | Audit, slowly-changing dim Type 2, time travel use cases |

**Common combo:** keep both. Append-only `bronze.user_changes` for full history; MERGE-applied `silver.users` for current state. Each serves different consumers.

### Idempotency in CDC Pipelines

Every CDC delivery is at-least-once by default. You'll see duplicate events in failure scenarios. The MERGE pattern is naturally idempotent because:
- Duplicate `c` for an existing row → no-op (or harmless re-set of same fields)
- Duplicate `u` → re-applies the same update
- Duplicate `d` → idempotent delete

The pitfall: if events arrive **out of order**, you can apply an old update over a newer one. Mitigations:
- Use the source LSN / commit timestamp as a sequence number; MERGE only if `s.source_lsn > t.last_lsn`
- Partition Kafka topics by primary key so per-row order is preserved within a partition

### Schema Evolution Under CDC

When the source DB adds a column, your CDC pipeline must handle it without breaking.

- **Debezium**: emits schema change events on a separate topic; downstream readers can pick up new columns
- **Iceberg / Delta sinks**: "schema evolution mode" can `MERGE WITH SCHEMA EVOLUTION` to add columns automatically

**The dangerous case:** column rename or type change. These almost always require a coordinated source + sink + consumer migration.

---

## ADVANCED — Production Concerns

### Replication Slots / Privileges

CDC requires source-side configuration:

| Source | Required setup |
|---|---|
| **Postgres** | `wal_level = logical`, replication user, publication on tables, replication slot |
| **MySQL** | Binary logging (`log_bin = ON`), `binlog_format = ROW`, `binlog_row_image = FULL` |
| **MongoDB** | Replica set (oplog requires it) |
| **DynamoDB** | Enable Streams on the table |

**Operational note:** Postgres replication slots hold WAL until consumed. A stuck CDC consumer means WAL fills disk. Monitor slot lag religiously.

### Throughput, Lag, and Operational Reality

CDC pipelines are 24/7 systems. Things to monitor:

| Metric | Why |
|---|---|
| **CDC tool consumer lag** (binlog position) | Are we keeping up with source writes? |
| **Source replication slot size / WAL retention** | Will we run out of disk if consumer stalls? |
| **Per-table message rate** | Detect anomalies (10x normal volume = something happened upstream) |
| **Sink merge duration** | Is the MERGE keeping up? |
| **Time-from-source-commit-to-sink-visible** | End-to-end latency SLO |
| **Schema change events** | Did anyone deploy a migration? |

### Common Failure Modes

| Failure | Cause | Fix |
|---|---|---|
| WAL fills disk | CDC consumer stalled | Page on slot lag; have replay procedure |
| Out-of-order updates apply old over new | Insufficient ordering | Partition by PK; add LSN sequence column to MERGE |
| Schema drift breaks pipeline | New column in source | Auto-evolve schemas in sink (with guardrails) or alert + handle |
| Initial snapshot takes hours | Large table, no parallelism | Parallel snapshot by primary key range |
| "Phantom" deletes | Detached transactions, replicas, weird DB configs | Investigate per source; usually source-config issue |
| Duplicate events on restart | At-least-once delivery | Always use idempotent sinks (MERGE by PK) |

### Handling Hard Deletes vs Soft Deletes

Hard delete in source → CDC emits `op=d`. Sink can:
- Apply the delete (mirror semantics)
- Mark as deleted (`is_deleted = true`) and keep the row for audit
- Move the row to an archive table

The right choice depends on requirements (compliance often wants hard deletes for GDPR; audit often wants soft deletes for traceability). Articulate the choice in interviews.

### Capturing Foreign-Key-Affected Joins

CDC gives you per-table changes. If your downstream analytics joins `users` and `orders`, you have to think about *both* streams:

- A `users.region` update doesn't show up in `orders` until the downstream join is recomputed
- Stream-stream joins (Flink) can handle this with versioned tables, but they're complex
- Most production setups recompute the join periodically (every N minutes) rather than streaming the join itself

### The "Outbox Pattern" — Best Practice for Application-Driven CDC

Naive CDC reads the same tables your application writes. This couples analytics to the application schema and exposes internal columns to downstream consumers.

**Outbox pattern:** application writes events to a dedicated `outbox` table in the same transaction as the business write. CDC reads only `outbox`. The outbox event format is the *public contract* for downstream.

```sql
BEGIN;
INSERT INTO orders (...) VALUES (...);
INSERT INTO outbox (event_type, payload, created_at)
  VALUES ('order_created', '{...}', now());
COMMIT;
```

Now CDC produces a stream of versioned, intentional events instead of accidental schema-coupled rows. This is increasingly the recommended pattern for systems where DE and engineering need clean contracts.

### CDC into a Warehouse vs Lakehouse

**Warehouse-native (Snowpipe Streaming, Redshift streaming ingestion, BigQuery streaming inserts):**
- Simpler setup, fewer moving parts
- Tighter coupling to one engine
- Costs can balloon at high throughput
- Limited transformations on the way in

**Lakehouse with Iceberg/Delta/Hudi:**
- More flexibility (multiple engines, including ad-hoc Spark / Athena)
- Compaction and partition strategy under your control
- Better fit when ML / data science also need raw history
- More moving parts to operate

For mid-level AWS DE interviews: knowing both, and being able to articulate when each fits, is the bar.

### Reverse ETL Note

CDC pushes source → analytics. **Reverse ETL** (Hightouch, Census) pushes warehouse → operational systems (CRMs, ad platforms, marketing tools). Same pipeline mentality, opposite direction. Worth being able to mention in conversation.

---

## Worked Pipeline — Postgres → Iceberg via Debezium

```
[Postgres prod]
  │ replication slot with publication on selected tables
  ▼
[Debezium connector (Kafka Connect on MSK Connect)]
  │ topics: db.public.users, db.public.orders, ...
  ▼
[Kafka (MSK)]
  │
  ▼
[Spark Structured Streaming]
  - Reads each topic
  - Deduplicates by (pk, source_lsn) within each micro-batch
  - MERGE INTO iceberg.silver.users / orders
  - Writes append-only iceberg.bronze.user_changes for history
  ▼
[Iceberg tables on S3, Glue catalog]
  │
  ▼
[Athena / Trino / dbt / Spark for downstream analytics]
```

Interview-relevant choices:
- Postgres logical replication, `wal_level=logical`, replication slot with monitoring on slot size
- Debezium with snapshot=`initial` for first load, then streaming
- Per-table topic, partitioned by primary key for ordering
- Spark Streaming over Flink chosen because team's Spark expertise is deeper
- Both bronze (full history, append-only) and silver (latest state, MERGE-applied)
- Iceberg over Hudi because the rest of the lakehouse is Iceberg
- Compaction job nightly to keep silver tables small-files-free
- Lag SLO: 5 minutes p95 from source commit to silver visibility

---

## Interview-Ready Cheat Sheet

**"What's CDC?"** Capturing every insert/update/delete from a source DB and emitting it as a stream. Replaces nightly snapshots with low-latency, low-source-impact replication.

**"Log-based vs polling-based?"** Log-based reads the DB transaction log — captures deletes, real-time, low source impact. Polling reads tables on a cadence — misses deletes, requires updated_at hygiene, can hammer source.

**"How do you apply changes idempotently?"** MERGE INTO target USING changes ON pk; handle delete via WHEN MATCHED AND op = 'd' THEN DELETE. Add LSN sequence to skip out-of-order applies.

**"What's the outbox pattern?"** Application writes to a dedicated `outbox` table in the same transaction as business writes. CDC reads only outbox. Decouples analytics contracts from internal schema.

**"How do you handle the initial backfill?"** Snapshot phase (paginate the source) + streaming phase (tail log). Most CDC tools handle the transition. Ensure idempotent sinks so restarts don't corrupt state.

**"Postgres CDC operational risks?"** Replication slot retains WAL until consumed. Stuck consumer → WAL fills disk → source DB outage. Monitor slot lag, alert aggressively.

**Quick trade-off pairs:**
- Snapshot vs CDC: simple but slow vs near-real-time but operational
- Latest-state mirror vs append-only history: current state vs auditability
- Direct table CDC vs outbox: leak source schema vs explicit contract
- Warehouse-native streaming vs lakehouse merging: simpler vs more flexible

---

## Resources & Links

- [Debezium documentation](https://debezium.io/documentation/) — start here
- [Confluent — Streaming Database Changes with Debezium](https://www.confluent.io/blog/cdc-and-streaming-analytics-using-debezium-kafka/)
- [AWS DMS docs](https://docs.aws.amazon.com/dms/) — including DMS to Kinesis / S3
- [Outbox pattern overview — Microservices.io](https://microservices.io/patterns/data/transactional-outbox.html)
- [Apache Hudi CDC patterns](https://hudi.apache.org/docs/cdc/) — purpose-built for it
- [Materialize and the streaming database future](https://materialize.com/blog/) — for the streaming-warehouse-as-CDC-sink angle

*Next: [dbt — Modular SQL Transforms](./10-dbt.md) — once your data lands in the warehouse/lakehouse, dbt is how you transform it.*
