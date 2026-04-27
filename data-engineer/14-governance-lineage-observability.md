# Governance, Lineage, and Observability — The Layer Above the Pipeline

**Phase:** 1 (Foundations)
**Difficulty progression:** Beginner → Intermediate → Advanced
**Last updated:** April 24, 2026
**Related:** [Data Quality & Contracts](./11-data-quality-contracts.md) · [dbt — Modular SQL Transforms](./10-dbt.md) · [AWS Data Stack](./12-aws-data-stack.md) · [Orchestration & Airflow](./05-orchestration-airflow.md)

---

## Why This File Exists

"Pipelines that work" is a junior bar. "Pipelines you can trust, audit, recover, and explain" is the mid-level bar. Governance, lineage, and observability are how you cross from the first to the second. They're also increasingly highlighted in DE postings — "experience with data observability tools" and "data governance practices" appear in roughly 40% of mid-level postings now.

---

## BEGINNER — Defining the Three Layers

### Governance, Lineage, Observability — What's the Difference?

| | Definition | Sample question it answers |
|---|---|---|
| **Governance** | Policies and controls for *who can do what with which data* | "Can a marketing analyst see customer emails?" |
| **Lineage** | Map of *where data flows* through systems | "If we drop this column, what dashboards break?" |
| **Observability** | Continuous *measurement of pipeline and data health* | "Why is yesterday's report missing 30% of expected rows?" |

These overlap heavily — most modern tools blur the boundaries.

### Why Companies Care

Bad data → bad decisions. Bad data also → regulatory fines (GDPR, CCPA, HIPAA), failed audits, broken trust, eroded business value. The data team that can't say "this number is correct, here's where it came from, and here's how it's monitored" is a team without credibility.

This conversion is what's driven the rise of "data quality is a first-class engineering discipline."

### The Categories of Tooling

| Category | Sample tools |
|---|---|
| **Data catalogs** | DataHub, Atlan, Collibra, Alation, Amundsen, AWS Glue Catalog, Unity Catalog |
| **Lineage** | OpenLineage / Marquez, dbt docs, DataHub, Atlan |
| **Observability** | Monte Carlo, Bigeye, Anomalo, Soda, Sifflet |
| **Governance / access** | Lake Formation, Unity Catalog, Immuta, Privacera |
| **Quality testing** | dbt tests, Great Expectations, Soda |

Many of these overlap. DataHub does catalog + lineage + (some) quality. Monte Carlo does observability + (some) lineage. The market is consolidating; expect category boundaries to blur.

---

## INTERMEDIATE — Lineage and Observability in Practice

### What Lineage Looks Like

```
[source: app.orders] → stg_orders (dbt) → int_orders__joined → fact_orders → finance_dashboard
                                                                            → ML feature pipeline
                                                                            → Hightouch sync to Salesforce
```

Each arrow is a *dependency*. Lineage tracks these arrows so you can:
- Trace a number on a dashboard back to the raw source
- Know what breaks when you change a model
- Find owners of upstream sources when investigating issues
- Visualize the impact of dropping / renaming a column

**Two kinds of lineage:**

| Kind | Granularity |
|---|---|
| **Table-level** | "fact_orders depends on stg_orders, dim_users" |
| **Column-level** | "fact_orders.amount depends on stg_orders.total_cents and dim_users.country" |

Column-level lineage is the gold standard for impact analysis. dbt and DataHub support it with varying completeness.

### OpenLineage — The Standard

**OpenLineage** is an open standard for emitting lineage events from data tools. Spec includes:
- Run events (start, complete, abort)
- Job and dataset facets
- Schema, statistics, source code references

Tools that emit OpenLineage: Airflow (via plugin), dbt (via dbt-ol), Spark (via OpenLineage Spark integration), Flink, Great Expectations.

Tools that consume OpenLineage: Marquez (the reference backend), DataHub, Astronomer Astro, Manta.

Why it matters: previously, every lineage tool needed its own integration with every data tool — N×M problem. OpenLineage makes it 1×N + 1×M.

### Catalogs in Production

A data catalog is a searchable inventory of your data assets, with metadata: schemas, owners, descriptions, tags, lineage, freshness, downstream usage, quality scores.

| Tool | Sweet spot |
|---|---|
| **DataHub** (LinkedIn → OSS) | Engineering-led shops, OSS-first, broad integrations |
| **Atlan** | Modern UX, business + tech users, fast to deploy |
| **Collibra / Alation** | Enterprise governance, regulated industries |
| **Unity Catalog** (Databricks) | Native if you're on Databricks |
| **AWS Glue Catalog + Lake Formation** | Native AWS lakehouse setups |
| **Amundsen** (Lyft → OSS) | Engineering-led, simpler than DataHub |
| **OpenMetadata** | Newer OSS, fast-growing |

For an AWS-first stack, the realistic combo is **AWS Glue Catalog + Lake Formation + a higher-level catalog (DataHub / Atlan / Unity)**. Glue is the technical catalog; the higher-level tool is the human-facing one.

### Observability — Detecting Bad Data

Observability tools watch tables for:

| Signal | What it catches |
|---|---|
| **Volume** | Row count outside expected band |
| **Freshness** | Table not updated within SLO |
| **Schema** | New / dropped / renamed columns |
| **Distribution** | Null rate, mean, cardinality drift |
| **Lineage breakage** | Upstream changed in a way that breaks downstream |
| **Custom rules** | Reconciliation, business-logic checks |

The shift in the last few years: from "we wrote tests for everything" to "tools auto-baseline metrics and alert on anomalies." Better coverage, less hand-curation.

### Custom (Build) vs Tool (Buy) for Observability

| You should build | You should buy |
|---|---|
| <10 critical pipelines | 30+ pipelines, business-critical |
| Strong DE team with capacity | Limited DE capacity |
| dbt-test heavy + Slack alerts cover 80% | Need automatic anomaly detection |
| Cost-sensitive | Have engineering budget |

A common middle path: build dbt tests + custom Slack alerts for the basics; bring in a tool (Monte Carlo, Bigeye, Soda) when human tuning can't keep up.

### Setting SLOs for Data

Recap from [11-data-quality-contracts.md](./11-data-quality-contracts.md):

```yaml
slos:
  - name: fact_orders_freshness
    description: fact_orders updated within 30 min of source change
    target: 99.5%  # measured monthly
    burn_rate_alert_threshold: 5x  # alert when SLO is being burned 5x faster than target

  - name: dim_customers_completeness
    description: row count within ±5% of 7-day trailing average
    target: 99%
```

Then alert on burn rate, not single missed measurements. This pattern (borrowed from SRE) prevents alert fatigue.

---

## ADVANCED — Governance, PII, and Compliance

### PII Handling — The Pattern Every DE Should Know

**PII (Personally Identifiable Information):** name, email, address, phone, SSN, credit card, IP, government ID, biometric data, etc.

Modern playbook:

```
Source                Transit                      Storage                 Query
  │                     │                             │                      │
  ├ Encrypt at source   ├ TLS 1.2+                    ├ KMS at rest          ├ Column masking
  │                     │                             │                      ├ Row-level filters
  │                     │                             ├ Tokenization for     ├ Audit log
  │                     │                             │  fields not needed   │
  │                     │                             │  in clear            │
  │                     │                             ├ PII columns tagged   │
  │                     │                             │  in catalog          │
```

| Technique | When |
|---|---|
| **Encryption** at rest and in transit | Default for everything |
| **Tokenization** (replace value with stable token) | When downstream doesn't need the actual value but needs uniqueness/joins |
| **Hashing** (one-way) | When you only need equality joins / not the original value |
| **Pseudonymization** | Replace direct identifiers with stable pseudonyms |
| **Anonymization** | True anonymization (k-anonymity, differential privacy); rarely fully achievable |
| **Column masking** | Hide value at query time based on the user's role (Lake Formation, Unity Catalog) |
| **Row-level security** | Filter rows based on user identity (e.g., tenant isolation) |

**The trap:** "We anonymized it by hashing" — hashing alone is reversible if values are predictable (you can rainbow-table common emails). Real anonymization requires multiple techniques + threat modeling.

### Compliance Regimes Worth Knowing

| Regime | Region/scope | DE-relevant points |
|---|---|---|
| **GDPR** | EU residents | Right to access, erasure, portability; lawful basis for processing |
| **CCPA / CPRA** | California | Right to know, delete, opt-out |
| **HIPAA** | US healthcare | Protected Health Information (PHI); BAAs; encryption requirements |
| **PCI-DSS** | Anyone touching card data | Tokenization mandatory; segmented network for cardholder data |
| **SOX** | US public companies | Financial data integrity, audit trails |
| **SOC 2** | SaaS vendors | Trust principles: security, availability, confidentiality, etc. |

You don't need expertise — you need to know:
- These exist and apply differently
- "Right to erasure" requires you to delete a person's data on request, including from analytics — that means delete-friendly table formats
- Data residency requirements affect where you store and process

### Data Deletion ("Forget Me") in Practice

GDPR / CCPA users can ask you to delete their data. In a lakehouse context:

| Approach | Pros / Cons |
|---|---|
| Hard delete in Iceberg/Delta/Hudi (`DELETE FROM ...`) | Pros: actually deletes. Cons: rewrites whole files; expensive at scale |
| Soft delete (mark `deleted_at`) | Pros: cheap. Cons: not actually deleted; auditors may not accept |
| Tokenize at ingest, delete tokens later | Pros: reversible isolation. Cons: adds complexity |
| Periodic partition rewrites | Pros: amortizes cost. Cons: deletion latency |

Industry default: lakehouse formats with a "deletion request queue" processed in scheduled compaction passes. Make sure you can demonstrate compliance within the regime's required time window (often 30 days).

### Access Control Models

| Model | Where you'll see it |
|---|---|
| **Role-Based Access Control (RBAC)** | Default for warehouses; user has roles, roles have grants |
| **Attribute-Based Access Control (ABAC)** | Lake Formation LF-tags, Unity Catalog tags |
| **Row-level security** | Filter rows by user identity (`tenant_id = current_user_tenant()`) |
| **Column-level security** | Mask PII columns from non-privileged users |
| **Cell-level security** | Mask specific cells based on combined criteria |

Snowflake / BigQuery / Redshift all support row + column policies natively. Lake Formation does it for the lake; Unity Catalog for Databricks.

### Cataloging the Data You Have

Practical pattern:

1. **Auto-discover** technical metadata via crawlers / connectors (Glue Crawlers, DataHub ingestion)
2. **Tag** sensitive columns automatically (regex / classifiers / Macie)
3. **Enforce** access policies via tags (LF-tag based access in Lake Formation)
4. **Annotate** with human metadata (descriptions, owners, business definitions)
5. **Connect** lineage from operational sources through to dashboards
6. **Surface** in a catalog UI everyone can search

Without this, your "data lake" is a "data swamp" — and the catalog is what differentiates them.

### AWS Macie — PII Discovery

AWS service that scans S3 for sensitive data (credit card numbers, SSNs, etc.) using ML and pattern matching.

| Use | When |
|---|---|
| Periodic scan of new data | Inventory PII you didn't realize was there |
| Continuous monitoring | Block accidental PII landings |
| Compliance audits | Evidence of due diligence |

Pair with Lake Formation tags + Glue Catalog for end-to-end PII tracking.

### Cost Observability

Often forgotten — but for DE, cost *is* a quality dimension.

| Tool | What |
|---|---|
| **AWS Cost Explorer + Budgets** | Per-service / per-tag breakdowns; alerts |
| **Snowflake Account Usage views** | Per-warehouse / per-query / per-user costs |
| **BigQuery information_schema.JOBS** | Per-query bytes processed |
| **dbt's `--profile` flag** | Time per model |
| **Custom dashboards** | Per-pipeline cost trend lines |

Treat unexpected cost spikes like data anomalies — they signal something changed (forgotten test, runaway query, blown-up cardinality).

### Audit Trails

| Source | What you get |
|---|---|
| **CloudTrail** | API calls — who created/modified/deleted AWS resources |
| **Warehouse query logs** | Snowflake QUERY_HISTORY, BigQuery audit, Redshift STL views |
| **Catalog audit logs** | Who looked up what; who granted access |
| **Application audit** | Outbox events for application-emitted operations |

For regulated environments, audit trails must be retained per the regime (GDPR: typically months; SOX: 7 years; HIPAA: 6 years).

### Disaster Recovery Patterns

Often grouped with governance because both are about "what happens when something goes wrong":

| Concern | Pattern |
|---|---|
| Lost a partition | Restore from S3 versioning / table format time travel |
| Lost the warehouse | Restore from latest snapshot; replay since-then changes from raw |
| Lost the catalog | Re-crawl from source-of-truth metadata; backups of Glue Catalog |
| Lost the orchestrator | DAG code in git; deploy fresh Airflow; replay from data source |
| Lost the source DB | Restore from PITR; re-run CDC backfill |

A DR plan is not "we have backups." It's "we have run a recovery drill and timed it."

### The Cultural Side — Ownership

Tools fail without ownership. Every dataset should have:

- An **owner** (team or person) responsible for its quality
- A **steward** (person who answers questions about it)
- A **consumer-facing description** (what it is, when not to use it)
- A **deprecation policy** (when does it go away?)

Catalogs make this explicit. Without it, every dataset is "ask in #data-help."

### When You'll Be Asked About This

Mid-level interviews tend to probe governance with questions like:
- "How would you ensure analysts can use customer data without seeing PII?"
- "How would you trace a number on a dashboard back to its source?"
- "What happens when a producer service drops a column?"
- "How would you handle a GDPR delete request?"
- "What does your team's on-call rotation look like for data incidents?"

These aren't trick questions; they're the everyday concerns of mid-level DE work. Have answers ready.

---

## Worked Example — Governance Architecture for an E-commerce DE Team

**Scenario:** You're hired as the second DE at a Series B e-commerce company. Lake on S3, Iceberg tables, dbt running on Athena, MWAA for orchestration. You're asked to set up governance and observability.

**My plan:**

**Catalog & Lineage**
- Glue Catalog as the technical catalog
- DataHub deployed for human-facing search + lineage
- OpenLineage emitting from Airflow + dbt + Spark; Marquez or DataHub as the backend
- dbt docs for analytics-engineering-facing lineage
- All datasets get owner + description in dbt YAML

**Quality & Observability**
- dbt tests on every model in marts (unique, not_null, relationships, business rules)
- dbt source freshness on bronze tables
- Volume + reconciliation checks daily, alerting to Slack
- Bring in Monte Carlo or Anomalo Q3 for automatic distribution / freshness anomaly detection across all silver/gold tables
- SLOs on top 5 business-critical tables; burn-rate alerts in PagerDuty
- Cost dashboards per pipeline

**Governance**
- Lake Formation with LF-tags: `pii=email`, `pii=phone`, `tier=public/internal/restricted`
- Column masking policies for analyst role
- Macie scanning new bronze tables for PII drift
- Audit logs to a separate compliance bucket; CloudTrail centralized

**Compliance**
- GDPR delete process: append delete request to a queue table; nightly compaction job rewrites affected partitions
- 30-day SLA for completion of erasure requests
- Annual DR drill: restore a critical table from time travel + replay; document time-to-recover

**Process**
- Every dataset has an owner (dbt YAML + DataHub)
- Every breaking schema change requires a deprecation period (announce in #data-changes; alert on usage; remove after grace)
- Postmortem template for data incidents

This is the kind of "you've actually thought about this" answer that distinguishes mid-level candidates from juniors who say "we'd run dbt tests."

---

## Interview-Ready Cheat Sheet

**"What is data lineage?"** Map of where data flows — producer → table → derived table → dashboard. Lets you trace numbers back to source and assess impact of changes. Table-level is common; column-level is the gold standard.

**"How would you set up data observability?"** Layered: dbt tests in transforms (hard checks), volume/freshness/distribution monitors (anomaly), reconciliation against source of truth (correctness), SLOs with burn-rate alerts (paging discipline). Tooling: Monte Carlo / Bigeye / Soda / custom.

**"How do you handle PII in a data lake?"** Encrypt at rest (KMS) + in transit (TLS), tag PII columns in catalog, enforce column masking via Lake Formation / Unity Catalog, audit access via CloudTrail, scan with Macie, tokenize where possible.

**"How would you handle a GDPR delete request?"** Lakehouse table format (Iceberg / Delta / Hudi) with `DELETE FROM ... WHERE user_id = ?`; periodic compaction to fully rewrite affected partitions; ensure deletion propagates to backups within retention policy.

**"What's OpenLineage?"** Open standard for emitting lineage events from data tools (Airflow, dbt, Spark, Flink). Tools emit; backends (Marquez, DataHub) consume. Solves the N×M integration problem.

**"What's a data catalog?"** Searchable inventory of data assets with metadata: schema, owner, description, tags, lineage, freshness, downstream usage, quality. The interface for "what data do we have, who owns it, can I trust it?"

**Quick trade-off pairs:**
- Build vs buy observability: control vs time-to-value
- Hard delete vs soft delete: compliance vs cost
- RBAC vs ABAC: simple vs flexible/scalable
- Manual catalog vs auto-discovery: curation quality vs coverage

---

## Resources & Links

- [OpenLineage spec](https://openlineage.io/) — the standard
- [Marquez](https://marquezproject.ai/) — reference OpenLineage backend
- [DataHub docs](https://datahubproject.io/) — open-source catalog
- [AWS Lake Formation](https://docs.aws.amazon.com/lake-formation/)
- [Databricks Unity Catalog](https://docs.databricks.com/data-governance/unity-catalog/index.html)
- [Monte Carlo blog — Observability fundamentals](https://www.montecarlodata.com/blog/)
- [Locally Optimistic — community blog on data quality / governance](https://locallyoptimistic.com/)

*Next: [System Design for DE](./15-system-design-de.md) — putting all of this together for the interview.*
