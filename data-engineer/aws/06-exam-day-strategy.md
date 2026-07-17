# DEA-C01 — Exam Day Strategy and Common Trap Patterns

**Purpose:** the meta-skills that turn 700 into 800+ on the DEA-C01.

---

## Exam Format Recap

- **65 questions**, **130 minutes** → ~2 minutes per question.
- **Multiple choice** (single answer) + **multiple response** (2+ correct).
- **Pass mark:** 720 / 1000 (scaled).
- **Unscored questions:** ~15 are unscored trial questions AWS uses for future exams. You won't know which. Answer everything.
- **Language:** English default. Request +30 min if English is not your first language.
- **Delivery:** Pearson VUE testing center or online-proctored.
- **Result:** provisional pass/fail immediately at end of exam; official within 5 business days.

---

## The 4-Pass Approach for the Actual Exam

Managing time is the biggest predictor of success.

### Pass 1 — Fast Sweep (~50 minutes for the whole exam)

- Answer every question you can decide on in **< 60 seconds**.
- **Flag anything harder.** Don't get stuck. Don't spend 5 minutes on question 12.
- Target: end of pass 1 with ~20–25 flagged, ~40–45 answered.

### Pass 2 — Flagged Questions (~60 minutes)

- Come back to flagged questions with a fresh eye. Often the answer is clearer.
- **Now you can spend 2–3 minutes each** on the harder ones.
- Do the eliminate-wrong-answers approach religiously.

### Pass 3 — Second-Guess Check (~15 minutes)

- Scan every answer once more. Look specifically for questions where you know you rushed.
- **Don't change answers without a strong reason.** Second-guessing often makes things worse.

### Pass 4 — Buffer (~5 minutes)

- Bathroom / hydration / breathing.
- Submit with 30+ seconds still on the clock (avoid last-second panic).

---

## Reading AWS Multiple-Choice Questions — the Anatomy

Every AWS question has three parts. Read them in this order:

### 1. Scenario (first 2–5 sentences)

Skim for:
- **Data volume** (gigabytes vs terabytes vs petabytes).
- **Latency requirement** (real-time / near-real-time / batch / daily).
- **Team constraint** (existing Kafka expertise? new to AWS?).
- **Cost pressure** ("most cost-effective" / "with minimum operational overhead").
- **Compliance** (HIPAA, PCI, GDPR, financial audit).
- **Regional constraint** ("data must not leave EU").

### 2. The specific ask (last sentence before options)

- "Which solution meets these requirements **with the LEAST operational overhead**?"
- "Which solution is **MOST cost-effective**?"
- "Which approach provides the **HIGHEST throughput**?"
- "Which combination of actions? **(Select TWO / THREE.)**"

**These qualifiers change everything.** "Most cost-effective" nudges toward serverless / managed. "Least operational overhead" nudges toward Firehose over KDS, Serverless over provisioned, Bedrock Knowledge Bases over building a custom RAG.

### 3. The options

- Eliminate first, don't fall in love.
- Look for **subtle inaccuracies** — a wrong verb, an impossible combination, a service being asked to do something it can't.

---

## The Elimination Method — the Highest-Value Skill

For each of the 4 options, ask **"why can't this be right?"** in this order:

1. **Does the service actually exist / do this?** (E.g., "Firehose replays data" — no.)
2. **Does the answer violate a hard constraint?** (Lambda 15-min timeout for a 4-hour job.)
3. **Is there operational overhead the question specifically said we want to avoid?** ("least ops" → serverless.)
4. **Is there a simpler AWS-native alternative?** (Rolling your own crawler when Athena partition projection would do.)
5. **Does the answer require multiple services that AWS provides one for?** (Kinesis + Lambda + Firehose when Firehose alone suffices.)

Cross off 2 out of 4 → 50/50 → guess if you must.

---

## The Common Trap Phrasings

AWS exam writers are consistent. Certain phrases in the *scenario* map to certain answers.

### "Least operational overhead"

Preferred answers include:
- **Serverless** services (Lambda, Fargate, Athena, Redshift Serverless, EMR Serverless, DynamoDB On-Demand).
- **Managed** over self-managed (MSK Serverless > MSK Provisioned > self-run Kafka).
- **Fully managed pipelines** (Firehose > KDS + Lambda).
- **Bedrock Knowledge Bases** > building custom RAG.
- **Step Functions** > MWAA when both work.
- **Athena partition projection** > Glue crawler.
- **S3 Tables** > self-managed Iceberg on S3.

### "Most cost-effective"

- **Serverless** for spiky / unpredictable loads.
- **Provisioned** for steady predictable loads.
- **Reserved / Savings Plans** for 1–3 year commitments.
- **Spot** for interruptible workloads (task nodes only in EMR).
- **Glacier Deep Archive** for compliance archives.
- **Parquet + partition + compression** for Athena cost.
- **Intelligent-Tiering** when access patterns unknown.

### "Highest throughput" / "high concurrency"

- **Enhanced fan-out** in KDS.
- **DAX** in front of DynamoDB.
- **Redshift Concurrency Scaling** or Serverless.
- **Multi-cluster** warehouses.
- **Provisioned concurrency** for Lambda (avoids cold start).
- **Global Tables** for cross-region low-latency reads.

### "Fault tolerant / resilient / highly available"

- **Multi-AZ** (RDS, Aurora, MSK, ElastiCache).
- **Aurora replicas.**
- **Global Tables** (DynamoDB) or **Global Database** (Aurora).
- **Cross-Region Replication** (S3, DynamoDB, EFS).
- **DLQ + retries.**
- **Multi-region S3 backups.**

### "Real-time" vs "Near-real-time"

- **Real-time (sub-second)** → KDS + Lambda / Flink; Redshift streaming ingestion.
- **Near-real-time (minutes)** → Firehose.
- **Batch (hours)** → S3 + Glue + scheduled.

### "Compliance / regulated / auditable"

- **CloudTrail** (management + data events).
- **KMS customer-managed CMKs** with rotation.
- **Object Lock Compliance mode** for WORM.
- **Lake Formation** for fine-grained data access + audit.
- **Macie** for PII discovery.
- **VPC endpoints** (no public internet).
- **Config Rules** for continuous compliance.
- **Secrets Manager** for rotating creds.

### "Cross-account"

- Almost always involves **resource-based policies** (S3 bucket policy, KMS key policy) **AND** IAM policy on the caller side.
- **Assume-role** for user-to-service or cross-account service access.
- **Lake Formation cross-account sharing** for lake data.
- **Redshift Data Sharing** for warehouse data.

### "Ingestion from on-prem" or "migration"

- **DataSync** for file / EFS / NFS.
- **DMS** for databases.
- **Snow Family** for TB–PB physical transfer.
- **Transfer Family** for ongoing SFTP.
- **Direct Connect + VPN** for network layer.

---

## The Trick Wrong-Answer Patterns

AWS seeds these repeatedly:

### "The right service used incorrectly"

- Lambda used for a 4-hour job (15-min timeout).
- Firehose asked to replay data (it can't).
- SQS Standard asked for ordering (it's not ordered).
- Glue crawler used when Athena partition projection would be simpler.
- Redshift asked for sub-ms KV point reads (that's DynamoDB).

### "The service that no longer exists / was deprecated"

Version 1.1 removed some services. Watch for:
- **AWS Cloud9** — deprecated in 2024. Not in scope.
- **AWS CodeCommit** — deprecated 2024. Not in scope.
- **AWS Schema Conversion Tool (SCT)** — replaced by **DMS Schema Conversion**.
- **Kinesis Data Analytics for SQL** — replaced by Managed Service for Apache Flink.
- **Elasticsearch Service** — renamed **OpenSearch Service**.

### "The plausible but wrong combination"

- "Kinesis Data Streams → Firehose → OpenSearch" for real-time dashboards — wrong because Firehose adds 60s+ buffer. Use KDS → Lambda → OpenSearch, or KDS → OpenSearch Serverless directly.
- "Glue crawler daily" when partition projection would eliminate the crawler.
- "MWAA" when Step Functions is cheaper and simpler.
- "Provisioned Redshift RA3" when the workload is spiky and Serverless is cheaper.

### The "always" / "never" answer

Wrong answers often contain absolutes: "Always use X for Y." The real answers usually acknowledge trade-offs.

---

## Multi-Response ("Select TWO / THREE") Strategy

- **Count matches the question.** Select exactly the number asked. Fewer or more = zero credit.
- **Answers are independent.** Each right answer earns partial credit. Wrong answer = zero credit for the question (in AWS exams — this varies).
- **Two-answer questions often pair.** Look for two options that *together* solve the problem — one for ingest, one for storage; one for security, one for auth.

---

## Reading Long AWS Documentation Extracts

Some questions include AWS-doc-style pseudo-code (IAM policies, JSON configs). To parse fast:

- **JSON policy** — check `Effect`, `Action`, `Resource`, `Condition`. Wrong ARN → wrong answer.
- **CloudFormation snippet** — scan for the resource type first (`AWS::Kinesis::Stream`), then property names.
- **Glue DPU or Kinesis shard math** — plug in numbers, don't derive from scratch.

---

## The Night Before

- Skim the [Service Quick Reference](./05-service-quick-reference.md).
- Skim the [Common Traps](#the-common-trap-phrasings) in this file.
- **Do not learn new material.** If it's not in your head at 9pm, it won't be by 9am.
- Sleep 8 hours.
- Test-center: know the location, arrive 30 min early.
- Online-proctored: check camera, ID ready, room clear.

---

## The First 5 Minutes of the Exam

- **Read the first question slowly.** Set the pace.
- **Don't panic if it's hard.** Random distribution — hard ones exist. Flag and move.
- **Trust your prep.** Two weeks of focused study is enough.

---

## After the Exam

If you pass:
- Congrats. Add to LinkedIn + resume.
- Claim your digital badge (Credly).
- 50% discount code for next AWS exam within 12 months.

If you don't:
- Wait 14 days minimum before re-book.
- Review the exam feedback (per-domain breakdown).
- Focus prep on weakest domain.
- Take another practice test before rescheduling.

---

## Resources

- [Official AWS DEA-C01 exam page](https://aws.amazon.com/certification/certified-data-engineer-associate/)
- [Pearson VUE — schedule exam](https://home.pearsonvue.com/aws)
- [AWS Digital Badges (Credly)](https://www.credly.com/organizations/amazon-web-services/badges)

---

*You've got this. Come back the day after with the pass badge.*
