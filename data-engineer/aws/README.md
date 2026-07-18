# AWS Certified Data Engineer — Associate (DEA-C01) — Study Hub

**Target exam:** AWS Certified Data Engineer – Associate, Exam Code **DEA-C01**
**Format:** 65 questions, 130 minutes, multiple choice + multiple response
**Pass mark:** 720 / 1000 (scaled)
**Cost:** $150 USD
**Validity:** 3 years
**Timeline in this repo:** 2 weeks focused prep → schedule exam → optional 3–4 week project post-exam
**Last updated:** July 4, 2026

---

## Why This Section Exists

The DEA-C01 is a services-heavy exam. You already have the *conceptual* DE foundation in the parent [`data-engineer/`](../) folder — SQL, modeling, streaming, table formats, orchestration, governance. This section is the **AWS-specific translation layer**: which service AWS wants you to pick for each pattern, the trap answers, and the operational details (partition sizes, IAM policies, KMS scope, VPC endpoints) the exam grills you on.

The style matches the rest of the repo: concept → why AWS picks it → interview trap → cheat sheet.

---

## Exam Structure and Domain Weights

| Domain | Weight | This repo's coverage |
|---|---|---|
| **1. Data Ingestion and Transformation** | ~34% | [01-domain1-ingestion-and-transformation.md](./01-domain1-ingestion-and-transformation.md) |
| **2. Data Store Management** | ~26% | [02-domain2-data-store-management.md](./02-domain2-data-store-management.md) |
| **3. Data Operations and Support** | ~22% | [03-domain3-operations-and-support.md](./03-domain3-operations-and-support.md) |
| **4. Data Security and Governance** | ~18% | [04-domain4-security-and-governance.md](./04-domain4-security-and-governance.md) |

Focus your time proportionally. Domain 1 alone is a third of the exam.

Supplements:
- [05-service-quick-reference.md](./05-service-quick-reference.md) — one-page cheat over every in-scope service.
- [06-exam-day-strategy.md](./06-exam-day-strategy.md) — how to read AWS multi-answer questions + trap phrasings.
- [07-project-llm-intelligence-platform.md](./07-project-llm-intelligence-platform.md) — the capstone project spec (LLM Intelligence Platform — high-signal-only aggregation from arXiv + vendor docs + practitioner discourse).

Cross-references to conceptual coverage already in the repo:
- [ETL vs ELT](../04-etl-vs-elt.md) — idempotency, backfills, DLQs.
- [Batch vs Streaming](../06-batch-vs-streaming.md) — Kafka, Flink, watermarks.
- [Spark Fundamentals](../07-spark-fundamentals.md) — used in Glue and EMR.
- [Table Formats](../08-table-formats.md) — Iceberg (in-scope), Delta, Hudi.
- [CDC](../09-cdc.md) — DMS + Debezium.
- [Data Quality & Contracts](../11-data-quality-contracts.md) — Glue Data Quality, DataBrew.
- [AWS Data Stack overview](../12-aws-data-stack.md) — start here for the wider stack.
- [Governance / Lineage / Observability](../14-governance-lineage-observability.md) — Lake Formation, Macie, IAM.

---

## The 14-Day Study Plan

Assumes 2–3 focused hours/day. Weekend days can absorb a bit more.

### Week 1 — Domains 1 + 2 (60% of exam)

**Day 1 — Orientation + Domain 1 (Ingestion)**
- Read [12-aws-data-stack.md](../12-aws-data-stack.md) (repo overview).
- Read [01-domain1](./01-domain1-ingestion-and-transformation.md) §1 (Batch ingestion) + §2 (Streaming ingestion).
- Focus: S3 as landing zone, Kinesis Data Streams vs Firehose vs MSK, DMS full-load + CDC.
- Watch: [AWS Skill Builder — DEA-C01 free study path](https://explore.skillbuilder.aws/) — pick Domain 1 modules.

**Day 2 — Domain 1 continued (Transformation)**
- Read [01-domain1](./01-domain1-ingestion-and-transformation.md) §3 (Glue) + §4 (EMR).
- Focus: Glue ETL vs Glue Studio vs Glue Data Brew; EMR on EC2 vs EMR on EKS vs EMR Serverless.
- Hands-on: sketch a Glue job that reads S3 CSV → writes S3 Parquet with partitioning.

**Day 3 — Domain 1 continued (Orchestration + Programming)**
- Read [01-domain1](./01-domain1-ingestion-and-transformation.md) §5 (Orchestration) + §6 (Programming concepts).
- Focus: Step Functions vs MWAA vs Glue workflows; EventBridge patterns; Lambda concurrency; CDK/CloudFormation basics.
- Hands-on: build a Step Functions state machine in the console (free tier).

**Day 4 — Domain 2 (Data Store Management)**
- Read [02-domain2](./02-domain2-data-store-management.md) §1 (Storage picker) + §2 (Redshift deep-dive).
- Focus: Redshift DISTKEY / SORTKEY / RA3 vs Provisioned vs Serverless; Redshift Spectrum + federated queries.
- Cross-read: [Warehouses & Lakehouses](../03-warehouses-lakes-lakehouse.md).

**Day 5 — Domain 2 continued (DynamoDB, Iceberg, cataloging)**
- Read [02-domain2](./02-domain2-data-store-management.md) §3 (DynamoDB) + §4 (Iceberg + S3 Tables) + §5 (Glue Data Catalog).
- Focus: DynamoDB partition keys, GSI vs LSI, on-demand vs provisioned, DAX, streams; S3 Tables (new); Glue crawlers.
- Cross-read: [Table Formats](../08-table-formats.md).

### Week 2 — Domains 3 + 4 + review

**Day 6 — Domain 3 (Operations + Analysis)**
- Read [03-domain3](./03-domain3-operations-and-support.md) §1 (Athena) + §2 (QuickSight, DataBrew).
- Focus: Athena partition projection, CTAS, federated queries, workgroups; QuickSight SPICE.

**Day 7 — Domain 3 continued (Monitoring)**
- Read [03-domain3](./03-domain3-operations-and-support.md) §3 (CloudWatch, CloudTrail) + §4 (Data Quality).
- Focus: CloudWatch Logs Insights syntax basics; CloudTrail data events vs management events; Glue Data Quality DQDL.

**Day 8 — Domain 4 (Security — IAM + KMS)**
- Read [04-domain4](./04-domain4-security-and-governance.md) §1 (IAM) + §2 (Encryption + KMS).
- Focus: IAM policies vs resource policies vs SCPs; assume-role; KMS CMK vs AWS-managed; envelope encryption; S3 SSE-S3 / SSE-KMS / SSE-C.

**Day 9 — Domain 4 continued (Lake Formation + Macie + VPC)**
- Read [04-domain4](./04-domain4-security-and-governance.md) §3 (Lake Formation) + §4 (Macie + PII) + §5 (VPC + networking).
- Focus: Lake Formation LF-tags + row/column-level access; Macie PII discovery; VPC endpoints for S3 / Kinesis.
- Cross-read: [Governance](../14-governance-lineage-observability.md).

**Day 10 — Service quick reference + weak-area review**
- Read [05-service-quick-reference.md](./05-service-quick-reference.md) end to end. This is your "did I miss anything?" pass.
- Take a **free AWS Skill Builder practice test** (they publish one).
- Note which service names you can't confidently place — re-read those service sections.

**Day 11 — Practice test #1 (paid or free)**
- Take a full 65-question practice test in one 130-min sitting. Simulated conditions.
- Recommended: [Tutorials Dojo](https://tutorialsdojo.com/) practice tests (best-in-class), [Whizlabs](https://www.whizlabs.com/), or [AWS's official practice exam](https://explore.skillbuilder.aws/) via SkillBuilder.
- Score < 70% → identify the weakest domain and re-read that file the next morning.

**Day 12 — Weak-area drilling**
- Re-read the file(s) you scored lowest on.
- Read [06-exam-day-strategy.md](./06-exam-day-strategy.md).
- Read the [AWS DEA-C01 sample questions](https://d1.awsstatic.com/training-and-certification/docs-data-engineer-associate/AWS-Certified-Data-Engineer-Associate_Sample-Questions.pdf) (10 official ones — do these under time pressure).

**Day 13 — Practice test #2**
- Second full timed test. Aim for ≥ 75%.
- Review every wrong answer — understand *why* the right answer is right, not just what it is.

**Day 14 — Light review, sleep**
- Re-skim [05-service-quick-reference.md](./05-service-quick-reference.md) and [06-exam-day-strategy.md](./06-exam-day-strategy.md).
- No new material.
- Sleep 8+ hours. Exam next morning.

---

## Project (Post-Exam) — GenAI Support Intelligence Platform

Full spec in [07-project-genai-support-intelligence.md](./07-project-genai-support-intelligence.md).

**One-sentence pitch:** a serverless GenAI data platform on AWS that ingests **only high-signal LLM content** — arXiv papers, official docs from Anthropic / OpenAI / Google / Meta / Mistral, HuggingFace model cards, curated practitioner discourse — filters out AI-slop with a Bedrock quality classifier, extracts model comparisons + best practices, and serves semantic search across the actually-useful state of LLM knowledge.

**Why this project:**
- **Real problem, no good solution today** — signal-to-noise in LLM discourse is broken; anyone building on LLMs wastes hours filtering it. The signal-tier design solves that.
- **Meta angle** — LLMs enriching data *about* LLMs. Recruiters at LLM builders (Anthropic, OpenAI, DeepMind, Mistral, Cohere, HuggingFace) recognize this shape as internal-tooling they already build.
- Hits **all four exam domains** with real depth (proves you understood exam scope, not just recognition).
- Uses **~20 in-scope services** (S3, Kinesis, DMS, Glue, Lambda, Step Functions, EventBridge, Iceberg, RDS pgvector, Redshift Serverless, Athena, IAM, KMS, Lake Formation, Macie, Bedrock, CloudWatch, CDK, etc.).
- **Cost-capped hard** — $30 total build spend, $5/mo idle, $8 any 24-hour period. Enforced via Budget alarm.
- **Interview-defensible by design** — every service choice in the spec has a rationale + rejected alternatives.
- **Parallel with study** — each weekend implements the domain you just studied.

**Timeline:** 4 weekends alongside your 14-day study plan. Full breakdown, per-source ingestion patterns, interview drills at end of each weekend, and design-decision defenses in the project file.

---

## What to Buy / Sign Up For

| Item | Cost | Verdict |
|---|---|---|
| AWS Skill Builder Standard (free) | Free | Take the DEA-C01 study path — the video content is excellent. |
| Tutorials Dojo practice tests | ~$15 | Best-in-class. Buy this. |
| Stephane Maarek's Udemy DEA-C01 course | ~$15–20 on sale | Optional but highly recommended if you like video. |
| AWS official sample questions PDF | Free | Do these last, under time pressure. |
| Real AWS account | Free tier for 12 months | Sign up if you don't have one — needed for the project. |
| Exam voucher | $150 | Schedule via Pearson VUE, online or in-person. Look for 50% retake vouchers before booking. |

Don't buy every course. Time in the AWS console (via free tier) beats 5 video courses.

---

## Concept-Check Questions Before You Schedule the Exam

You should be able to answer all of these in one sentence without hesitation:

1. When do you pick Kinesis Data Streams vs Kinesis Data Firehose?
2. When do you pick MSK over Kinesis?
3. Glue vs EMR — trade-off in one line?
4. Redshift DISTKEY vs SORTKEY — what each optimizes?
5. Redshift Spectrum — what problem does it solve?
6. DynamoDB partition key design — how do you avoid hot partitions?
7. GSI vs LSI in DynamoDB — key differences?
8. Iceberg on AWS — what did S3 Tables add?
9. Step Functions Standard vs Express — when each?
10. MWAA vs Step Functions — when each?
11. Lake Formation LF-tags — what problem do they solve?
12. Macie — what does it discover and where?
13. S3 encryption: SSE-S3 vs SSE-KMS vs SSE-C — which do you pick for a compliance-regulated dataset?
14. VPC endpoints — when do you need one?
15. Athena partition projection — vs. Glue crawler-based partitions?

If any is fuzzy, re-read the relevant file.

---

## Resources (bookmark)

**Official AWS**
- [DEA-C01 Exam Guide (PDF)](https://docs.aws.amazon.com/pdfs/aws-certification/latest/data-engineer-associate-01/data-engineer-associate-01.pdf) — the source of truth.
- [DEA-C01 Sample Questions](https://d1.awsstatic.com/training-and-certification/docs-data-engineer-associate/AWS-Certified-Data-Engineer-Associate_Sample-Questions.pdf) — 10 questions.
- [AWS Skill Builder — DEA-C01 study path](https://explore.skillbuilder.aws/learn/public/learning_plan/view/2196/data-engineer-learning-plan).
- [Well-Architected — Data Analytics Lens](https://docs.aws.amazon.com/wellarchitected/latest/analytics-lens/analytics-lens.html).

**Community & practice**
- [Tutorials Dojo — DEA-C01](https://tutorialsdojo.com/aws-certified-data-engineer-associate-dea-c01/) — practice tests + study notes.
- [Stephane Maarek Udemy course](https://www.udemy.com/course/aws-certified-data-engineer-associate-dea-c01/).
- [r/AWSCertifications](https://www.reddit.com/r/AWSCertifications/) — recent trip reports.
- [ExamPro DEA-C01](https://www.exampro.co/aws-data-engineer-associate) — free video content.

**Deep-dive references (when a topic doesn't click)**
- [AWS Big Data Blog](https://aws.amazon.com/blogs/big-data/) — long-form real architectures.
- [AWS Solutions Library — Data & Analytics](https://aws.amazon.com/solutions/).

---

*Start with [01-domain1-ingestion-and-transformation.md](./01-domain1-ingestion-and-transformation.md). It's the biggest domain and everything else builds on it.*
