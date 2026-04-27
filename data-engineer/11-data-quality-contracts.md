# Data Quality & Contracts — Tests, Expectations, Schemas, and SLOs

**Phase:** 1 (Foundations)
**Difficulty progression:** Beginner → Intermediate → Advanced
**Last updated:** April 24, 2026
**Related:** [dbt — Modular SQL Transforms](./10-dbt.md) · [Governance, Lineage, Observability](./14-governance-lineage-observability.md) · [ETL vs ELT & Pipeline Patterns](./04-etl-vs-elt.md)

---

## Why This Matters

Data quality is what separates "ran a pipeline" from "shipped reliable analytics." Most production data incidents are quality incidents — silently bad data that broke a dashboard, a model, or a financial report. Companies treat this seriously enough that "data observability" and "data contracts" are now standard interview topics, even at the mid level.

If your resume claims a project, expect to be asked: "How did you know the data was right?"

---

## BEGINNER — The Quality Dimensions and the Basic Toolkit

### Six Dimensions of Data Quality

A working vocabulary that every interviewer recognizes:

| Dimension | Question it answers |
|---|---|
| **Accuracy** | Does the value reflect reality? |
| **Completeness** | Are all expected rows / columns present? |
| **Consistency** | Do values agree across systems? Do FKs resolve? |
| **Timeliness** | Did the data arrive within SLA? |
| **Uniqueness** | Are there duplicate rows where we expect uniqueness? |
| **Validity** | Does each value match its domain (range, format, enum)? |

**Why this taxonomy is useful in interviews:** when asked "how do you ensure data quality?", structuring an answer around these dimensions sounds organized and complete. "I'd test for accuracy via reconciliation, completeness via row-count checks, consistency via FK tests..."

### The Layers of Defense

| Layer | Where checks run |
|---|---|
| **Schema enforcement** | At the boundary — reject incompatible data at ingest |
| **Pipeline tests** | Inside transforms — assert before continuing |
| **Post-load validation** | After write — row counts, anomaly detection, contracts |
| **Observability monitoring** | Continuous — drift, freshness, anomalies in dashboards |
| **Reconciliation** | Compare against source of truth (e.g., warehouse vs. operational DB) |

A junior approach has tests in one layer; a mid-level approach has them in multiple, with explicit roles for each.

### dbt's Built-in Tests (Recap)

Quick reference from [10-dbt.md](./10-dbt.md):

```yaml
columns:
  - name: order_id
    tests:
      - unique
      - not_null
  - name: status
    tests:
      - accepted_values: {values: ['placed', 'shipped', 'delivered']}
  - name: customer_id
    tests:
      - relationships:
          to: ref('dim_customers')
          field: customer_id
```

These four (unique, not_null, accepted_values, relationships) cover the majority of basic checks. Add `dbt-utils` and `dbt-expectations` packages for more.

---

## INTERMEDIATE — The Real Toolkit

### Great Expectations (GE)

The dedicated Python data-quality framework. Defines "expectations" against datasets and runs validations.

```python
import great_expectations as gx

df.expect_column_values_to_not_be_null("user_id")
df.expect_column_values_to_be_unique("order_id")
df.expect_column_values_to_be_between("amount", min_value=0, max_value=100000)
df.expect_column_values_to_match_regex("email", r"^[^@]+@[^@]+\.[^@]+$")
df.expect_table_row_count_to_be_between(min_value=1000, max_value=10000000)
```

**Strengths over dbt tests:**
- More expectations available out of the box
- Works on Spark / Pandas / SQL
- Generates a "data docs" site with validation history
- Better for cross-system reconciliation tests

**Weaknesses:**
- Heavier setup than dbt tests
- Configuration sprawl in large projects
- Mostly batch-flavored

In practice, many shops use **both**: dbt tests for in-warehouse SQL transforms, GE for ingestion validation and cross-system checks.

### Anomaly Detection vs Hard Checks

| Hard checks | Anomaly checks |
|---|---|
| Pass / fail boolean | Score / threshold |
| `not_null`, `unique`, `accepted_values` | "Row count today is 5σ below trailing 30-day mean" |
| Catch known failure modes | Catch unknown / drift |
| Easy to implement | Need historical data + tuning |

A mid-level pipeline has both. Hard checks for known constraints; anomaly checks for "did the world change in a way we didn't expect."

### Freshness Checks

The most-overlooked quality dimension.

```yaml
# dbt source freshness
sources:
  - name: app
    freshness:
      warn_after: {count: 12, period: hour}
      error_after: {count: 24, period: hour}
    loaded_at_field: ingested_at
```

```sql
-- inline check (any SQL)
select max(updated_at) from silver.users
-- alert if older than expected
```

If `dbt source freshness` warns, you know upstream is stuck — *before* downstream consumers notice broken dashboards.

### Volume / Row-Count Checks

```sql
-- "this table should have between 80% and 200% of its 7-day trailing average"
with stats as (
  select count(*) as today, avg_count_last_7_days as expected
  from ...
)
select case when today between 0.8 * expected and 2.0 * expected then 'pass' else 'fail' end
```

Many incidents look like "the daily load only captured 2% of normal volume because a source connector silently failed." Volume checks catch these.

### Distribution / Schema Drift

A column that's typically 99% non-null suddenly is 50% non-null → something changed. Distribution checks compare today's stats against historical.

| Check | Example |
|---|---|
| Null rate change | `null_rate(col) > historical_null_rate * 2` |
| Cardinality drift | `count_distinct(col)` outside expected range |
| Mean/median drift | numeric col mean shifted > N standard deviations |
| New / disappearing values | Categorical col now has values not seen historically |

Tools: Monte Carlo, Bigeye, Soda, Anomalo, Sifflet, custom dbt macros. We'll cover tools more in [14-governance-lineage-observability.md](./14-governance-lineage-observability.md).

### Reconciliation

The hardest, most valuable check: prove the data in your warehouse matches the source.

```sql
-- "yesterday's order amount sum matches operational DB"
select
    (select sum(amount) from warehouse.fact_orders where order_date = current_date - 1) as warehouse,
    (select sum(amount) from operational.orders where order_date = current_date - 1) as source,
    abs(warehouse - source) as diff
```

Run reconciliation as part of nightly pipeline. Alert when diff exceeds tolerance.

This is the check that catches "we changed the join logic and silently undercounted refunds for two weeks." Worth the effort.

---

## ADVANCED — Data Contracts

### What Data Contracts Are

A **data contract** is an explicit, versioned agreement between a data producer and consumers about what data the producer will provide: schema, semantics, freshness, ownership, and SLAs. It moves the "this column dropped silently" failure mode left — into producer code, where it's caught at deploy time.

**Concrete contract elements:**
- Schema (column names, types, nullability)
- Primary keys / uniqueness
- Allowed value ranges
- Freshness SLAs ("loaded within 1 hour of source change")
- Ownership (who owns it; how to reach them)
- Versioning policy (how breaking changes are communicated)

### Why They Exist

Without contracts:
- Producer team renames a column → analytics breaks → DE finds out from a Slack message
- Producer adds a sketchy default → silently bad metrics
- Consumers don't know who to ask when it breaks

With contracts:
- Producer's CI catches schema changes that violate the contract
- Consumers can rely on a stable interface
- Breakage becomes a deploy-time concern, not a Tuesday-morning fire

### dbt Model Contracts (Recap)

```yaml
models:
  - name: dim_customers
    config:
      contract: {enforced: true}
    columns:
      - name: customer_id
        data_type: bigint
        constraints:
          - type: not_null
          - type: primary_key
      - name: email
        data_type: varchar
        constraints:
          - type: not_null
```

dbt enforces this at compile time — if the model's compiled SQL doesn't produce these columns and types, dbt errors. This is the easiest way to introduce contract thinking.

### Source-Side Contracts

Real contracts live where the producer is. Pattern:

```
[Producer service]
   ├── publishes to outbox table or Kafka topic
   └── schema declared in protobuf / avro / pydantic
       └── registered in schema registry
                  │
                  └── CI rejects PRs that break compatibility
```

Tools:
- **Confluent / AWS Glue Schema Registry** — for Kafka with Avro/Protobuf/JSON schema
- **Buf Schema Registry** — for Protobuf
- **JSONSchema in CI** — for REST API events
- **dbt-style YAML contracts** for warehouse tables

### Compatibility Modes

When evolving a schema, what's safe?

| Mode | Producers can: | Consumers see: |
|---|---|---|
| **BACKWARD** | Add optional fields, drop fields | Old consumers read new data fine |
| **FORWARD** | Add fields with defaults | New consumers read old data fine |
| **FULL** | Both above | Either order works |
| **NONE** | Anything | Brace yourself |

The default for production should be `BACKWARD` (consumers move at their own pace) or `FULL` (safest). `NONE` is for people who like Tuesday morning fires.

### Schema Evolution Failure Modes

| Change | Impact | Mitigation |
|---|---|---|
| Add nullable column | Safe | Done; consumers ignore unknown columns |
| Add non-nullable column | Breaking for producers writing old format | Provide default; deploy producers first |
| Drop column | Breaks any consumer using it | Deprecate (mark, alert) → migrate consumers → drop later |
| Rename column | Breaks all consumers | Add new, dual-write, migrate, drop old |
| Change type (compatible) | Some serializers break | Coordinated migration |
| Change type (incompatible) | Everything breaks | Don't. New column, migrate consumers, drop old. |

### SLOs and SLAs for Data

| | SLA | SLO |
|---|---|---|
| Audience | External (contractual) | Internal (engineering target) |
| Examples | "99.9% uptime, 1hr support response" | "p95 freshness ≤ 30 min" |
| Consequence of violation | Service credits / breach | Page / postmortem |

**Common data SLOs:**
- Freshness — "data updated within X minutes of source change"
- Completeness — "row count within ±N% of expected"
- Correctness — "reconciliation matches source within ±$0.01"
- Availability — "queries on this table succeed 99.9% of the time"

### Implementing SLOs

```
SLO definition (YAML / config)
   ↓
Continuous measurement (queries, observability tool)
   ↓
Error budget tracking (how much can we be over before paging?)
   ↓
Alerts (page on burn rate, not single missed window)
   ↓
Postmortem when SLO burned > N% in a quarter
```

Tools: Monte Carlo (built-in SLOs), Sifflet, Soda, custom dbt-based dashboards, Datadog alerts.

### Quality in Streaming Pipelines

Stream quality is harder because data is unbounded. Patterns:

| Pattern | Description |
|---|---|
| **Schema validation at ingest** | Schema registry rejects bad messages |
| **DLQ topics** | Bad messages routed to a side topic for inspection |
| **Sliding window aggregates with thresholds** | Alert when "events per minute" leaves expected band |
| **Sample-and-validate downstream** | Periodically sample stream output and run heavier validations |
| **Replay-based validation** | Run a quality check job by replaying a window of the stream against a separate sink |

### Observability Tooling Landscape

| Tool | Sweet spot |
|---|---|
| **Monte Carlo** | All-in-one observability with auto-anomaly detection; pricey |
| **Bigeye** | Strong on metrics-based monitoring |
| **Soda** | Open-source quality framework with cloud add-ons |
| **Anomalo** | ML-driven anomaly detection |
| **Great Expectations** | Open-source quality framework, batch-flavored |
| **Datafold** | Data diffs and CI integration; great for migrations |
| **Sifflet** | Newer entrant, lineage + quality |
| **Custom (dbt + Slack alerts + Grafana)** | Many shops still build their own |

For interviews: know the category exists, name 1-2 tools, articulate when you'd build vs buy.

### Build vs Buy

| Build | Buy |
|---|---|
| Small team, simple needs | Many sources / pipelines, business-critical |
| dbt tests + custom alerts cover 80% | Need automatic anomaly detection / lineage |
| Cost-sensitive / no budget | Have engineering budget for reliability |
| Want full control over checks | Want time-to-value over flexibility |

Common answer: start with dbt tests + GE + custom alerts; bring in a tool when you have 50+ pipelines and human review can't keep up.

---

## Worked Example — Quality Architecture for an E-commerce Pipeline

You're asked: "How would you ensure quality for an e-commerce orders pipeline?"

**Layered defense:**

1. **Source-side schema contract** — Operational team's `orders` outbox events have a registered Protobuf schema; CI blocks producer PRs that break BACKWARD compatibility.
2. **Ingest validation** — Debezium emits to Kafka; sink job rejects malformed events to DLQ; alert if DLQ rate > 0.1%.
3. **Bronze landing** — Raw JSON events in S3 (Iceberg); volume + freshness checks (dbt source freshness; alert if >30 min stale).
4. **Silver transformation tests** (dbt):
   - `unique` on `order_id`
   - `not_null` on critical fields
   - `relationships` from `fact_orders.customer_id` to `dim_customers.customer_id`
   - `accepted_values` on `status`
   - `dbt_utils.expression_is_true: amount >= 0`
5. **Volume anomaly check** — daily row count compared to 7-day trailing band; alert if outside ±25%.
6. **Reconciliation** — daily SUM(amount) matches operational DB total to within $0.01; nightly job; page on diff.
7. **Contract on `dim_customers` and `fact_orders`** — typed schema; downstream teams can rely on it; renames go through deprecate/migrate cycle.
8. **SLOs:** freshness p95 ≤ 30 min; correctness on reconciliation 99.9%; completeness within ±5% of forecast volume.
9. **Observability** — Monte Carlo (or custom Slack/Grafana) tracks all of the above; on-call rotation responds to alerts.
10. **CI** — slim CI on dbt PRs; data diff (Datafold-style) shows row-level diff vs prod for changed models.

This layered answer hits all the dimensions and tools — the kind of structure mid-level interviewers look for.

---

## Interview-Ready Cheat Sheet

**"How do you ensure data quality?"** Multi-layer: schema contracts at ingest, dbt/GE tests inside transforms, volume + freshness + reconciliation checks post-load, anomaly detection continuously, SLOs with paging.

**"What are data contracts?"** Explicit, versioned agreements between data producers and consumers about schema, semantics, freshness, ownership. Caught at deploy time, not Tuesday morning.

**"What's the difference between an SLA and an SLO?"** SLA: external/contractual ("we promise 99.9% with credits if breached"). SLO: internal target ("we aim for p95 freshness ≤ 30 min, with paging if we burn budget").

**"Hard checks vs anomaly detection?"** Hard checks are pass/fail rules for known constraints (unique, not_null, ranges). Anomaly detection catches unknown drift (volume spikes, distribution shift).

**"How would you handle a producer renaming a column?"** With a contract, that's a breaking change blocked at producer CI. Without one: deprecate (alert + dual columns), migrate consumers, drop old after grace period.

**"Build vs buy for data observability?"** Build for small/simple (dbt + GE + Slack); buy when you can't keep up with anomaly detection across many pipelines. Most teams hit the inflection point around 30-50 pipelines.

**Quick trade-off pairs:**
- Hard tests vs anomaly detection: known vs unknown failure modes
- dbt tests vs Great Expectations: SQL-only vs cross-system
- Source contracts vs ingest validation: producer-side vs consumer-side
- Build vs buy: control/cost vs time-to-value

---

## Resources & Links

- [Great Expectations docs](https://docs.greatexpectations.io/)
- [dbt — Tests and contracts](https://docs.getdbt.com/docs/build/data-tests)
- [Monte Carlo blog — data observability](https://www.montecarlodata.com/blog/) — good vendor blog with conceptual posts
- [Data Contracts — Andrew Jones](https://www.oreilly.com/library/view/driving-data-quality/9781098169299/) — the canonical book
- [Locally Optimistic — data quality posts](https://locallyoptimistic.com/) — community blog with strong DE quality content
- [Datafold — data diff](https://www.datafold.com/) — for CI-time row-level diffing

*Next: [AWS Data Stack](./12-aws-data-stack.md) — the services that show up on every AWS DE posting.*
