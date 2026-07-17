# DEA-C01 Domain 3 — Data Operations and Support (22%)

**Weight:** ~22% of the exam.
**Prereqs:** [Data Quality & Contracts](../11-data-quality-contracts.md) · [Governance / Lineage / Observability](../14-governance-lineage-observability.md)
**Related tasks:** T3.1 Automate · T3.2 Analyze · T3.3 Maintain & monitor · T3.4 Data quality

---

## §1 — Amazon Athena (Task 3.2)

Serverless Presto/Trino over S3. Pay per TB scanned.

### Core patterns

- **Query S3 in place** — no data loading. Read Parquet / ORC / CSV / JSON / Iceberg.
- **Uses Glue Data Catalog** for table metadata.
- **Athena workgroups** — cost / access control per team. Set per-query and per-workgroup data-scanned limits.

### Cost-reduction techniques (recurring exam theme)

- **Columnar formats** (Parquet / ORC) — read only the columns you need. 5–10× cheaper.
- **Compression** (Snappy for Parquet) — reduces bytes scanned.
- **Partitioning** — filter by partition key → skip data. `PARTITIONED BY (year, month, day)`.
- **Predicate pushdown** — WHERE filters applied at scan time.
- **Compaction** — merge small files into ~128MB–1GB Parquet files (small files kill perf).
- **`SELECT`-only-needed-columns** — avoid `SELECT *`.

### Athena partition projection

Instead of adding partitions to the Glue Catalog (via crawlers or `MSCK REPAIR TABLE`), tell Athena the partition scheme up front:

```sql
CREATE EXTERNAL TABLE mydb.logs (...)
PARTITIONED BY (dt string)
LOCATION 's3://bucket/logs/'
TBLPROPERTIES (
  'projection.enabled' = 'true',
  'projection.dt.type' = 'date',
  'projection.dt.range' = '2020-01-01,NOW',
  'projection.dt.format' = 'yyyy-MM-dd',
  'storage.location.template' = 's3://bucket/logs/${dt}/'
);
```

Athena computes partitions on the fly from the projection — no crawler needed. **Massive cost + latency win for time-series data.** Common exam answer.

### CTAS (CREATE TABLE AS SELECT)

```sql
CREATE TABLE new_table
WITH (
  format = 'PARQUET',
  external_location = 's3://.../new_table/',
  partitioned_by = ARRAY['day']
) AS
SELECT ... FROM raw_table;
```

Common uses: reformat CSV → Parquet, materialize aggregations, create partitioned copy.

### Athena federated queries

Query non-S3 data sources (RDS, DynamoDB, DocumentDB, CloudWatch Logs, on-prem) via Lambda-backed **data source connectors**.

### Athena for Apache Spark (Athena notebooks)

Interactive PySpark notebooks in Athena, using Athena as the compute engine. Skill 3.2.4 explicitly mentions this. Good for exploratory Spark without spinning up EMR.

### Athena vs Redshift Spectrum

Both query S3 with SQL:
- **Athena** — fully serverless, no cluster. Best for ad-hoc, occasional.
- **Redshift Spectrum** — requires a Redshift cluster. Best when combining S3 data with existing Redshift tables via JOIN.

---

## §2 — QuickSight, DataBrew, SageMaker Data Wrangler

### Amazon QuickSight

BI tool. Two "editions":
- **Standard** — basic dashboards.
- **Enterprise** — SPICE, ML insights, embedded analytics, row-level security, VPC connectivity.

**SPICE** (Super-fast, Parallel, In-memory Calculation Engine) — QuickSight's in-memory columnar cache. Loads data into SPICE for fast dashboard rendering. Refresh schedule per dataset.

**Row-level security** — filter dashboard data per user via a permissions dataset.

**Q (QuickSight Q)** — natural language querying ("show revenue by region last quarter").

### AWS Glue DataBrew

Visual data prep, 250+ prebuilt transformations. Point at S3 / Redshift / RDS → clean / normalize / dedup / mask → write output.

- **Recipes** — versioned transformation pipelines.
- **Projects** — workspace with a sample dataset.
- **Jobs** — recipe applied to full dataset.
- **PII detection** built-in — flags/masks PII columns.

### Amazon SageMaker Data Wrangler

Similar to DataBrew but in the ML context. Feature engineering for SageMaker training data.

Skill 3.2.2 mentions this — for data verification and cleaning in ML pipelines.

---

## §3 — Monitoring: CloudWatch, CloudTrail, X-Ray (Task 3.3)

### Amazon CloudWatch

Metrics + logs + alarms + dashboards + events (moved to EventBridge). Three sub-services:

**CloudWatch Metrics** — time-series metrics with dimensions.
- **Standard resolution:** 1 min granularity, up to 15 months retention.
- **High resolution:** 1 sec, up to 3 hours at 1s then aggregated.
- **Custom metrics** via `PutMetricData` API — per-metric cost.

**CloudWatch Logs** — log ingestion, storage, query.
- **Log groups** — logical container (e.g., one per Lambda function).
- **Log streams** — one per source instance.
- **Retention** — configurable per log group (1 day to 10 years, or never expire).
- **Metric filters** — extract metrics from log patterns.
- **Subscription filters** — stream logs to Kinesis / Firehose / Lambda for real-time processing.

**CloudWatch Logs Insights** — SQL-lite query language for logs:

```
fields @timestamp, @message
| filter @message like /ERROR/
| stats count() by bin(5m)
| sort @timestamp desc
| limit 20
```

**CloudWatch Alarms** — trigger on metric thresholds → SNS notification, Auto Scaling, EC2 action.

**CloudWatch dashboards** — combine metrics + logs into one view.

### AWS CloudTrail

Audit log of every AWS API call.

- **Management events** — control-plane API calls (default logged).
- **Data events** — data-plane operations (S3 GET/PUT, Lambda invocations). Costs extra, not on by default.
- **Insights events** — anomaly detection.

**CloudTrail Lake** — managed data lake for CloudTrail events. Query with SQL for centralized cross-account audit.

**Common trap:** to log S3 object-level access, you must enable **CloudTrail data events** — management events don't cover S3 GET/PUT.

### AWS Config

Continuous compliance monitoring. Records resource configuration changes over time. Rules evaluate compliance ("all S3 buckets encrypted?"). Auto-remediation via SSM Automation.

Skill 4.5.4 explicitly asks about "viewing configuration changes" — Config is the answer.

### Amazon Managed Grafana

Managed Grafana dashboards. Data sources: CloudWatch, Prometheus, OpenSearch, X-Ray. Skill 3.3 mentions Grafana for observability.

### AWS X-Ray (light)

Distributed tracing. Out of scope for the exam (per official guide) but might appear in passing.

---

## §4 — Data Quality (Task 3.4)

### AWS Glue Data Quality

Managed data quality checks over Glue tables and DynamicFrames.

- **DQDL (Data Quality Definition Language)** — declarative rules:

```
Rules = [
  IsComplete "email",
  Uniqueness "user_id" > 0.99,
  ColumnValues "status" in ["ACTIVE","INACTIVE"],
  ColumnLength "phone" between 10 and 15,
  RowCount between 1000 and 10000000
]
```

- **Rule recommendations** — Glue can suggest rules by profiling the data.
- **Results** — pushed to CloudWatch, S3, EventBridge for alerting.
- **Enforcement** — fail the Glue job on rule violation (data contract behavior).

Cross-read: [Data Quality & Contracts](../11-data-quality-contracts.md).

### DataBrew for quality

Also has data quality rules and profile jobs (statistics per column). Skill 3.4.2 and 3.4.3 mention DataBrew.

### Data sampling techniques (Skill 3.4.4)

Basic concepts to recognize:
- **Random sampling** — uniform probability. `TABLESAMPLE BERNOULLI(1)`.
- **Systematic sampling** — every nth row.
- **Stratified sampling** — proportional across categories. Preserves distribution.
- **Reservoir sampling** — fixed-size sample from a stream of unknown length.

### Data skew handling (Skill 3.4.5)

Cross-read: [Spark Fundamentals — Skew](../07-spark-fundamentals.md).

Standard techniques:
- **Salting** — append random suffix to hot keys, spread across partitions.
- **Broadcast join** — replicate small side to every executor to avoid shuffle.
- **AQE (Adaptive Query Execution)** — Spark auto-detects and splits skewed partitions.
- **Custom partitioner** — override default hash partitioning.

---

## §5 — Automation Patterns (Task 3.1)

### Common serverless data-processing flows

Recognize these blueprints:

**Flow 1: Event-driven S3 processing**
```
S3 PUT → EventBridge → Lambda → Glue Job → S3 output → EventBridge → next Lambda
```

**Flow 2: Scheduled ETL**
```
EventBridge Scheduler (cron) → Step Functions → Glue crawler → Glue job → Athena CTAS
```

**Flow 3: Streaming to warehouse**
```
Kinesis Data Stream → Firehose (JSON→Parquet) → S3 → EventBridge → Glue crawler → Athena/Redshift Spectrum
```

**Flow 4: Streaming into Redshift directly**
```
Kinesis Data Stream → Redshift streaming ingestion (materialized view) → dashboards
```

**Flow 5: CDC into lakehouse**
```
RDS → DMS → Kinesis → Firehose → S3 → Iceberg via Glue → Athena / EMR
```

**Flow 6: Real-time LLM enrichment**
```
Kinesis → Lambda → Bedrock InvokeModel → DynamoDB (state) + Firehose (audit) → S3
```

---

## Common Exam Traps

**Q: "Query terabytes of S3 CSV once a week for reporting"**
A: Convert to Parquet + partition, then Athena. Not Redshift (persistent cluster wasteful for weekly).

**Q: "Real-time dashboard on streaming data"**
A: Redshift streaming ingestion → QuickSight. Or Firehose → OpenSearch → QuickSight.

**Q: "Log all S3 GET requests"**
A: CloudTrail **data events** on the S3 bucket.

**Q: "Detect when EC2 CPU > 80% for 5 minutes and send email"**
A: CloudWatch Alarm → SNS topic → email subscriber.

**Q: "Track compliance drift on S3 encryption settings"**
A: AWS Config rule.

**Q: "Zero-code data profile + PII detection on a S3 dataset"**
A: DataBrew profile job.

**Q: "Ensure email column is 99% populated in Glue table"**
A: Glue Data Quality DQDL `IsComplete "email" > 0.99`.

**Q: "Add newly-created S3 partitions to a table daily, no crawler cost"**
A: Athena partition projection.

**Q: "Query streaming data with SQL, sub-minute latency"**
A: Redshift streaming ingestion. Or Managed Service for Apache Flink for windowed aggregations.

**Q: "Interactive Spark notebooks without EMR"**
A: Athena notebooks for Spark.

---

## Concept-Check (self-quiz)

1. Athena partition projection vs Glue crawler — cost/latency trade-off?
2. Athena workgroup — what does it control?
3. CTAS in Athena — one use case.
4. Redshift Spectrum vs Athena — pick one for querying S3 alongside Redshift dims.
5. QuickSight SPICE — what problem does it solve?
6. CloudWatch Logs Insights — write a filter for lines containing "TIMEOUT".
7. CloudTrail management events vs data events — one difference.
8. AWS Config — what does it monitor?
9. Glue Data Quality — what language defines rules?
10. Data skew fix — three techniques.
11. Real-time Redshift dashboards — which ingestion mode?
12. Stratified sampling — when do you use it?
13. Reservoir sampling — when do you use it?
14. Athena notebook Spark — when over EMR notebook?
15. DataBrew profile job — what does it produce?

---

## Resources

- [Athena Best Practices](https://docs.aws.amazon.com/athena/latest/ug/performance-tuning.html)
- [Athena Partition Projection](https://docs.aws.amazon.com/athena/latest/ug/partition-projection.html)
- [Glue Data Quality (DQDL)](https://docs.aws.amazon.com/glue/latest/dg/glue-data-quality.html)
- [CloudWatch Logs Insights syntax](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax.html)
- [CloudTrail Lake](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-lake.html)

---

*Next: [Domain 4 — Data Security and Governance](./04-domain4-security-and-governance.md).*
