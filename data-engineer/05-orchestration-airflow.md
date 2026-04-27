# Orchestration & Airflow — DAGs, Sensors, and Modern Alternatives

**Phase:** 1 (Foundations)
**Difficulty progression:** Beginner → Intermediate → Advanced
**Last updated:** April 24, 2026
**Related:** [ETL vs ELT & Pipeline Patterns](./04-etl-vs-elt.md) · [AWS Data Stack](./12-aws-data-stack.md) · [dbt — Modular SQL Transforms](./10-dbt.md)

---

## Why Orchestration Matters

A pipeline isn't "code that runs on a schedule." A pipeline is a graph of dependent tasks with retries, alerts, SLAs, lineage, and idempotency. Orchestration is the layer that makes this real. Pretty much every DE job posting names Airflow, and increasingly Dagster or Step Functions. You need one deeply and the others conversationally.

---

## BEGINNER — The Core Concepts

### What an Orchestrator Does

| Need | Without orchestrator | With orchestrator |
|---|---|---|
| Run job at 2am | cron + email on failure | DAG with retries, alerts, history |
| Run B after A finishes | "we hope it's done by then" | Explicit dependency edge |
| Re-run yesterday | manual SQL hacking | Clear and re-run a task instance |
| Track failures | check logs | Web UI, history, search |
| Wait for upstream | flaky polling script | Sensor with timeout + retries |
| Pass data between steps | files in shared dir | XComs / asset passing |
| Audit trail | tribal knowledge | Run history, lineage |

### Airflow Vocabulary

| Term | Meaning |
|---|---|
| **DAG** | Directed Acyclic Graph — the pipeline definition (a Python file) |
| **Task** | A single unit of work (operator instance) |
| **Operator** | A class that does a kind of work (PythonOperator, SQLExecuteQuery, KubernetesPodOperator) |
| **Sensor** | An operator that waits for a condition (file lands, partition exists, time passes) |
| **DAG run** | One execution of a DAG (associated with a logical date) |
| **Task instance** | One execution of one task within a DAG run |
| **XCom** | Cross-task communication — small data passed between tasks |
| **Trigger rule** | Condition for a task to run (all_success, one_failed, all_done, …) |
| **Pool** | Concurrency limit grouping — "max 3 BigQuery jobs at once" |
| **Variable / Connection** | Runtime config / credentials |
| **Executor** | Backend that runs tasks (Local, Celery, Kubernetes, CeleryKubernetes) |

### A Minimal DAG

```python
from airflow.decorators import dag, task
from datetime import datetime, timedelta

@dag(
    dag_id="daily_orders",
    start_date=datetime(2025, 1, 1),
    schedule="0 2 * * *",        # 2am daily
    catchup=False,
    default_args={"retries": 2, "retry_delay": timedelta(minutes=5)},
    tags=["finance"],
)
def daily_orders():
    @task
    def extract(execution_date: str) -> str:
        # pull yesterday's orders → write to S3
        return "s3://bucket/orders/2025-01-15.parquet"

    @task
    def transform(input_path: str) -> str:
        # transform → write silver
        return "silver.orders"

    @task
    def validate(table: str):
        # row counts, NULL checks
        ...

    path = extract("{{ ds }}")     # ds = logical date (YYYY-MM-DD)
    table = transform(path)
    validate(table)

daily_orders()
```

The TaskFlow API (`@task` decorators) is the modern way; older DAGs use explicit `PythonOperator` and `>>` chaining. Both work.

### Schedule Intervals — the One Thing That Confuses Everyone

In Airflow, a DAG run for date `2025-01-15` runs *after* `2025-01-15` ends — it represents the *period* `2025-01-15 00:00 → 2025-01-16 00:00`. So `{{ ds }}` is always one period in the past relative to wall clock.

This means:
- For a daily DAG with `schedule="0 2 * * *"`, the run *triggered at 2am on 2025-01-16* has `ds = 2025-01-15`. You're processing yesterday's data.
- This is a feature: it gives you a stable, replayable date-keyed identity for each run.
- New users get confused that "today's run" represents yesterday. Once you internalize it, scheduling is sane.

### catchup, depends_on_past, max_active_runs

| Setting | What it does |
|---|---|
| `catchup=True` | When you deploy a DAG with a past `start_date`, run all missed intervals to catch up |
| `catchup=False` | Just start running from now (recommended default) |
| `depends_on_past=True` | A task instance won't run until the previous date's instance succeeded |
| `max_active_runs` | How many runs can be active at once (parallelism across dates) |
| `max_active_tis_per_dag` | Per-task concurrency limit |

**Default for a new pipeline:** `catchup=False`, `depends_on_past=False`, `max_active_runs=1` to start. Loosen as needs evolve.

---

## INTERMEDIATE — Where Mid-Level Interviews Live

### Operators You Should Know

| Operator | What |
|---|---|
| `PythonOperator` / `@task` | Run a Python function |
| `BashOperator` | Run a shell command |
| `SQLExecuteQuery` (provider) | Execute SQL on a connection |
| `KubernetesPodOperator` | Run a containerized task on K8s |
| `EmailOperator` | Send email (often replaced with Slack alerts) |
| `S3*` operators (provider) | S3 listing, copying, sensing |
| `EmrCreateJobFlow*` / `EmrAddSteps*` | Manage EMR clusters and steps |
| `GlueJobOperator` | Trigger AWS Glue jobs |
| `DbtCloudRunJob` / `dbt-bash + cli` | Run dbt from Airflow |
| `BranchPythonOperator` | Choose a branch of the DAG to run |
| `ShortCircuitOperator` | Skip downstream if condition is False |
| `TriggerDagRunOperator` | Trigger another DAG |

### Sensors

A sensor waits for a condition. Two execution modes:

| Mode | Behavior | Use |
|---|---|---|
| `poke` | Holds a worker slot, polls every `poke_interval` | Quick waits (≤ 5 min) |
| `reschedule` | Releases the slot between pokes | Longer waits |
| **Deferrable** sensors (modern) | Use the triggerer, no slot held while waiting | Production default |

**Always prefer deferrable / async sensors in modern Airflow.** A non-deferrable sensor that polls for 12 hours holds a worker slot the entire time — at scale this kills your concurrency.

### XComs and Data Passing

XCom = "cross-communication." Small values passed between tasks (return values from `@task` go through XCom).

**Rules of thumb:**
- XCom is small. Don't push a 10MB DataFrame.
- For data, write to S3 / a table and pass the *path* through XCom.
- Custom XCom backend (S3-backed) lets you transparently store larger values, but mostly you just pass IDs and paths.

### Trigger Rules

By default, a task runs when all upstreams succeed. You can override:

| Rule | When the task runs |
|---|---|
| `all_success` (default) | All upstreams succeeded |
| `all_failed` | All upstreams failed |
| `one_success` | At least one upstream succeeded |
| `one_failed` | At least one upstream failed |
| `none_failed` | No upstream failed (success or skipped both fine) |
| `all_done` | All upstreams done (any state) — useful for cleanup tasks |
| `none_skipped` | No upstream was skipped |

A common pattern is a `cleanup` task with `trigger_rule=all_done` so it always runs to release resources.

### Branching

```python
@task.branch
def pick_branch(ds: str):
    if is_holiday(ds):
        return "skip_load"
    return "run_load"
```

Returns the task_id (or list) to run; other branches are skipped. Combine with `none_failed` trigger rule downstream so the join task still runs.

### Pools and Concurrency

Pools cap concurrency for related tasks:

```python
PythonOperator(
    task_id="bigquery_load",
    pool="bigquery_pool",  # max 3 slots configured in Airflow UI
    ...
)
```

Use pools for:
- Rate-limited APIs (Stripe, Salesforce)
- Warehouses with concurrency limits
- Cost-sensitive operations

### Connections, Variables, Hooks

- **Connection** — credentials + endpoint config (e.g., a Postgres connection with host/port/user/password)
- **Variable** — typed key-value config (e.g., `slack_alert_channel = "#data-alerts"`)
- **Hook** — Python wrapper around a connection (`PostgresHook`, `S3Hook`)

For secrets, use a secrets backend (AWS Secrets Manager, HashiCorp Vault) — not the metastore DB.

### Idempotent DAG Patterns (must know)

Tie this directly to [04-etl-vs-elt.md — Idempotency](./04-etl-vs-elt.md#idempotency).

```python
@task
def transform(execution_date: str):
    # parameterize by date — same input, same output
    write_partition(
        table="silver.events",
        partition_col="event_date",
        partition_value=execution_date,
        df=read_and_transform(date=execution_date),
        mode="overwrite",       # idempotent partition replace
    )
```

Bad pattern (don't do this in interview answers):
```python
# NOT idempotent — reads "yesterday" relative to wall clock
df = read_source_where(f"ts > current_date - 1")
append_to_table("events", df)
```

Always parameterize by `{{ ds }}` (Airflow's logical date) or function args.

### Alerts and SLAs

```python
default_args = {
    "owner": "data-team",
    "email": ["data-alerts@example.com"],
    "email_on_failure": False,        # use slack/PagerDuty
    "on_failure_callback": slack_alert,
    "retries": 3,
    "retry_delay": timedelta(minutes=5),
    "retry_exponential_backoff": True,
    "sla": timedelta(hours=2),
}
```

**SLAs** alert you if a task didn't complete within the expected window — separate from "the task failed." Critical for upstream-of-deadline jobs (e.g., must complete before 8am exec dashboard).

In modern Airflow, the `on_failure_callback` to Slack/PagerDuty is the production pattern; email_on_failure is mostly noise.

### Backfills

```bash
airflow dags backfill --start-date 2024-01-01 --end-date 2024-01-31 daily_orders
```

For one task only:
```bash
airflow tasks clear -s 2024-01-01 -e 2024-01-31 daily_orders -t transform
```

The clear pattern is the standard way to re-run a fixed slice. Combined with idempotent tasks, backfills are simple. Without idempotency, they're a nightmare. See [04-etl-vs-elt.md](./04-etl-vs-elt.md).

---

## ADVANCED — Production and Modern Alternatives

### Airflow Architecture (You Will Be Asked This)

Components:

- **Webserver** — UI, API
- **Scheduler** — parses DAGs, schedules tasks, queues runs
- **Worker(s)** — execute tasks (CeleryExecutor, KubernetesExecutor)
- **Triggerer** — runs deferrable / async sensors and operators
- **Metastore (Postgres / MySQL)** — DAG state, task history, connections, variables
- **Message broker** (CeleryExecutor only) — Redis or RabbitMQ for task queue

In production:
- **Kubernetes Executor** for elastic scale: each task gets its own pod
- **Deferrable operators** for long-running waits (no worker slot held)
- **DAG serialization** — DAGs parsed once and stored serialized in metastore for fast UI/scheduler
- **HA**: multiple schedulers (Airflow 2 supports it), Postgres HA, web servers behind LB

**MWAA** (Amazon Managed Workflows for Apache Airflow) is AWS's managed service. Cloud Composer is GCP's. Both abstract operations.

### Common Anti-Patterns

| Anti-pattern | Fix |
|---|---|
| Heavy Python code at the top level of a DAG file | Move to imports / functions; top-level runs every parse |
| Reading external systems at parse time (DB query in DAG def) | Use templated fields, late binding |
| Massive monolithic DAGs (100+ tasks) | Split into linked DAGs or use task groups |
| `start_date = datetime.now()` | Pin to a fixed past date |
| Side effects in branching | Use ShortCircuit or Branch operator, not raise/skip mid-task |
| One DAG per table, but pipeline is one logical unit | Use TaskGroups |
| Treating XCom as a data bus | Pass references (S3 paths, table names), not data |

### Dynamic Task Mapping

Generate tasks at runtime based on data:

```python
@task
def list_files() -> list[str]:
    return s3_list("s3://bucket/raw/")

@task
def process(path: str) -> dict:
    ...

paths = list_files()
process.expand(path=paths)  # creates one task per path
```

Replaced the older "dynamic DAG generation" anti-pattern. Use this when you need fan-out parallelism over a runtime-determined list.

### Datasets / Asset-Aware Scheduling (Airflow 2.4+)

Modern Airflow lets you trigger DAGs based on dataset updates rather than time:

```python
my_dataset = Dataset("s3://bucket/silver/orders")

@dag(schedule=[my_dataset])
def downstream_consumer(): ...
```

When an upstream task writes to that dataset (with `outlets=[my_dataset]`), the downstream DAG triggers. This is closer to event-driven orchestration and pairs nicely with lineage.

### Modern Alternatives — When You'd Pick Each

**Dagster**
- Asset-centric (you declare *what data exists*, Dagster figures out tasks)
- First-class data quality, partitioning, IO managers
- Excellent for analytics-engineering shops; integrates beautifully with dbt
- Growing fast, less mature than Airflow operationally

**Prefect**
- Pythonic, decorator-first
- Smaller install, easier local dev
- Hybrid execution (run anywhere, orchestration in cloud)
- Good for engineering-heavy teams; smaller ecosystem than Airflow

**AWS Step Functions**
- Serverless, AWS-native
- State-machine model (JSON definition)
- Excellent for AWS-only stacks: Glue + Lambda + EMR
- No DAG observability beyond Step Functions UI; not great for data lineage
- Cheap for low-volume; can be expensive for high-cardinality workflows

**Mage / Kestra / Argo Workflows**
- Newer entrants, niche use cases (Mage = analytics, Argo = K8s-native)

**dbt Cloud / dbt Fusion**
- Orchestrates dbt specifically; not a general-purpose orchestrator
- Often combined with Airflow ("Airflow triggers dbt jobs")

### When to Pick What

| Scenario | Pick |
|---|---|
| Industry-standard, biggest community | Airflow |
| dbt-heavy analytics shop, want asset model | Dagster |
| Small team, Pythonic, fast to onboard | Prefect |
| AWS-only stack, serverless cost model | Step Functions |
| Already on K8s, want minimal new infra | Argo Workflows |

For interview purposes: **know Airflow deeply, know what Step Functions and Dagster bring**, and be able to articulate why each exists.

### Airflow + dbt — the canonical pattern

```
Airflow DAG
├── extract from sources → S3
├── load to warehouse (Snowflake/BQ/Redshift)
├── trigger dbt run (with --select stage / mart selectors)
├── trigger dbt test
└── notify downstream / publish dataset event
```

`Cosmos` (Astronomer) renders dbt projects as Airflow task graphs automatically — turning each dbt model into an Airflow task with proper dependencies and visibility. This is increasingly the standard.

### Observability and Lineage Hooks

- **OpenLineage** plugin for Airflow emits lineage events on each task run
- Airflow's own lineage support (`inlets` / `outlets`) feeds DAG-level lineage
- Datasets give you asset-level lineage in the UI
- Marquez is the open-source backend for OpenLineage; DataHub also consumes it

See [14-governance-lineage-observability.md](./14-governance-lineage-observability.md).

---

## Worked DAG — Hourly Stripe Pipeline

```python
from airflow.decorators import dag, task, task_group
from airflow.providers.slack.notifications.slack import SlackNotifier
from airflow.datasets import Dataset
from datetime import datetime, timedelta

stripe_silver = Dataset("snowflake://prod/silver/stripe_charges")

@dag(
    dag_id="stripe_hourly",
    start_date=datetime(2025, 1, 1),
    schedule="@hourly",
    catchup=False,
    max_active_runs=1,
    default_args={
        "retries": 3,
        "retry_delay": timedelta(minutes=2),
        "retry_exponential_backoff": True,
        "on_failure_callback": SlackNotifier(slack_conn_id="slack", channel="#data-alerts"),
    },
    tags=["finance", "stripe"],
)
def stripe_hourly():
    @task(pool="stripe_api_pool", pool_slots=1)
    def extract(data_interval_start) -> str:
        path = f"s3://raw/stripe/charges/dt={data_interval_start:%Y-%m-%d}/hr={data_interval_start:%H}/"
        pull_stripe_charges(start=data_interval_start, end=data_interval_start + timedelta(hours=1), out=path)
        return path

    @task
    def stage(path: str) -> str:
        return load_to_snowflake_stage(path, table="raw.stripe_charges")

    @task(outlets=[stripe_silver])
    def transform(table: str) -> str:
        run_dbt(select="silver.stripe_charges silver.dim_customer")
        return "silver.stripe_charges"

    @task
    def validate(table: str):
        assert_row_count_within_band(table, lookback_days=7, tolerance_pct=50)

    @task(trigger_rule="all_done")
    def cleanup():
        delete_old_staging_files(days=7)

    raw = extract()
    staged = stage(raw)
    silver = transform(staged)
    validate(silver) >> cleanup()

stripe_hourly()
```

What this DAG demonstrates for an interviewer:
- Idempotent partitioning by `dt` and `hr` (re-runs replace the same partition)
- Pool to respect Stripe API rate limits
- Retries with backoff for flaky API
- Slack alerting via callback (no email noise)
- Dataset outlet so downstream consumers can subscribe (event-driven)
- Late validation step (row counts within historical band)
- Cleanup task with `all_done` so it always runs

---

## Interview-Ready Cheat Sheet

**"What's the difference between a DAG and a task?"** DAG is the pipeline graph; tasks are the nodes. A DAG run is one execution at a logical date; a task instance is one task within that run.

**"What does `{{ ds }}` mean in Airflow?"** Logical execution date for the DAG run — the *start* of the period being processed. For a daily DAG, ds is yesterday's date when the run actually happens.

**"Why do you prefer deferrable sensors?"** Non-deferrable sensors hold a worker slot while waiting. At scale, you'll exhaust workers waiting for files. Deferrable sensors hand off to the triggerer and free the slot.

**"How would you handle a flaky API extract?"** Retries with exponential backoff, pool to control concurrency, on_failure_callback for alerting only after retries exhausted, and idempotent writes so the eventual successful run produces the right data.

**"Airflow vs Step Functions?"** Airflow: open-source, biggest ecosystem, complex to operate (use MWAA). Step Functions: serverless, AWS-native, cheaper at low volume, weaker lineage/observability story. Pick based on stack and team capacity.

**"What's a TaskGroup vs a SubDAG?"** TaskGroups are visual grouping in the UI with no separate execution context; replaced SubDAGs (which had isolation/observability problems). Use TaskGroups.

**"How do you do cross-DAG dependencies?"** Modern: Datasets (event-driven). Older: TriggerDagRunOperator or ExternalTaskSensor.

**Quick trade-off pairs:**
- Airflow vs Dagster: time-based + ecosystem vs asset-based + dbt-friendly
- Time schedule vs Dataset trigger: simple vs event-driven
- Pool vs max_active_runs: per-task vs DAG-wide concurrency
- XCom vs S3 path: small data vs large data

---

## Resources & Links

- [Apache Airflow docs](https://airflow.apache.org/docs/apache-airflow/stable/index.html)
- [Astronomer's Airflow Guides](https://docs.astronomer.io/learn) — practical, well-written
- [Airflow's "Best Practices" page](https://airflow.apache.org/docs/apache-airflow/stable/best-practices.html) — read it twice
- [Cosmos — dbt × Airflow integration](https://astronomer.github.io/astronomer-cosmos/)
- [Dagster docs](https://docs.dagster.io/) — for the asset-based model
- [AWS Step Functions Workshop](https://catalog.workshops.aws/stepfunctions/en-US) — hands-on serverless

*Next: [Batch vs Streaming](./06-batch-vs-streaming.md) — when orchestrated batch isn't enough.*
