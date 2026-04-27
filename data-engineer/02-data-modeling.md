# Data Modeling — Schemas, SCDs, and the Patterns Interviewers Test

**Phase:** 1 (Foundations)
**Difficulty progression:** Beginner → Intermediate → Advanced
**Last updated:** April 24, 2026
**Related:** [SQL Deep Dive](./01-sql-deep-dive.md) · [Warehouses & Lakehouses](./03-warehouses-lakes-lakehouse.md) · [dbt — Modular SQL Transforms](./10-dbt.md)

---

## Why Data Modeling Matters

A well-modeled warehouse makes every downstream query faster, every dashboard easier, and every analyst happier. A badly modeled warehouse means analysts write 200-line CTEs to answer "how many orders did we ship last week," and your pipeline costs balloon because everything joins everything.

Interviewers ask data modeling questions to see whether you understand the *why* behind the schema, not just the *what*.

---

## BEGINNER — The Core Vocabulary

### OLTP vs OLAP

The single most foundational distinction. See [03-warehouses-lakes-lakehouse.md](./03-warehouses-lakes-lakehouse.md) for the full story.

| | OLTP (transactional) | OLAP (analytical) |
|---|---|---|
| Purpose | Run the business — orders, payments, app reads/writes | Understand the business — dashboards, reports, ML features |
| Workload | Many small reads/writes, single-row | Few large reads, scan millions of rows |
| Schema | Normalized (3NF) — minimize duplication | Denormalized (star/snowflake) — minimize joins |
| Examples | Postgres, MySQL, DynamoDB | Snowflake, BigQuery, Redshift, Databricks SQL |
| Storage | Row-oriented | Columnar |

A DE's job is often to extract data from OLTP systems, reshape it into OLAP models, and serve it to analysts and ML.

### Normalization (Briefly — for OLTP)

| Normal form | Rule |
|---|---|
| 1NF | Atomic columns (no comma-separated lists, no nested arrays) |
| 2NF | 1NF + no partial dependencies on a composite primary key |
| 3NF | 2NF + no transitive dependencies (non-key cols depend only on the key) |

In practice, you'll rarely be quizzed on the formal NF definitions. The interview-relevant takeaway: **OLTP systems are typically modeled to 3NF for write integrity; analytical systems denormalize for read speed.**

### Dimensional Modeling — The Default Analytical Pattern

Coined by Ralph Kimball. The world is split into **facts** (events that happened) and **dimensions** (the context around them).

**Facts:** "Customer X bought Y units of product Z at price P at time T from store S using payment method M."

**Dimensions:** Customer, product, store, time, payment method.

A fact table has:
- Foreign keys to each dimension
- Numeric measures (quantity, amount, duration)
- A grain — what one row represents (one transaction line, one daily snapshot, etc.)

A dimension table has:
- A surrogate key (usually an integer)
- Descriptive attributes (name, category, address, …)
- Often a "natural key" / "business key" from the source system

### Star Schema vs Snowflake Schema

```
Star schema:
  fact_orders
    ├── dim_customer (flat, all customer attrs in one table)
    ├── dim_product
    ├── dim_store
    └── dim_date

Snowflake schema:
  fact_orders
    ├── dim_customer
    │     └── dim_country (customer's country normalized out)
    ├── dim_product
    │     └── dim_category (category normalized out)
    └── ...
```

| | Star | Snowflake |
|---|---|---|
| Joins per query | Fewer (1 hop) | More (multi-hop) |
| Storage | More (duplication in dims) | Less (normalized) |
| Query speed | Faster | Slower |
| Maintenance | Simpler | More complex |
| Modern recommendation | Default for analytics | Avoid unless dim cardinality is huge |

**Default to star.** Snowflake schemas were a sensible compromise when storage was expensive. On modern columnar warehouses, storage is cheap and join costs are higher than the storage savings.

### One Big Table (OBT)

The other end of the spectrum: denormalize *everything* into one wide table per analytical use case.

| | OBT |
|---|---|
| Pros | Zero joins, blazing fast scans, simple for BI tools |
| Cons | Massive duplication, hard to update dimension changes, schema sprawl |
| When | Real-time analytics, BI dashboards on Snowflake/BigQuery, when joins are expensive |

The modern stack increasingly mixes star + OBT — star for the source of truth, OBT for serving layer / BI.

---

## INTERMEDIATE — The Patterns You'll Be Tested On

### Slowly Changing Dimensions (SCDs)

What do you do when a customer's address changes? Does history matter? This is the most-asked data modeling interview question.

| Type | Behavior | Example |
|---|---|---|
| **SCD Type 0** | Never changes (immutable) | birth_date |
| **SCD Type 1** | Overwrite — keep only current value | typo fixes |
| **SCD Type 2** | Add a new row with effective_from / effective_to | customer address (history matters) |
| **SCD Type 3** | Add a column for previous value | "current region" + "previous region" |
| **SCD Type 4** | Mini-dimension or history table for fast-changing attrs | demographics that change often |
| **SCD Type 6** | Hybrid 1+2+3 — current value column + history rows | rare; complex |

**Type 2 is the one to know cold:**

```
customer_id  natural_key  name      city      effective_from  effective_to    is_current
1            CUST-100     A. Smith  Boston    2024-01-01      2025-03-15      false
2            CUST-100     A. Smith  NYC       2025-03-15      9999-12-31      true
```

Joining a fact to a Type 2 dim:
```sql
SELECT f.*, d.city
FROM fact_orders f
JOIN dim_customer d
  ON f.customer_natural_key = d.natural_key
 AND f.order_date >= d.effective_from
 AND f.order_date <  d.effective_to;
```

This gives you the customer's city *as of the order date* — historical accuracy. Use `is_current = true` for current-state queries.

**Surrogate keys vs natural keys:** Always join facts to dims on surrogate keys (the integer `customer_id`), not on the natural/business key, so SCD Type 2 history works out of the box.

### Fact Table Grains

The grain is the most important decision in fact table design. Mix grains and you double-count and lose your sanity.

| Grain | Example |
|---|---|
| **Transactional** | One row per individual event (one order line, one click, one page view) |
| **Periodic snapshot** | One row per entity per period (daily account balance, monthly subscription state) |
| **Accumulating snapshot** | One row per process instance, updated as it progresses (order: placed → paid → shipped → delivered) |

State the grain in a comment at the top of your DDL. "One row per order line per day" is more useful documentation than 50 column comments.

### Fact Table Patterns

| Pattern | Used for |
|---|---|
| **Additive measures** | Sum across any dimension (revenue, count) |
| **Semi-additive** | Sum across some dims but not others (account balance — sum across accounts but not across days) |
| **Non-additive** | Can't sum (ratios, averages) — store the components, derive the ratio |
| **Factless fact table** | Just FKs, no measures (events that happened — student attended class) |

### Modeling Hierarchies

| Approach | When to use |
|---|---|
| Flatten in dim (level1, level2, level3 columns) | Fixed-depth hierarchies (geography: country/state/city) |
| Adjacency list (parent_id) | Variable depth, simple structure |
| Path enumeration (`/electronics/computers/laptops/`) | Read-heavy, full path queries |
| Bridge table | Many-to-many or ragged hierarchies (org charts where some employees have multiple managers) |

### Many-to-Many Relationships

Use a **bridge table** between fact and dimension.

```
fact_orders ── bridge_order_promotions ── dim_promotion
```

Each row in the bridge represents one applied promotion. To get total revenue per promotion, join through the bridge — but watch for double counting; sometimes you need an "allocation factor" column.

---

## ADVANCED — Production Patterns and Trade-offs

### Data Vault — The Other Modeling Paradigm

Kimball isn't the only game. **Data Vault** (Dan Linstedt) splits data into:

- **Hubs** — business keys (one per business concept: customer, product, order)
- **Links** — relationships between hubs (many-to-many, time-stamped)
- **Satellites** — descriptive attributes around hubs/links, with effective dates

**Advantages:**
- Schema is highly insert-only (great for auditability and parallel ingestion)
- Easy to add new sources (new satellites, no remodeling)
- Survives schema changes well

**Disadvantages:**
- Heavy join graph for queries (often need a Kimball-style mart on top for analysts)
- Conceptually denser; team needs training

**When you'd use it:** Large enterprise warehouses with many source systems, strong governance/compliance needs (banking, healthcare), or where source schemas churn often. Most modern startups skip Data Vault and go straight to Kimball + dbt.

### Activity Schema — A Newer Pattern

Coined by Ahmed Elsamadisi (Narrator). One ultra-wide event log table:

```
ts  customer_id  activity              feature_json   revenue_impact
... user-1       'viewed_product'      {"sku":"X"}    null
... user-1       'added_to_cart'       {"sku":"X"}    null
... user-1       'completed_checkout'  {}             49.99
```

You compute every derived metric — sessions, funnels, retention — from this one log via window functions. Powerful for analytics, less suitable for high-cardinality dimensional querying.

### Wide vs Tall Tables

| | Wide | Tall (long) |
|---|---|---|
| Shape | Many columns, fewer rows | Fewer columns, many rows (e.g., key-value) |
| Use | BI dashboards, OBT serving | Logging, time series, sparse attrs |
| Pivoting | Already pivoted | Pivot at query time via [conditional aggregation](./01-sql-deep-dive.md#pivot--unpivot) |

### Modeling Semi-Structured Data

Modern warehouses (Snowflake `VARIANT`, BigQuery `STRUCT`/`ARRAY`, Postgres `JSONB`) let you store JSON natively.

**The pattern:**
1. Land raw events as JSON in a staging table.
2. Flatten / extract critical paths into typed columns in a "stage" model.
3. Build dimensional models on top of the stage models.

Don't query raw `VARIANT` from BI tools — too slow, too brittle. Materialize typed columns first.

### Late-Arriving Data

What happens when an order comes in dated three days ago, but your daily aggregates already ran?

| Strategy | Pros | Cons |
|---|---|---|
| Reprocess affected windows (rolling backfill) | Correct | Re-compute cost |
| Insert with original event date, accept eventual consistency | Cheap | Stale aggregates briefly wrong |
| Use a "process date" column separate from "event date" | Auditable | More complex queries |

The interview answer: **separate event time from process time, and design pipelines to be [idempotent](./04-etl-vs-elt.md#idempotency) over event-time windows.** Re-running a job for "yesterday" should produce the same result whether you run it now or next week.

### Surrogate Key Generation

| Approach | Notes |
|---|---|
| Auto-increment (Postgres `SERIAL`, MySQL) | Centralized, simple — but can't generate offline |
| UUID v4 | Distributed, no coordination — but random hurts indexes/sort keys |
| UUID v7 / ULID | Time-ordered UUIDs — UUID benefits without random index hurt |
| Hash of natural key | Deterministic — same input → same surrogate; great for dbt incremental models |
| Snowflake ID (Twitter-style) | Time + machine + counter; sortable, distributed |

**For warehouses, deterministic hash keys are increasingly the norm** — `MD5(CONCAT_WS('|', natural_key_cols))` — because they let you regenerate the dimension without losing FK relationships.

### Designing for Schema Evolution

Production data models change. Plan for it.

- **Add columns, don't repurpose them.** Renaming a column with semantics change is a silent disaster.
- **Default new columns to NULL or a sentinel.** Don't backfill in-place if you can avoid it.
- **Version schemas at the contract layer.** See [11-data-quality-contracts.md](./11-data-quality-contracts.md).
- **Use views as the public interface** to physical tables, so you can refactor underlying structures without breaking consumers.

### Modeling for Cost (Modern Warehouses)

On Snowflake/BigQuery/Redshift, your model directly affects compute cost. See [13-partitioning-performance.md](./13-partitioning-performance.md) for the deep dive.

Quick principles:
- Cluster / partition by the column you filter on most (usually a date).
- Avoid SELECT \*. Columnar engines bill by columns scanned.
- Materialize hot aggregates (incremental dbt models) instead of recomputing per query.
- Don't store nulls when "not applicable" suffices — but don't pretend "0" means "missing."

---

## Worked Example — Modeling an E-commerce Warehouse

You're given:
- `app_db.users` (Postgres) — user profile, mutable
- `app_db.products` — product catalog, mutable (price changes!)
- `app_db.orders`, `app_db.order_items` — orders and line items
- Stripe events stream — payment events
- Web analytics events — page views and clicks

**Target marts:**

```
dim_customer        SCD Type 2 on (email, address, plan_tier)
                    surrogate: customer_sk (deterministic hash)

dim_product         SCD Type 2 on (name, category, list_price)

dim_date            generated table, every date 2018-01-01 → today + 1y

fact_order_lines    grain: one row per order line per ship event
                    measures: quantity, unit_price, line_total, discount, tax
                    FKs: customer_sk, product_sk, order_date_key, ship_date_key

fact_payments       grain: one row per payment event (charge, refund, dispute)
                    measures: amount, fee
                    FKs: customer_sk, order_id, event_date_key

fact_web_events     grain: one row per page view (factless w/ session id)
                    used for funnel analysis via window functions
```

**Why these choices:**
- SCD Type 2 on customer because billing history matters (address-of-record at time of order is legally meaningful).
- Product price is captured *both* as a Type 2 dim attribute *and* in `fact_order_lines.unit_price` — never let downstream analysts compute revenue from current price; always use the price at the time of the order.
- Separate fact for payments because they have a different grain and lifecycle than orders (refunds happen later).
- `dim_date` so reports can join to it for fiscal week / fiscal year / business-day-of-month logic without redefining it everywhere.

This kind of walkthrough is exactly the kind of mid-level system-design answer interviewers want. See [15-system-design-de.md](./15-system-design-de.md) for more.

---

## Interview-Ready Cheat Sheet

**"Star schema vs snowflake schema?"** Star: dims are flat, fewer joins, more storage, faster queries. Snowflake: dims are normalized, more joins, less storage, slower. Default to star on modern warehouses.

**"How do you handle changing customer addresses?"** SCD Type 2: add a new row with effective_from / effective_to, set is_current flag, join facts to the row valid at the order date. Surrogate keys make this work transparently for downstream queries.

**"What's the grain of your fact table?"** State it explicitly. "One row per order line per day" beats every column comment. Mixing grains is the #1 way to corrupt analytics silently.

**"How would you handle a 'current and history' requirement?"** Type 2 dim with `is_current = true` filter for current state, or a separate "current dim" view. Don't try to model both in one row — leads to confusion.

**"Why not just normalize everything to 3NF?"** OLTP optimizes for write integrity (3NF). OLAP optimizes for read speed (denormalized). Joining 8 dim tables on every dashboard query is slow and expensive on columnar warehouses.

**Quick trade-off pairs:**
- Star vs snowflake: query speed vs storage / dim normalization
- Kimball vs Data Vault: simpler analytics vs auditability/extensibility
- Wide table (OBT) vs star: speed/simplicity vs reusability/governance
- Surrogate key (auto-int vs hash): centralized vs distributed/deterministic

---

## Resources & Links

- [Kimball Group Design Tips](https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/kimball-techniques/design-tips/) — the source
- [The Data Warehouse Toolkit (Kimball)](https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/books/data-warehouse-dw-toolkit/) — the canonical book
- [dbt — Building a Kimball dimensional model](https://docs.getdbt.com/blog/kimball-dimensional-model) — modern stack adaptation
- [Activity Schema spec](https://www.activityschema.com/) — Narrator's pattern

*Next: model an actual project end-to-end. The [projects/](../projects/README.md) folder will have one designed for this.*
