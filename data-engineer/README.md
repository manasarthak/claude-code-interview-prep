# Data Engineering — Interview Prep Hub

**Phase:** 1 (Foundations through Production)
**Target seniority:** ~2 YOE, climbing toward solid mid-level
**Cloud focus:** AWS-primary (with secondary GCP/Snowflake/Databricks notes)
**Last updated:** April 24, 2026

---

## Mentor's Note — Read This First

You're at an interesting career inflection point. With ~2 YOE, you're past "fundamentals only" interviews but not yet expected to architect petabyte-scale systems from scratch. The interviews you'll face land in a specific zone:

- **Strong SQL is non-negotiable.** The single most-tested topic across every DE interview, regardless of company. If your SQL isn't sharp, fix that before anything else. See [01-sql-deep-dive.md](./01-sql-deep-dive.md).
- **One pipeline tool deeply, others superficially.** Pick Airflow as your "deep" one (most common in postings) and know enough Dagster/Prefect/Step Functions to discuss trade-offs.
- **One processing engine deeply.** Spark via PySpark. Know it cold — partitioning, shuffles, broadcast joins, AQE. See [07-spark-fundamentals.md](./07-spark-fundamentals.md).
- **Cloud + warehouse + lake formats.** AWS stack ([12-aws-data-stack.md](./12-aws-data-stack.md)) + columnar warehouses ([03-warehouses-lakes-lakehouse.md](./03-warehouses-lakes-lakehouse.md)) + at least one open table format ([08-table-formats.md](./08-table-formats.md)).
- **System design at your level looks like:** "Design a pipeline that ingests 100M events/day from Kafka into a warehouse with under-1-hour freshness and idempotent backfills." Not "design a global multi-region exabyte lakehouse." See [15-system-design-de.md](./15-system-design-de.md).

The honest truth: most rejections at your level are from weak SQL, hand-wavy answers about idempotency/exactly-once, or inability to walk through a real pipeline you've built end-to-end. We'll fix all three.

---

## The Topic Index

Read in this order. Each file follows the Beginner → Intermediate → Advanced format and ends with an Interview-Ready Cheat Sheet.

### Tier 1 — Must Know Cold (interviewer assumes you have these)

1. [SQL Deep Dive](./01-sql-deep-dive.md) — window functions, CTEs, query plans, optimization
2. [Data Modeling](./02-data-modeling.md) — normalization, star/snowflake, SCDs, OBT
3. [Warehouses, Lakes, Lakehouses](./03-warehouses-lakes-lakehouse.md) — OLTP vs OLAP, columnar storage, when to use what
4. [ETL vs ELT & Pipeline Patterns](./04-etl-vs-elt.md) — idempotency, incremental loads, backfills

### Tier 2 — Expected at Mid-Level

5. [Orchestration & Airflow](./05-orchestration-airflow.md) — DAGs, sensors, alternatives
6. [Batch vs Streaming](./06-batch-vs-streaming.md) — Kafka, Kinesis, Flink, exactly-once, watermarks
7. [Spark Fundamentals](./07-spark-fundamentals.md) — RDD/DF, lazy eval, partitioning, shuffles, AQE
8. [Open Table Formats](./08-table-formats.md) — Iceberg, Delta Lake, Hudi
9. [Change Data Capture](./09-cdc.md) — log-based vs trigger-based, Debezium
10. [dbt — Modular SQL Transforms](./10-dbt.md) — materializations, tests, lineage

### Tier 3 — Differentiators (these add resume weight)

11. [Data Quality & Contracts](./11-data-quality-contracts.md) — Great Expectations, dbt tests, schema evolution
12. [AWS Data Stack](./12-aws-data-stack.md) — Glue, EMR, Redshift, Kinesis, MSK, Athena, Lake Formation
13. [Partitioning & Performance](./13-partitioning-performance.md) — file sizes, bucketing, Z-ordering, compaction
14. [Governance, Lineage, Observability](./14-governance-lineage-observability.md) — OpenLineage, DataHub, Monte Carlo
15. [System Design for DE](./15-system-design-de.md) — interview frameworks + worked examples

---

## Certifications That Actually Move the Needle

Honest mentor breakdown — most certs are signaling, not skill. The ones below are worth the time because they're either *the* expected ticket for the role or genuine knowledge investments. Order matters.

### Tier A — Get this first (highest ROI for your level)

**[AWS Certified Data Engineer – Associate (DEA-C01)](https://aws.amazon.com/certification/certified-data-engineer-associate/)**
- $150, 130 min, 65 questions, 3-year validity
- The *only* AWS cert designed specifically for DE. Covers Glue, Redshift, Kinesis, S3, Lake Formation, Athena — exactly the services in 80% of AWS DE postings
- Roughly 40–80 hours of prep depending on your hands-on experience
- Best cost/value cert in the entire DE landscape
- This is your first cert. Period.

### Tier B — Pick one to differentiate

**[Databricks Certified Data Engineer Associate](https://www.databricks.com/learn/training/certification)**
- $200. Validates Databricks Lakehouse Platform skills (Spark, Delta Lake, Unity Catalog, DLT)
- Pick this if your target companies use Databricks (a lot do — finance, retail, healthcare)
- Pairs naturally with Spark deep knowledge from [07-spark-fundamentals.md](./07-spark-fundamentals.md)

**[SnowPro Core Certification](https://www.snowflake.com/certifications/)**
- $175. Snowflake architecture, virtual warehouses, data sharing, Time Travel, security
- Pick this if you target companies on Snowflake (modern data stack shops, many startups)
- The Advanced: Data Engineer is $375 more — only do it if Snowflake is your daily driver

### Tier C — Skip unless required

- AWS Certified Data Analytics – Specialty: being retired, replaced by DEA-C01
- AWS Solutions Architect Associate: useful breadth but not DE-specific; only if you want to pivot toward platform/architect roles
- Azure DP-203 / DP-700: only if interviewing at Microsoft-stack shops
- GCP Professional Data Engineer: solid cert but smaller job market than AWS

### Tier D — Don't bother

- "Bootcamp certificates" from Coursera/Udemy/etc. as resume credentials. Use them to learn, not to claim.
- Certs from vendors not present in your target market

**My recommendation for you:** AWS DEA-C01 first (3 months from today), then Databricks Associate or SnowPro Core depending on where the jobs you want are clustered. Two certs is the sweet spot — beyond that, hiring managers see padding.

---

## What's in Job Postings Right Now (April 2026)

Based on current scraping of mid-level AWS DE postings, the consistent must-have stack:

| Bucket | What postings actually want |
|---|---|
| **Languages** | Python (mandatory), SQL (mandatory), one of Scala/Java (nice to have) |
| **Processing** | Spark (PySpark) — mentioned in ~70% of postings |
| **Orchestration** | Airflow most common, MWAA on AWS, Step Functions for serverless |
| **Storage** | S3 + Parquet/Iceberg/Delta, Redshift or Snowflake for warehouse |
| **Streaming** | Kafka or Kinesis (often "exposure to" rather than expert) |
| **Transform** | dbt is becoming table-stakes for analytics-engineering-flavored roles |
| **CI/CD** | Git, GitHub Actions, Terraform for infra as code |
| **Observability** | One of: dbt tests, Great Expectations, Monte Carlo, custom freshness checks |

What employers say but really mean:
- "5+ years experience" — they'll usually take 2 YOE if you can demonstrate the work
- "Master's preferred" — almost never blocking if your projects/resume are strong
- "Experience with [obscure tool]" — they'll teach you if you nail fundamentals

---

## Resume / Project Strategy

The single biggest lever at your level isn't more certs — it's **one or two real, deep, end-to-end projects** that mimic production data engineering. See the [projects/](../projects/README.md) folder for high-value project ideas designed to be resume-worthy.

A good DE project on your resume answers:
- What real-world problem did it solve?
- What was the data volume / velocity / variety?
- What was the architecture diagram (not just "I used Airflow")?
- What trade-offs did you make and why?
- How did you handle failure modes — backfills, idempotency, schema changes?
- What did you measure (cost, latency, freshness, accuracy)?

If you can't answer those for a project on your resume, it's not pulling its weight.

---

## Cross-Disciplinary Links

DE and AI engineering overlap more every year. The vector / RAG / feature-store space is increasingly DE territory:

- [RAG Fundamentals](../ai/rag/01-rag-fundamentals.md) — embedding pipelines are DE pipelines with vector outputs
- [LLM Fundamentals](../ai/llms/01-llm-fundamentals.md) — useful context when you're asked about ML platform work
- [Prompting Fundamentals](../ai/prompting/01-prompting-fundamentals.md) — useful for any "we have an LLM in the pipeline" interview question

---

## How to Use This Hub

1. **Self-assess** — read each Cheat Sheet at the bottom of each file. Anything you can't explain confidently → that's where to focus.
2. **Code along** — every concept here should map to something you've actually run. Cloning a Spark job from a tutorial is fine; pretending you've used something you haven't is not.
3. **Build the projects** in [../projects/](../projects/README.md) — they are designed to hit the most-asked interview topics.
4. **Mock interviews** — once you've covered Tiers 1 and 2, schedule mock interviews. Real-time questioning exposes gaps tutorials don't.

---

## Sources

- [AWS Certified Data Engineer – Associate (DEA-C01) Exam Guide](https://aws.amazon.com/certification/certified-data-engineer-associate/)
- [Best Data Engineering Certifications in 2026 — Dataquest](https://www.dataquest.io/blog/best-data-engineering-certifications/)
- [Best Data Engineering Certifications by Career ROI — Careery](https://careery.pro/blog/data-engineer-careers/best-data-engineering-certifications)
- [I Analyzed 1,000+ Data Engineering Job Postings — Towards Data Engineering](https://medium.com/towards-data-engineering/i-analyzed-1-000-data-engineering-job-postings-heres-which-certifications-actually-matter-in-2026-544fb1594d79)
- [How to Become an AWS Data Engineer in 2026 — KnowledgeHut](https://www.knowledgehut.com/blog/cloud-computing/how-to-become-aws-data-engineer)
