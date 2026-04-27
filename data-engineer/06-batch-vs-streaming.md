# Batch vs Streaming — Kafka, Kinesis, Flink, and Exactly-Once

**Phase:** 1 (Foundations)
**Difficulty progression:** Beginner → Intermediate → Advanced
**Last updated:** April 24, 2026
**Related:** [ETL vs ELT & Pipeline Patterns](./04-etl-vs-elt.md) · [CDC](./09-cdc.md) · [Spark Fundamentals](./07-spark-fundamentals.md) · [AWS Data Stack](./12-aws-data-stack.md)

---

## Why This Topic Matters

Streaming has gone from "nice to have" to "expected literacy" for mid-level DEs. You won't necessarily build streaming pipelines daily, but you *will* be asked to compare batch vs streaming, explain exactly-once, design a real-time analytics pipeline, or articulate when streaming is overkill (it usually is).

---

## BEGINNER — The Fundamentals

### Batch vs Streaming — Five Real Differences

| | Batch | Streaming |
|---|---|---|
| Data shape | Bounded ("yesterday's events") | Unbounded ("the events as they happen") |
| Trigger | Time / cron / external sensor | New record arrives |
| Latency | Minutes to hours | Milliseconds to seconds |
| State | Computed per run, discarded | Accumulated and managed across time |
| Failure model | Retry the run | Replay from a checkpoint / offset |

**Honest mentor advice:** for most analytical use cases, batch (or micro-batch every 5–15 min) is sufficient and 10× cheaper than streaming. Push back on streaming for "we want it real-time" without a concrete sub-minute SLA tied to user value.

### When Streaming Actually Earns Its Cost

| Use case | Why streaming |
|---|---|
| Fraud detection | Decisions must happen in seconds before charge completes |
| Personalization (real-time recs) | Stale = useless; user already left |
| Operational alerting | Latency directly costs revenue / safety |
| CDC into a downstream warehouse | Lower latency for analytics-on-source data |
| IoT telemetry | Volume + latency requirements |
| Real-time dashboards (sub-minute) | Business decisions on stale data are bad |

### Streaming Vocabulary

| Term | Meaning |
|---|---|
| **Event** | A single record (a user clicked, a sensor measured) |
| **Topic / Stream** | An ordered, durable log of events (Kafka topic, Kinesis stream) |
| **Partition / Shard** | A parallelism unit within a topic; ordering is preserved within a partition |
| **Offset** | Position of a consumer in a partition |
| **Producer / Consumer** | Writer / reader of events |
| **Consumer group** | Coordinated set of consumers; each partition consumed by one group member |
| **Watermark** | Threshold time below which all events are assumed seen (handles late data) |
| **Checkpoint** | Saved consumer state for recovery |

---

## INTERMEDIATE — Kafka, Kinesis, and Stream Processors

### Kafka in 30 Lines

A topic is an append-only log split into partitions. Producers append events; consumers read at their own pace, tracking offsets. The log is durable for a configured retention (hours, days, forever).

Key properties:
- **Ordering preserved within a partition.** Cross-partition ordering doesn't exist.
- **Fan-out via consumer groups.** Multiple independent consumers can read the same topic at their own offsets.
- **Replay.** Reset offsets to re-process history (assuming retention covers it).
- **Exactly-once support** (with idempotent producers + transactional writes).

**Producer key matters:** Kafka hashes the message key to choose a partition. Same key → same partition → preserved order for that key. Choose your key for ordering needs (`user_id` if you want all events for a user in order).

```
Topic: user_events  (3 partitions)

partition 0: [u_5: signup] [u_5: login] [u_5: click]    ← all u_5 events here
partition 1: [u_3: ...]    [u_3: ...]
partition 2: [u_99: ...]   [u_99: ...]
```

### Kinesis (the AWS twin)

Same shape as Kafka, AWS-managed:

| Concept | Kafka | Kinesis |
|---|---|---|
| Topic | Topic | Stream |
| Partition | Partition | Shard |
| Position | Offset | Sequence number |
| Replay | By offset | By sequence number / timestamp |
| Retention | Configurable (default 7d) | 24h (default) up to 365d |
| Throughput unit | Per-partition (configurable) | 1 MB/s in or 2 MB/s out per shard |

| Feature | Kafka | Kinesis |
|---|---|---|
| Ordered partitions | ✅ | ✅ (per shard) |
| Multi-consumer | ✅ (consumer groups) | ✅ (enhanced fan-out for low latency) |
| Replay | ✅ | ✅ within retention |
| Exactly-once | ✅ (transactions) | At-least-once; build idempotency into consumers |
| Schema registry | Confluent / open-source | AWS Glue Schema Registry |
| Operations | You run it (or Confluent Cloud, MSK) | Fully managed |

**On AWS, MSK** (Managed Kafka) is the path most production shops take when they want Kafka semantics with less ops burden than self-hosted Kafka. **MSK Serverless** removes capacity sizing entirely.

### Stream Processors — Where Logic Lives

A stream processor reads from one or more streams, transforms events, maintains state, and writes results.

| Processor | Strengths |
|---|---|
| **Flink** | Best-in-class true streaming; exactly-once with state checkpoints; complex event-time semantics |
| **Spark Structured Streaming** | Micro-batch (sub-minute); great if your team already knows Spark; less low-latency |
| **Kafka Streams** | Java library; lightweight; runs alongside your services; great for simple transforms |
| **ksqlDB** | SQL on Kafka; great for analysts; limited for complex pipelines |
| **AWS Kinesis Data Analytics (Managed Flink)** | Serverless Flink on AWS |
| **AWS Lambda → Kinesis** | Simplest serverless streaming for low-volume, stateless transforms |
| **Materialize / RisingWave** | "Streaming SQL warehouse" — incrementally maintained materialized views over streams |

**Practical mid-level choice for AWS-first stack:**
- Simple transforms / fan-out → Lambda or Kinesis Firehose
- Stateful aggregations / joins → Managed Flink (Kinesis Data Analytics)
- Already on EMR / Databricks → Spark Structured Streaming
- Simple analytics on streams → Materialize / ksqlDB

### Windowing

Streaming is unbounded — to compute aggregates you need *windows*.

| Window | Behavior | Example |
|---|---|---|
| **Tumbling** | Fixed-size, non-overlapping | "events per 5-minute bucket" |
| **Hopping / Sliding** | Fixed-size, overlapping | "5-min window every 1 min" |
| **Session** | Closed by inactivity gap | "user session = events with no gap > 30 min" |
| **Global** | All events, with custom triggers | Less common |

Windows belong to *event time* (when did it happen) — not *processing time* (when did we see it). Otherwise late events break aggregates.

### Watermarks — Handling Late Data

```
Watermark at time T means: "I assume all events with timestamp ≤ T have arrived."
```

Trade-off:
- **Conservative watermark** (move slowly): few late events dropped, higher latency
- **Aggressive watermark** (move fast): low latency, more events dropped or trigger window re-emissions

In Flink:
```java
WatermarkStrategy
  .<Event>forBoundedOutOfOrderness(Duration.ofSeconds(30))  // tolerate 30s out-of-order
  .withTimestampAssigner((e, ts) -> e.eventTime());
```

Late-arrival policies:
- Drop them (default)
- Allow them with **allowed lateness** — re-emit window
- Send them to a side output for inspection

### Exactly-Once Semantics

The phrase that confuses everyone. Three layers must align:

1. **Producer → broker idempotent** — Kafka idempotent producer; Kinesis: requires consumer-side dedup
2. **Broker → consumer at-least-once with offset commit on success** — standard
3. **Consumer → sink exactly-once** — the hard part

Achieved via:
- **Idempotent sink** — consumer writes are deterministic by event_id; re-runs don't duplicate
- **Transactional sink** — commit offset and write to sink atomically (Kafka transactions, Flink two-phase commit)
- **Exactly-once stateful processing** — Flink checkpoints state and offsets atomically

**Most production systems target "at-least-once with idempotent sinks,"** which is functionally exactly-once and operationally simpler.

### Backpressure

When a consumer can't keep up with a producer, you have backpressure. Options:

| Strategy | When |
|---|---|
| Buffer (queue) | Brief bursts; queue grows during spike, drains after |
| Scale consumers (more partitions / parallelism) | Sustained higher load |
| Drop (sample) | When stale data is fine but late data is worse |
| Block producers | Critical correctness; producers slow down to match |
| Spill to disk / S3 | Spark's approach when memory full |

Real production systems combine these — alerts on lag (Kafka consumer lag, Kinesis iterator age) trigger autoscaling, with spillover paths for emergencies.

---

## ADVANCED — Production Patterns

### The Lambda → Kappa Story

Recap from [Warehouses & Lakehouses](./03-warehouses-lakes-lakehouse.md):

**Lambda architecture** (Hadoop era):
- Batch layer (slow, accurate) + Speed layer (fast, approximate) + Serving layer
- Two pipelines doing the same thing → bug-for-bug parity nightmare

**Kappa architecture** (modern):
- One streaming pipeline; reprocessing = replay from log
- Requires durable, replayable streams (Kafka with long retention)
- Stream processor with state checkpoints (Flink)

Most modern stacks pick Kappa-flavored. The "reprocess history" story still requires careful design — replaying a year of events through a stateful processor is its own engineering challenge.

### Streaming + Lakehouse — The Modern Pattern

```
Producers → Kafka/Kinesis → [Stream processor] → Iceberg/Delta/Hudi tables
                            (Flink / Spark Streaming)        │
                                                             ▼
                                                Same tables read by analytics
                                                (Trino, Athena, Spark, dbt)
```

The lakehouse format provides:
- Atomic appends (no duplicate-write corruption)
- Snapshots for time travel
- Schema evolution
- Consumers see consistent point-in-time views

**Hudi was built specifically for this** (CDC into lakehouse with merge-on-read), but Iceberg and Delta have caught up. See [08-table-formats.md](./08-table-formats.md).

### CDC into Streams

Change-data-capture (debezium, AWS DMS) emits database changes as streaming events. Common pattern:

```
Postgres WAL → Debezium connector → Kafka topic per table → stream processor → lakehouse table
```

This gives you near-real-time analytics on operational source data without hammering the source DB. Deep dive in [09-cdc.md](./09-cdc.md).

### Stateful Stream Processing

Stateful operations (joins across streams, sessionization, dedup, running aggregates) require state stores per partition. Flink, Kafka Streams, and Spark Structured Streaming all manage this — but at scale, state is the operational complexity.

| Concern | Mitigation |
|---|---|
| State size growth | TTL on state, periodic cleanup, compaction |
| State recovery on failure | Checkpoints to durable storage (S3, HDFS) |
| State migration on schema change | Versioned state, migration logic |
| Operator-local state vs distributed | Flink's RocksDB state backend, Spark's stateful operators |

### Stream-Stream Joins

Joining two streams is non-trivial because event arrivals don't align in time.

```
Stream A: order_events
Stream B: payment_events
Join: order with corresponding payment within 30 minutes
```

Patterns:
- **Time-bounded inner join** — keep state for both sides for 30 min, emit when match found
- **Interval join** — bound by event-time interval (Flink)
- **Temporal table join** — join stream against versioned table (CDC + reference data)

The state management is what makes stream-stream joins hard. If you can articulate that you understand the trade-offs, you're ahead of most candidates.

### Schema Registry

Streams without schemas turn into garbage by month 2. The schema registry stores the canonical schema for each topic and validates producers/consumers.

**Common formats and registries:**
- Avro + Confluent Schema Registry — the classic combo
- Protobuf + Confluent Schema Registry / Buf Schema Registry
- JSON Schema + various
- AWS Glue Schema Registry — AWS-native option

**Compatibility modes:** BACKWARD (consumers reading new data with old schema), FORWARD (consumers reading old data with new schema), FULL (both). Choose based on whether producers or consumers move first.

### Observability for Streams

| Metric | Why |
|---|---|
| Consumer lag (Kafka) / iterator age (Kinesis) | Are consumers keeping up? |
| Throughput (events/sec, MB/sec) | Capacity planning |
| Processing latency (event_time → output_time) | Are SLAs met? |
| Checkpoint duration / failure rate | Stateful processor health |
| Late-event count | Watermark too aggressive? |
| Sink write failures | Downstream health |

Tools: Prometheus + Grafana for metrics; OpenLineage for stream lineage; managed dashboards in Confluent Cloud / AWS console.

### Cost Control for Streams

Streams cost money 24/7. Cost levers:

- **Right-size partitions / shards.** Too few = bottleneck; too many = wasted compute / per-partition cost.
- **Compression** on producers (snappy, lz4, zstd) — significant savings on broker storage and network.
- **Retention policy.** Default Kafka retention is 7 days; set explicitly per topic.
- **Compacted topics** for changelog topics where only "latest per key" matters.
- **Tiered storage** (Confluent / MSK) — recent data on broker disks, older on S3.
- **Right-size stream processor parallelism** to actual load; autoscale where possible.

### When You'd Use Each AWS Streaming Component

| Component | Use |
|---|---|
| **Kinesis Data Streams** | Application-emitted events, you control producers/consumers |
| **Kinesis Data Firehose** | Stream → S3 / Redshift / OpenSearch with batching, no code |
| **MSK** | You want Kafka semantics, ecosystem (Connect, Streams) |
| **MSK Serverless** | Kafka without capacity sizing |
| **Managed Flink (Kinesis Data Analytics)** | Stateful streaming SQL/Java |
| **Lambda** | Lightweight transforms, low volume |
| **EventBridge** | Event routing across AWS services (not data engineering throughput) |
| **SNS/SQS** | Simple pub-sub or queue, not high-volume analytics |

Deep dive in [12-aws-data-stack.md](./12-aws-data-stack.md).

---

## Worked Example — Real-Time Fraud Detection

You're asked: "Design a fraud detection pipeline for credit card transactions."

```
Card swipe at terminal
    ↓
Producer: payments-service publishes to Kafka topic `txn.events`
    ↓
Flink job: stateful per-card window
    - Compute features: tx velocity, geo-distance from last txn, amount vs user mean
    - Score against model (loaded from S3, periodically refreshed)
    - If score > threshold → emit to Kafka topic `txn.flagged`
    ↓
Consumer: fraud-service reads `txn.flagged`, blocks or steps up auth
    ↓
Sink: also write all txns + scores to Iceberg table for offline analysis / model retraining
```

Interview-relevant choices to articulate:
- **Latency target** — sub-second; rules out batch entirely
- **Exactly-once on flagged** — yes, via Kafka transactions; double-flagging is bad
- **State per card_id** — keyed stream by card_id; bounded by TTL
- **Watermark** — small (5s); fraud uses processing time mostly
- **Model serving** — Flink loads model from S3, refreshes hourly; alternative: external model server
- **Backfill / replay** — replay topics to retrain or evaluate new models on historical data
- **Failure** — checkpoint to S3; on restart, replay from last checkpoint

Pieces a junior misses:
- Schema registry for `txn.events` — versioning, compatibility
- DLQ topic for malformed events — don't crash on one bad row
- Operational metrics — lag, score distribution drift, model freshness
- Cost — what's the per-event cost; how does volume scale to peak (Black Friday?)

---

## Interview-Ready Cheat Sheet

**"Batch vs streaming — when do you pick which?"** Streaming when latency is sub-minute and tied to user value (fraud, personalization, alerting). Batch (or micro-batch) for everything else — cheaper, simpler, easier to evolve.

**"Explain exactly-once."** Producer idempotent + at-least-once consumption + idempotent sink (or transactional commit). Most prod systems are "at-least-once with idempotent sinks," which is functionally exactly-once.

**"What's a watermark?"** A threshold time below which all events are assumed seen. Aggressive watermark = lower latency + more late drops. Conservative = higher latency + fewer drops.

**"Kafka vs Kinesis?"** Same model. Kafka = open, larger ecosystem, more ops (or use MSK / Confluent). Kinesis = fully managed, smaller ecosystem, AWS-tight. Pick by stack, ops capacity, ecosystem needs.

**"How does Flink achieve exactly-once?"** Periodic state + offset checkpoints to durable storage; on failure, restart from last checkpoint. With transactional sinks, sink writes are committed only at checkpoint barriers (two-phase commit).

**"How would you reprocess a stream?"** Reset consumer offsets to earlier point + ensure stream retention covers the window. State of the processor must be reset or recomputed. With Kappa architecture, this is the standard pattern.

**Quick trade-off pairs:**
- Kafka vs Kinesis: open ecosystem vs managed simplicity
- Tumbling vs sliding window: discrete buckets vs overlapping smoothness
- Aggressive vs conservative watermark: latency vs late-data correctness
- At-least-once + idempotent vs transactional exactly-once: simpler ops vs strict guarantee

---

## Resources & Links

- [Kafka — The Definitive Guide (free O'Reilly book by Confluent)](https://www.confluent.io/resources/kafka-the-definitive-guide-v2/)
- [Designing Data-Intensive Applications — Kleppmann](https://dataintensive.net/) — chapters 11 & 12 are essential
- [Apache Flink Documentation](https://flink.apache.org/) — especially the streaming concepts section
- [The Streaming Systems book — Tyler Akidau et al.](http://streamingsystems.net/) — the canonical reference
- [Confluent's "Streaming 101 / 102" articles](https://www.confluent.io/blog/streaming-systems-foundations/) — clear and practical
- [AWS MSK & Kinesis docs](https://docs.aws.amazon.com/msk/latest/developerguide/what-is-msk.html)

*Next: [Spark Fundamentals](./07-spark-fundamentals.md) — the most common batch and streaming engine you'll meet.*
