# dbt — Modular SQL Transforms, Tests, and Lineage

**Phase:** 1 (Foundations)
**Difficulty progression:** Beginner → Intermediate → Advanced
**Last updated:** April 24, 2026
**Related:** [SQL Deep Dive](./01-sql-deep-dive.md) · [Data Modeling](./02-data-modeling.md) · [Orchestration & Airflow](./05-orchestration-airflow.md) · [Data Quality & Contracts](./11-data-quality-contracts.md) · [Governance, Lineage, Observability](./14-governance-lineage-observability.md)

---

## Why dbt Matters

dbt has become table-stakes for modern analytics-engineering work and increasingly required in DE postings. It's the "T" in modern ELT — runs SQL transformations inside your warehouse, tests them, generates lineage, and produces docs. If you've never used dbt, you'll be at a disadvantage in interviews vs candidates who have.

The good news: dbt is straightforward once you grasp the model.

---

## BEGINNER — Concepts and Vocabulary

### What dbt Is (and Isn't)

dbt is a **transformation framework that runs SQL inside your warehouse**. It is *not* an orchestrator (use Airflow / Dagster), *not* an ingestion tool (use Fivetran / Airbyte / custom), and *not* a data warehouse (it runs on top of one).

| dbt does | dbt doesn't |
|---|---|
| Compile SQL with templating | Move data between systems |
| Manage model dependencies | Schedule pipelines (cron) |
| Run tests on data | Replace your warehouse |
| Produce lineage and docs | Do streaming |
| Version-control transforms | Provide BI tools |

### Project Structure

A dbt project is a directory of SQL + YAML:

```
my_project/
├── dbt_project.yml         # project config
├── profiles.yml            # warehouse credentials (often outside the repo)
├── models/
│   ├── staging/
│   │   ├── stg_orders.sql
│   │   ├── stg_users.sql
│   │   └── _stg_sources.yml
│   ├── intermediate/
│   │   └── int_orders_with_users.sql
│   └── marts/
│       ├── core/
│       │   ├── dim_users.sql
│       │   ├── fact_orders.sql
│       │   └── _core_models.yml
│       └── finance/
├── tests/                  # singular SQL tests
├── macros/                 # reusable Jinja/SQL macros
├── snapshots/              # SCD Type 2 snapshots
├── seeds/                  # CSV fixtures
└── analyses/               # one-off ad-hoc queries
```

### A Model Is Just a SELECT

Each `.sql` file in `models/` is one model. dbt wraps it in `CREATE TABLE / VIEW AS` and runs it against your warehouse.

```sql
-- models/staging/stg_orders.sql
{{ config(materialized='view') }}

select
    id           as order_id,
    customer_id,
    total_amount as amount,
    cast(created_at as timestamp) as ordered_at,
    status
from {{ source('app', 'orders') }}
where status not in ('test', 'voided')
```

`{{ source('app', 'orders') }}` references a raw table declared in YAML. `{{ ref('other_model') }}` references another dbt model. dbt resolves these into a DAG and a topological run order.

### Materializations

How dbt persists a model in the warehouse:

| Materialization | Behavior | Use for |
|---|---|---|
| **view** | `CREATE VIEW` — recomputes every read | Cheap, light staging models |
| **table** | `CREATE TABLE AS SELECT` — full rebuild each run | Mid-size dimensions, marts where freshness from rebuild is fine |
| **incremental** | First run = full build; subsequent runs = process new rows only | Large fact tables, append/update workloads |
| **ephemeral** | Inlined as a CTE in downstream models — never materialized | Reusable subqueries that don't deserve their own table |
| **snapshot** | SCD Type 2 — captures changes over time | Tracking changes to dimension data |

### Sources, Refs, and the DAG

```yaml
# models/staging/_stg_sources.yml
sources:
  - name: app
    schema: raw
    tables:
      - name: orders
      - name: users
```

```sql
-- a model
select * from {{ ref('stg_orders') }}
join     {{ ref('stg_users') }} using (user_id)
```

dbt parses every `ref()` and `source()` to build a DAG. `dbt run` executes models in dependency order, parallelizing where possible.

`dbt run --select staging+`  runs everything downstream of `staging`. `dbt run --select +fact_orders` runs everything upstream of `fact_orders`. The selector syntax is powerful and worth learning.

### A Three-Layer Convention

The community-standard layout maps to [the medallion / Kimball pattern](./03-warehouses-lakes-lakehouse.md#medallion-architecture-bronze--silver--gold):

| Layer | Purpose | Materialization default |
|---|---|---|
| **staging** | Rename / cast / light cleanup of one source table | view |
| **intermediate** | Joins, business logic, reusable computations | view or ephemeral |
| **marts** | Business-facing fact / dim tables | table or incremental |

This is *the* pattern interviewers expect. Use it.

---

## INTERMEDIATE — Tests, Macros, and Incremental Models

### Tests — dbt's Other Killer Feature

dbt has built-in tests for the most common data-quality assertions:

```yaml
# models/marts/_marts.yml
models:
  - name: fact_orders
    columns:
      - name: order_id
        tests:
          - unique
          - not_null
      - name: customer_id
        tests:
          - relationships:
              to: ref('dim_customers')
              field: customer_id
      - name: status
        tests:
          - accepted_values:
              values: ['placed', 'shipped', 'delivered', 'cancelled']
```

`dbt test` runs all tests; failures count and (by default) error if any rows fail.

**Beyond the four built-ins**, the `dbt-utils` and `dbt-expectations` packages offer dozens more:

- `dbt_utils.expression_is_true: amount >= 0`
- `dbt_utils.recency: (datepart='day', field='ordered_at', interval=2)` — "this table got updated within the last 2 days"
- `dbt_expectations.expect_column_values_to_be_between: ...`

**Singular tests** (`tests/*.sql`) are arbitrary SELECTs that should return zero rows:

```sql
-- tests/no_orphaned_orders.sql
select * from {{ ref('fact_orders') }}
where customer_id not in (select customer_id from {{ ref('dim_customers') }})
```

For deeper data quality patterns see [11-data-quality-contracts.md](./11-data-quality-contracts.md).

### Incremental Models

The most-tested intermediate dbt topic.

```sql
-- models/marts/fact_events.sql
{{ config(
    materialized='incremental',
    unique_key='event_id',
    on_schema_change='append_new_columns'
) }}

select
    event_id,
    user_id,
    event_ts,
    ...
from {{ ref('stg_events') }}
{% if is_incremental() %}
where event_ts >= (select coalesce(max(event_ts), '1900-01-01') from {{ this }})
                  - interval '3 days'   -- 3-day lookback window for late data
{% endif %}
```

What `is_incremental()` does:
- First run: returns `false`, runs the full SELECT, creates the table
- Subsequent runs: returns `true`, runs the SELECT with the date filter, then upserts via the configured strategy

**Incremental strategies** (warehouse-dependent):

| Strategy | Behavior |
|---|---|
| `append` | Just `INSERT` — only good when source guarantees no dupes |
| `merge` (default on most engines) | `MERGE` on `unique_key` — handles updates and inserts |
| `delete+insert` | Delete matching rows, then insert |
| `insert_overwrite` | Replace whole partitions (fast on Spark, BigQuery, Iceberg) |

**`--full-refresh`** rebuilds an incremental model from scratch — use when business logic changes or after backfills.

### Macros and Jinja

dbt models are Jinja-templated SQL. You can write macros for reusable SQL:

```sql
-- macros/cents_to_dollars.sql
{% macro cents_to_dollars(column_name, scale=2) %}
    round({{ column_name }} * 0.01, {{ scale }})
{% endmacro %}
```

Use it in a model:
```sql
select
    order_id,
    {{ cents_to_dollars('amount_cents') }} as amount,
    ...
```

**Common macro patterns:**
- `pivot()` — generate dynamic pivot SQL
- `get_column_values()` — query the warehouse at compile time to drive Jinja
- `dbt_utils.star()` — `SELECT *` minus excluded columns
- `dbt_utils.surrogate_key()` — deterministic hash key from columns

### Snapshots — dbt's SCD Type 2

```sql
-- snapshots/customers_snapshot.sql
{% snapshot customers_snapshot %}
{{ config(
    target_schema='snapshots',
    unique_key='customer_id',
    strategy='timestamp',
    updated_at='updated_at'
) }}
select * from {{ source('app', 'customers') }}
{% endsnapshot %}
```

dbt manages `dbt_valid_from` / `dbt_valid_to` columns automatically. Each `dbt snapshot` run captures a new version when source rows change. This is the canonical way to implement [SCD Type 2](./02-data-modeling.md#slowly-changing-dimensions-scds) in dbt.

**Strategy options:**
- `timestamp` — track changes by an `updated_at` column
- `check` — compare full row contents (slower, but works without timestamps)

### dbt Tests vs Data Contracts

Tests are *runtime* checks — "did the data we just produced satisfy these rules?" Contracts (newer dbt feature) are *schema-level* declarations — "this model promises these columns with these types."

```yaml
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
```

If a model's compiled SQL doesn't match the contract, dbt errors before running. Critical for downstream-facing models that other teams consume.

### Documentation

```yaml
models:
  - name: fact_orders
    description: One row per shipped order line, including refunds.
    columns:
      - name: order_id
        description: Unique identifier for the order.
```

`dbt docs generate` + `dbt docs serve` produces a browsable site with descriptions, lineage graph, and column-level metadata. This is also the lineage view many DEs / analysts use day-to-day.

---

## ADVANCED — Production dbt

### Project Structure for Real Teams

Beyond toy projects, organize models by domain:

```
models/
├── staging/                  # one folder per source system
│   ├── stripe/
│   │   ├── stg_stripe__charges.sql
│   │   └── stg_stripe__customers.sql
│   ├── salesforce/
│   └── _sources.yml
├── intermediate/
│   └── finance/
│       └── int_finance__monthly_recurring_revenue.sql
└── marts/
    ├── finance/
    ├── product/
    └── core/
```

Naming convention: `{layer}__{domain}__{purpose}` — easy to grep and tab-complete.

### CI/CD with dbt

Standard pattern in modern teams:

1. **PR opened** → CI runs `dbt build` against a staging schema with only changed models + their downstream
2. **Tests run** on the changed models
3. **Slim CI** — `dbt build --select state:modified+ --defer --state path/to/prod-manifest`
   - Compare your branch's manifest against prod's; only build/test modifications
   - `--defer` lets unchanged refs resolve against prod, so you don't rebuild the whole project
4. **PR review** includes auto-generated lineage diff
5. **Merge** triggers production `dbt run` via Airflow/Dagster

Slim CI is the *single biggest dbt productivity feature* most teams under-use.

### Performance Patterns

| Pattern | When |
|---|---|
| Use `view` for thin staging | Cheap, fresh; warehouse handles compute on read |
| Switch to `table` if a view is read many times | Trade compute-on-read for compute-once-on-build |
| Use `incremental` for big append/upsert tables | Don't rebuild fact tables daily |
| Use `insert_overwrite` on partitioned tables | Spark, BigQuery, Iceberg — fast partition replace |
| Materialize hot intermediates as `table` | Don't compute the same join graph 10 times |
| Add `cluster_by` / `partition_by` configs | Push down predicates; cluster columns drive pruning |
| Use `+pre_hook` / `+post_hook` for index/stats refresh | Especially on Redshift / Postgres |

### Costs of Carelessness

| Anti-pattern | Cost |
|---|---|
| Full-refresh every fact model nightly | Compute hours; could be 10× cheaper as incremental |
| `select *` everywhere | Columnar engines bill for all columns scanned |
| Massive monolithic intermediate models | Hard to debug; recomputed in every downstream |
| Tests run as part of `dbt run` (they aren't) → forgetting to run tests | Stale data passes silently. Use `dbt build` (run + test) or CI gates |

### dbt + Iceberg / Delta / Hudi

dbt supports lakehouse table formats well:

```sql
{{ config(
    materialized='incremental',
    file_format='delta',           -- or 'iceberg' / 'hudi'
    incremental_strategy='merge',
    unique_key='event_id',
    partition_by=['event_date']
) }}
```

This unlocks dbt for the Spark/EMR/Databricks side of the stack, not just warehouse SQL.

### dbt Cloud vs dbt Core

| | dbt Core (OSS) | dbt Cloud |
|---|---|---|
| Cost | Free | Per-user |
| IDE | VS Code, your editor | Web IDE |
| Scheduler | None (use Airflow / cron) | Built-in |
| Documentation hosting | Self-host | Managed |
| Semantic layer | Limited | Full (MetricFlow) |
| CI/CD | DIY | Built-in |
| Best for | Engineering-heavy teams with Airflow already | Smaller teams, less ops, analyst-led |

**For interview purposes:** know both exist; most engineering-led DE shops run dbt Core orchestrated by Airflow / Dagster.

### dbt Mesh / Multi-project Pattern

For large orgs, splitting one giant dbt project into multiple via "dbt mesh" (cross-project refs, public/private model contracts) lets teams own their domains.

```yaml
# project A exposes a model
models:
  - name: dim_customers
    access: public
    contract: {enforced: true}

# project B references it
select * from {{ ref('project_a', 'dim_customers') }}
```

This is increasingly common at companies with 50+ DEs.

### When dbt Isn't the Answer

- Streaming/real-time transforms — dbt is batch (sub-minute streaming options exist but are limited)
- ML feature pipelines that need Python — pair with Snowpark / Spark / dbt-Python models
- Cross-system orchestration — dbt is for transforms; the orchestrator is for end-to-end pipelines
- ETL with heavy ingestion logic — dbt assumes data is in your warehouse already

---

## Worked Example — Building a "fact_orders" Model

Pseudocode walk that an interviewer would expect at mid-level:

```
sources:
  app.orders, app.order_items, app.users (raw landed by Fivetran or CDC)

staging/
  stg_orders.sql           view; cast types, rename, light filter
  stg_order_items.sql      view; same
  stg_users.sql            view; same

intermediate/
  int_orders__joined.sql   view; join orders ↔ order_items ↔ users (order line grain)
                           handles the "one row per line" decision

marts/core/
  dim_users.sql            table; current state of users (or use snapshots for history)
  fact_orders.sql          incremental; one row per order line per ship event
                           unique_key='order_line_id'
                           lookback 7 days
                           partitioned by order_date (Iceberg insert_overwrite)
                           tests: unique, not_null, relationships to dim_users

semantic / metrics:
  monthly_revenue = sum(amount) over fact_orders, filtered by status
  active_customers = count distinct user_id over rolling 30 days
```

Tests, contract on `fact_orders`, docs descriptions, lineage in dbt docs. Slim CI runs only changed models on PRs. Airflow runs `dbt build --select tag:nightly` at 2am. That's the full picture interviewers want.

---

## Interview-Ready Cheat Sheet

**"What is dbt?"** A SQL-based transformation framework. Models are SELECTs that dbt compiles, runs in dependency order, and tests against your warehouse. Replaces ad-hoc Python scripts and stored procedures for ELT-T.

**"Materialization options?"** view (cheap, recomputed each read), table (full rebuild), incremental (process only new rows + merge), ephemeral (inlined as CTE), snapshot (SCD Type 2 history).

**"How does an incremental model work?"** First run does a full select; subsequent runs filter by `is_incremental()` to read only new data, then merge/append/insert_overwrite into the existing table. `--full-refresh` rebuilds.

**"What kinds of tests does dbt support?"** Built-in: unique, not_null, accepted_values, relationships. dbt-utils and dbt-expectations packages add many more. Singular tests are arbitrary SQL that should return zero rows.

**"What's `ref()` and why does it matter?"** It references another dbt model. dbt parses all `ref()` calls to build the DAG and run order. Always use `ref()` instead of hardcoded table names — that's how the DAG works.

**"Snapshots vs incremental models?"** Snapshots track row-level changes over time (SCD Type 2). Incremental models efficiently process new data into a current-state table.

**"How would you set up CI?"** Slim CI: build only changed models + downstream, with `--defer` against prod manifest so unchanged refs don't rebuild. Tests gate the merge.

**Quick trade-off pairs:**
- view vs table: cheap-and-fresh vs compute-once-and-cache
- incremental vs full refresh: speed/cost vs simplicity
- dbt Core vs dbt Cloud: ops control vs managed convenience
- staging-intermediate-marts vs flat: scalable vs simple

---

## Resources & Links

- [dbt official documentation](https://docs.getdbt.com/) — start here
- [dbt fundamentals course (free)](https://learn.getdbt.com/) — official intro
- [dbt-utils](https://github.com/dbt-labs/dbt-utils) and [dbt-expectations](https://github.com/calogica/dbt-expectations) packages
- [Best practices guide](https://docs.getdbt.com/best-practices) — read it twice
- [GitLab's dbt project](https://gitlab.com/gitlab-data/analytics) — real-world OSS dbt project to study
- [The Analytics Engineering Handbook (dbt Labs)](https://www.getdbt.com/analytics-engineering)
- [Cosmos — running dbt projects in Airflow](https://astronomer.github.io/astronomer-cosmos/)

*Next: [Data Quality & Contracts](./11-data-quality-contracts.md) — go beyond dbt's built-in tests.*
