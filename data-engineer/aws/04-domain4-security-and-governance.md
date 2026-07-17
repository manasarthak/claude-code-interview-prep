# DEA-C01 Domain 4 — Data Security and Governance (18%)

**Weight:** ~18% of the exam.
**Prereqs:** [Governance / Lineage / Observability](../14-governance-lineage-observability.md)
**Related tasks:** T4.1 AuthN · T4.2 AuthZ · T4.3 Encryption/masking · T4.4 Audit logs · T4.5 Privacy + governance

---

## §1 — IAM (Task 4.1 & 4.2)

IAM is fundamental. Expect 5–8 questions across the exam.

### Core concepts

- **Users** — long-term identities. Deprecated pattern for humans (prefer SSO); OK for legacy service accounts.
- **Groups** — user containers for policy attachment.
- **Roles** — assumable identities (by services, users, external accounts). **Preferred for both AWS services and cross-account access.**
- **Policies** — JSON documents defining permissions. Attached to users / groups / roles.
- **Trust policy** — on a role, defines *who can assume it*.
- **Permissions boundary** — max permissions a principal can ever have. Advanced pattern.

### Policy types

| Type | Attached to | Effect |
|---|---|---|
| **Managed policy (AWS-managed)** | User / group / role | Reusable AWS-provided |
| **Managed policy (customer-managed)** | User / group / role | Reusable your-own |
| **Inline policy** | Single user / group / role | Embedded, one-off |
| **Resource-based policy** | S3 bucket / KMS key / Lambda / SQS / SNS / others | Attached to the resource, not the principal |
| **Service Control Policy (SCP)** | AWS Organizations OU / account | Guardrail — deny at org level |
| **Session policy** | Assumed role session | Narrows the role's max permissions for one session |

### Policy evaluation (memorize this order)

1. **Explicit DENY** anywhere → denied.
2. **Explicit ALLOW** somewhere → allowed.
3. **No allow** → implicit deny.

An SCP denial overrides IAM allows. IAM denials override IAM allows. Resource-based policies can grant cross-account when combined with IAM allow on the caller side.

### Least privilege

Skill 4.2.6 explicitly asks about "constructing custom policies that meet the principle of least privilege." Rules:

- Grant only the actions needed (not `s3:*`).
- Scope resources to specific ARNs, not `*`.
- Use `Condition` blocks for extra constraints (source IP, MFA, VPC).

Example:
```json
{
  "Effect": "Allow",
  "Action": ["s3:GetObject"],
  "Resource": "arn:aws:s3:::my-bucket/reports/*",
  "Condition": {
    "IpAddress": {"aws:SourceIp": "203.0.113.0/24"},
    "Bool": {"aws:MultiFactorAuthPresent": "true"}
  }
}
```

### IAM roles for AWS services

Common patterns:
- **Lambda execution role** — Lambda assumes; grants access to CloudWatch Logs + downstream resources.
- **EC2 instance role** (via instance profile) — code on the EC2 gets temporary creds.
- **Glue service role** — Glue assumes for job execution.
- **ECS task role** — permissions for the container process.
- **EKS IRSA (IAM Roles for Service Accounts)** — pod-level IAM identity.

### Cross-account access

- **Assume role from account B into account A** — set up trust policy in A's role allowing B; user in B calls `sts:AssumeRole`.
- **Resource-based policies** — attach directly to S3 bucket / KMS key granting external account access.

### AWS Secrets Manager vs Systems Manager Parameter Store

Both store secrets. Differences:

| | Secrets Manager | Parameter Store |
|---|---|---|
| Rotation | Automatic (via Lambda) | Manual |
| Cost | ~$0.40/secret/month + API calls | Free (Standard); paid (Advanced) |
| Cross-account | Yes | Limited |
| Max size | 64 KB | 4 KB (Standard) / 8 KB (Advanced) |
| Use for | RDS passwords, API keys with rotation | Config values, non-secret parameters |

Common exam pick:
- Rotating RDS password → Secrets Manager.
- Storing app config values → Parameter Store.

### VPC security groups (Skill 4.1.1)

- **Stateful** — return traffic auto-allowed.
- **Whitelist only** — no deny rules.
- **Attached to ENIs** — applied per-instance, per-Lambda-in-VPC, per-RDS.

Contrast with **NACLs** (stateless, allow + deny, per-subnet).

### SageMaker Unified Studio domains (Skill 4.1.7 — new)

- **Domain** — top-level org unit in Unified Studio.
- **Domain units** — sub-orgs (teams, business units).
- **Projects** — workspace with associated data assets + compute.

You won't be quizzed deeply here — just recognize the hierarchy.

---

## §2 — Encryption + KMS (Task 4.3)

### AWS KMS basics

- **Customer Master Key (CMK)** — the encryption key. Two flavors:
  - **AWS-managed CMK** (e.g., `aws/s3`) — created and rotated by AWS. Free.
  - **Customer-managed CMK** — you create, control policy, control rotation.
- **Data keys** — used to encrypt actual data. Encrypted by CMK. This is envelope encryption.

### Envelope encryption

The pattern KMS uses for large data:
1. `GenerateDataKey` → returns plaintext data key + encrypted-under-CMK data key.
2. Encrypt data locally with plaintext data key.
3. Store encrypted data + encrypted data key together.
4. To decrypt: `Decrypt(encrypted data key)` → plaintext data key → decrypt data.

Reason: KMS has a 4 KB payload limit; envelope encryption lets you handle any size.

### KMS key policies

Each CMK has a **key policy** (its own resource-based policy). Access requires *both* the key policy AND the caller's IAM policy to allow the action.

Common patterns:
- Grant Lambda role access to decrypt via CMK.
- Cross-account access via KMS key policy + IAM policy in the other account.

### KMS key rotation

- **AWS-managed CMKs** — auto-rotated annually.
- **Customer-managed CMKs** — annual rotation opt-in. Key material rotated; key ID stable.
- **Manual rotation** — create new CMK, re-encrypt old data.

### CloudHSM

Dedicated HSM for keys AWS can't access. FIPS 140-2 Level 3. Rare on the exam.

### S3 encryption modes (Skill 4.3.2 favorite)

| Mode | Key owner | Key management | Use for |
|---|---|---|---|
| **SSE-S3** | AWS | AWS handles keys | Default, easy |
| **SSE-KMS** | AWS + Customer | You manage CMK, KMS audits usage via CloudTrail | Regulated data, auditability |
| **SSE-KMS with S3 Bucket Keys** | AWS + Customer | Reduces KMS API calls (caches data key per bucket) | Cost optimization at high volume |
| **DSSE-KMS (Dual-layer)** | AWS + Customer | Two independent layers of encryption | Highest compliance |
| **SSE-C** | Customer supplies key per request | You manage keys | You handle key material yourself |

**Trap:** SSE-S3 is on by default for new buckets (since 2023). "How to enforce encryption" is now often "block unencrypted uploads via bucket policy."

Bucket policy to enforce SSE-KMS:
```json
{
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:PutObject",
  "Resource": "arn:aws:s3:::my-bucket/*",
  "Condition": {
    "StringNotEquals": {"s3:x-amz-server-side-encryption": "aws:kms"}
  }
}
```

### Encryption for other services

- **EBS** — encrypted volumes via KMS.
- **RDS / Aurora** — TDE via KMS, TLS in transit.
- **DynamoDB** — always encrypted at rest (AWS-owned CMK by default; customer-managed CMK optional).
- **Redshift** — cluster encryption via KMS or HSM.
- **Kinesis Data Streams** — SSE via KMS.
- **SNS / SQS** — SSE via KMS.
- **EFS** — encryption at rest and in transit.

### Encryption in transit

TLS on:
- S3 (via `aws:SecureTransport` condition to enforce).
- RDS (require SSL parameter).
- Redshift (enable SSL in cluster parameter group).
- API Gateway (TLS by default).
- Kinesis (TLS by default).

Common bucket policy to require TLS:
```json
{
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:*",
  "Resource": ["arn:aws:s3:::b/*", "arn:aws:s3:::b"],
  "Condition": {"Bool": {"aws:SecureTransport": "false"}}
}
```

### Cross-account encryption

For a Lambda in Account B to decrypt an S3 object in Account A encrypted with a CMK in A:
1. S3 bucket policy in A allows B's Lambda role → `s3:GetObject`.
2. KMS key policy in A allows B's Lambda role → `kms:Decrypt`.
3. Lambda's IAM policy in B allows both actions.

All three layers must align. Common exam trap: candidate forgets the KMS policy.

---

## §3 — Lake Formation (Task 4.2)

Lake Formation is the AWS-native governance layer over S3 data lakes.

### What Lake Formation does

- **Registers S3 locations** as governed data.
- **Fine-grained permissions** — column, row, cell.
- **Data filters** — row-level filters based on query context.
- **LF-tags** — attribute-based access control (ABAC). Tag columns / tables; grant permissions per tag.
- **Cross-account sharing** — grant permissions to accounts / IAM principals in other accounts.
- **Blueprints** — pre-built ingestion pipelines.

### Enforcement

Lake Formation intercepts queries from **Athena, Redshift Spectrum, EMR, Glue ETL, QuickSight, SageMaker** and applies its permission model. It layers on top of IAM.

### LF-tags — the modern pattern

Instead of granting permissions per table:

1. Define tags: `PII: sensitive`, `PII: public`, `Domain: finance`.
2. Tag tables and columns.
3. Grant `LF-Tag Access`: user X can query tables tagged `Domain: finance` and columns tagged `PII: public`.

**Massive win at scale** — one policy per role, not per table.

### Row / column / cell access

- **Column** — hide specific columns from certain roles (e.g., `salary` hidden from analysts).
- **Row filter** — WHERE clause applied server-side (e.g., `region = 'EU'` for EU analysts).
- **Cell filter** — combination.

### Federated identity in Lake Formation

Since 2023: Lake Formation supports IAM Identity Center (SSO) users directly. Users authenticate with corporate IdP, permissions applied to their SSO identity in Lake Formation.

### Redshift + Lake Formation

Redshift can consume Lake Formation permissions on Iceberg/Parquet tables via **Redshift Spectrum + Lake Formation integration**.

---

## §4 — PII, Macie, Compliance (Task 4.5)

Cross-read: [Governance / Lineage / Observability](../14-governance-lineage-observability.md).

### Amazon Macie

- **Scans S3** for PII / PHI / financial identifiers.
- **Uses ML** classifiers for common PII patterns (SSNs, credit cards, driver's licenses, passwords, health records).
- **Findings** — sends to EventBridge for automation, Security Hub, S3.
- **Custom identifiers** — regex-based for company-specific patterns.

Common pattern: Macie discovers PII in S3 → EventBridge → Lambda auto-tags or quarantines the object → Lake Formation applies stricter access.

### PII masking / anonymisation

Options on AWS:
- **DataBrew mask transformations** — hash, redact, encrypt columns.
- **Glue Studio built-in PII detection transform**.
- **Bedrock Guardrails PII masking** on LLM outputs.
- **Athena / Redshift UDFs** for redaction at query time.
- **Column-level access control via Lake Formation** — hide PII columns entirely.

Cross-read for the conceptual distinction (reversible vs irreversible): [Governance §Anonymisation vs Pseudonymisation](../14-governance-lineage-observability.md).

### Data privacy strategies (Skill 4.5.3)

**Prevent replicas to disallowed regions:**
- **S3 Cross-Region Replication (CRR)** with destination filter.
- **SCPs** to deny `s3:PutObject` to non-approved region buckets.
- **AWS Config rules** to detect misconfigured resources.

### Data sovereignty

- **Region choice** — data lives where you write it.
- **AWS regions with sovereignty guarantees** — Europe (Frankfurt / Paris / Stockholm), gov clouds.
- **CloudTrail cross-region trails + S3 CRR** — audit stays regional.

### AWS Config (revisit)

Tracks resource configuration changes over time. Answers "who changed the S3 bucket policy and when?" (Config gives the *state history*; CloudTrail gives the *API calls*.)

---

## §5 — VPC + Networking (Skill 4.1)

Enough networking to know what the exam wants.

### VPC endpoints (critical exam topic)

VPC endpoint = private access from your VPC to an AWS service without going over the public internet.

Two types:
- **Gateway endpoint** — S3 and DynamoDB only. Free. Route-table entry.
- **Interface endpoint (PrivateLink)** — ENI in your subnet with a private IP. Cost per hour + per GB. Available for most other services.

**When required on the exam:**
- Lambda in a VPC needs to talk to S3 → gateway endpoint (or NAT — but endpoint is preferred, cheaper).
- Glue needs to talk to Kinesis → interface endpoint.
- Regulated workloads that must never egress to public internet → interface endpoints for everything.

### AWS PrivateLink

Underlies interface endpoints. Also lets you expose your own services to other VPCs without VPC peering.

### Security groups vs NACLs

Already covered above.

### VPC Peering / Transit Gateway

- **Peering** — 1:1 VPC-to-VPC. Doesn't transit.
- **Transit Gateway** — hub-and-spoke for many VPCs and on-prem via VPN or Direct Connect.

### Direct Connect

Dedicated fiber from your data center to AWS. Predictable throughput + lower latency than internet.

### Route 53 (light)

DNS. Rarely deep on this exam but recognize its role.

### AWS WAF + Shield

- **WAF** — web application firewall (L7). SQL injection / XSS rules.
- **Shield Standard** — free DDoS protection.
- **Shield Advanced** — paid, 24/7 DDoS response team.

Not deep DE topics — recognize their existence.

---

## §6 — Audit Logging (Task 4.4)

### The audit stack

- **CloudTrail** — every API call.
- **CloudTrail Lake** — centralized SQL over CloudTrail events.
- **CloudWatch Logs** — application + service logs.
- **CloudWatch Logs Insights** — SQL-lite over logs.
- **S3 access logs** — bucket-level access logs (separate from CloudTrail data events).
- **VPC Flow Logs** — network flow metadata.
- **AWS Config** — config change history.

### Cross-account centralized logging

Common architecture:
- **Log archive account** — S3 bucket with cross-account write from all other accounts.
- **CloudTrail organization trail** — one trail captures all accounts, writes to log archive.
- **CloudWatch cross-account observability** — dashboards + alarms across accounts.

### Analyzing logs at scale

- **Athena over CloudTrail S3** — query API history with SQL.
- **CloudWatch Logs Insights** — sub-second queries over recent logs.
- **OpenSearch Service** — index logs for full-text + dashboard visualization.
- **EMR** — for TB-scale log processing.

---

## Common Exam Traps

**Q: "S3 objects must never be accessed unencrypted (in-transit)"**
A: Bucket policy denying `aws:SecureTransport = false`.

**Q: "Force all uploads to use KMS encryption"**
A: Bucket policy denying `PutObject` without `s3:x-amz-server-side-encryption = aws:kms`.

**Q: "Cross-account S3 access with customer-managed CMK"**
A: Three layers — bucket policy, KMS key policy, caller's IAM policy. Miss any and it fails.

**Q: "Prevent replication of PII to non-approved regions"**
A: SCPs + AWS Config rules + CRR destination filters.

**Q: "Grant analysts read of S3 lake but hide the `ssn` column"**
A: Lake Formation column-level permissions.

**Q: "Row-level filter by user's region"**
A: Lake Formation data filter with row-level predicate.

**Q: "Discover PII in existing S3 buckets"**
A: Amazon Macie.

**Q: "Rotate RDS master password every 30 days"**
A: Secrets Manager with rotation Lambda.

**Q: "Log every S3 GET on this bucket"**
A: CloudTrail data events on the bucket.

**Q: "Log all AWS API calls across all accounts to one place"**
A: CloudTrail organization trail → S3 in the log archive account.

**Q: "Grant Lambda in Account B access to secret in Account A"**
A: Resource policy on the secret allowing B's Lambda role + IAM policy in B allowing `secretsmanager:GetSecretValue`.

**Q: "Query S3 without going over the internet from Lambda in VPC"**
A: S3 gateway endpoint.

**Q: "Attribute-based access control on data lake"**
A: Lake Formation LF-tags.

---

## Concept-Check (self-quiz)

1. IAM policy evaluation order — write the three-step rule.
2. Resource-based policy vs identity-based policy — one example each.
3. SCP vs IAM policy — where does SCP live?
4. Assume-role flow — describe the trust policy's role.
5. Secrets Manager vs Parameter Store — one differentiator.
6. Envelope encryption — why does KMS use it?
7. S3 SSE-KMS vs SSE-S3 — one advantage each.
8. VPC gateway endpoint — which two services?
9. Lake Formation LF-tag — what's the workflow?
10. Column-level security in Lake Formation — supported query engines?
11. Macie — what does it scan and what does it find?
12. CloudTrail management vs data events — pick one for "S3 GET logs."
13. AWS Config — what history does it record?
14. Cross-account CMK decryption — three policies to align?
15. Enforce TLS on S3 bucket — bucket policy condition?

---

## Resources

- [IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [KMS Developer Guide](https://docs.aws.amazon.com/kms/latest/developerguide/)
- [Lake Formation Docs](https://docs.aws.amazon.com/lake-formation/)
- [Macie Docs](https://docs.aws.amazon.com/macie/)
- [S3 Security Best Practices](https://docs.aws.amazon.com/AmazonS3/latest/userguide/security-best-practices.html)
- [Security Pillar — Well-Architected](https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/welcome.html)

---

*Next: [Service Quick Reference](./05-service-quick-reference.md).*
