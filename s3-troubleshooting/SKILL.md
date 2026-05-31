---
name: s3-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon S3 problems
  by analyzing bucket configurations, access policies, replication
  state, and following structured runbooks. Activate when: access
  denied errors, bucket policy conflicts, CORS issues, replication
  failures, lifecycle rule problems, versioning issues, encryption
  errors, performance throttling, event notification failures,
  presigned URL issues, static website hosting problems, S3 Select
  errors, multipart upload failures, or the user says something is
  wrong with S3 without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with S3, IAM, KMS, CloudTrail,
  CloudWatch, and optionally CloudFront permissions.
---

# S3 Diagnostics

## When to use

Any S3 investigation where the console alone is insufficient — access denied analysis, bucket policy evaluation, cross-account access, replication troubleshooting, performance issues, encryption problems, or lifecycle rule debugging.

## Investigation workflow

### Step 1 — Collect and triage

```
aws s3api get-bucket-location --bucket <bucket>
aws s3api get-bucket-policy --bucket <bucket>
aws s3api get-bucket-acl --bucket <bucket>
aws s3api get-bucket-versioning --bucket <bucket>
aws s3api get-bucket-encryption --bucket <bucket>
aws s3api get-public-access-block --bucket <bucket>
```

### Step 2 — Domain deep dive

```
aws s3api get-bucket-replication --bucket <bucket>
aws s3api get-bucket-lifecycle-configuration --bucket <bucket>
aws s3api get-bucket-cors --bucket <bucket>
aws s3api get-bucket-notification-configuration --bucket <bucket>
aws s3api get-bucket-logging --bucket <bucket>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=ResourceName,AttributeValue=<bucket>
aws cloudwatch get-metric-statistics --namespace AWS/S3 --metric-name 4xxErrors ...
```

Read `references/s3-guardrails.md` before concluding on any S3 issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `get-bucket-policy` | Bucket policy evaluation |
| `get-bucket-acl` | Legacy ACL permissions |
| `get-public-access-block` | Public access settings |
| `get-bucket-encryption` | Default encryption config |
| `get-bucket-versioning` | Versioning state |
| `get-bucket-replication` | Cross-region/account replication |
| `get-bucket-lifecycle-configuration` | Lifecycle rules |
| `get-bucket-cors` | CORS configuration |
| `get-bucket-notification-configuration` | Event notifications |
| `head-object` | Object metadata, storage class, encryption |
| `get-object-acl` | Object-level ACL |
| `get-object-retention` | Object Lock retention |

## Gotchas: S3

- S3 access evaluation: IAM policy + bucket policy + ACL + S3 Block Public Access + VPC endpoint policy + SCP. An explicit deny in ANY of these blocks access.
- Bucket policies have a 20 KB size limit. Complex policies hit this limit.
- S3 Block Public Access has FOUR independent settings at both account and bucket level. Account-level settings override bucket-level settings.
- Object ownership: BucketOwnerEnforced disables ACLs entirely. BucketOwnerPreferred and ObjectWriter still use ACLs. New buckets default to BucketOwnerEnforced.
- Versioning cannot be disabled once enabled — only suspended. Suspended versioning still retains existing versions.
- Delete markers are NOT objects. They are zero-byte markers that make the current version appear deleted. The previous versions still exist.
- S3 Replication requires versioning on BOTH source and destination. It does NOT replicate existing objects (use S3 Batch Replication for that). Delete markers are optionally replicated.
- Lifecycle rules: transitions have minimum duration requirements (30 days to IA, 90 days to Glacier). Objects smaller than 128 KB are not transitioned to IA/Intelligent-Tiering.
- S3 request rate: 5,500 GET/HEAD and 3,500 PUT/POST/DELETE per prefix per second. Spread keys across prefixes for high throughput.
- Presigned URLs inherit the permissions of the IAM principal that created them. If the principal's permissions change, the URL may stop working.
- S3 event notifications: each event type can only have ONE destination per configuration. Duplicate event configurations cause errors.
- KMS-encrypted objects require kms:Decrypt permission on the KMS key for the reader, AND kms:GenerateDataKey for the writer.
- Cross-account access requires BOTH the source account IAM policy AND the destination bucket policy to allow access.
- S3 Transfer Acceleration uses CloudFront edge locations. It helps for long-distance uploads but adds cost.
- Multipart uploads that are not completed or aborted leave orphaned parts that incur storage costs. Use lifecycle rules to clean them up.

### Storage class comparison

| Class | Retrieval | Min Duration | Use Case |
|-------|-----------|-------------|----------|
| Standard | Instant | None | Frequent access |
| Intelligent-Tiering | Instant | 30 days | Unknown access patterns |
| Standard-IA | Instant | 30 days | Infrequent, rapid retrieval |
| One Zone-IA | Instant | 30 days | Infrequent, single AZ OK |
| Glacier Instant | Instant | 90 days | Archive, instant retrieval |
| Glacier Flexible | Minutes-hours | 90 days | Archive, flexible retrieval |
| Glacier Deep Archive | 12-48 hours | 180 days | Long-term archive |

## Anti-hallucination rules

1. Always cite specific bucket policies, IAM policies, or API responses as evidence.
2. Versioning cannot be disabled, only suspended. Never claim versioning can be turned off.
3. S3 Block Public Access account settings override bucket settings. Never ignore account-level blocks.
4. Replication does NOT replicate existing objects. Never claim enabling replication copies existing data.
5. Objects < 128 KB are NOT transitioned to IA classes. Never claim all objects transition.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 34 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Access & Permissions | A1-A4 | Access denied, bucket policy conflicts, cross-account access, VPC endpoint |
| B — Public Access | B1-B3 | Block public access, ACL issues, bucket policy public access |
| C — Encryption | C1-C3 | SSE-S3/SSE-KMS errors, cross-account KMS, client-side encryption |
| D — Replication | D1-D3 | Replication failures, cross-account replication, replication lag |
| E — Lifecycle | E1-E3 | Transition failures, expiration issues, abort multipart |
| F — Versioning | F1-F3 | Delete marker confusion, version management, MFA delete |
| G — Performance | G1-G3 | Throttling (503), transfer acceleration, multipart upload |
| H — Events & Integration | H1-H3 | Event notification failures, Lambda triggers, SQS/SNS delivery |
| I — Static Hosting & CORS | I1-I3 | Website hosting errors, CORS failures, CloudFront integration |
| J — Object Lock & Compliance | J1-J2 | Retention issues, legal hold |
| Z — Catch-All | Z1 | General troubleshooting |
