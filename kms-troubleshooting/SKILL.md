---
name: kms-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS KMS problems
  by analyzing key policies, grants, cross-account access, key states,
  encryption operations, service integrations, quotas, and following
  structured runbooks. Activate when: access denied on KMS operations,
  key policy evaluation issues, default key policy confusion, grant
  management problems, grant token issues, cross-account key access
  failures, ViaService condition errors, disabled key errors, pending
  deletion recovery, key rotation configuration, encryption context
  mismatches, asymmetric key issues, multi-region key problems,
  S3 SSE-KMS errors, EBS encryption failures, RDS encryption issues,
  request rate limit throttling, key quota limits, key policy vs IAM
  policy conflicts, audit logging gaps, or the user says something
  is wrong with KMS without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with KMS, IAM, S3, EC2, RDS,
  CloudTrail, CloudWatch, and optionally Organizations, Lambda,
  and EBS permissions.
---

# AWS KMS Diagnostics

## When to use

Any KMS investigation where the console alone is insufficient — key policy analysis, grant management, cross-account access debugging, key state troubleshooting, encryption context issues, service integration problems, throttling analysis, or security auditing.

## Investigation workflow

### Step 1 — Collect and triage

```
aws kms describe-key --key-id <key-id>
aws kms get-key-policy --key-id <key-id> --policy-name default
aws kms list-grants --key-id <key-id>
aws kms get-key-rotation-status --key-id <key-id>
aws kms list-aliases --key-id <key-id>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=kms.amazonaws.com --max-results 20
```

### Step 2 — Domain deep dive

```
aws kms list-key-policies --key-id <key-id>
aws kms list-resource-tags --key-id <key-id>
aws kms describe-key --key-id <key-id> --query 'KeyMetadata.{State:KeyState,Origin:Origin,Manager:KeyManager,Spec:KeySpec,Usage:KeyUsage,MultiRegion:MultiRegionConfiguration}'
aws iam simulate-principal-policy --policy-source-arn <principal-arn> --action-names kms:Decrypt kms:Encrypt --resource-arns <key-arn>
aws cloudwatch get-metric-statistics --namespace AWS/KMS --metric-name ThrottleCount --start-time <start> --end-time <end> --period 300 --statistics Sum
```

### Step 3 — Detailed investigation

```
aws kms list-retirable-grants --retiring-principal <principal-arn>
aws kms list-keys --query 'Keys[*].KeyId'
aws kms describe-custom-key-stores
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=Decrypt --max-results 20
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=GenerateDataKey --max-results 20
aws service-quotas get-service-quota --service-code kms --quota-code L-6E3AF000
```

Read `references/kms-guardrails.md` before concluding on any KMS issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-key` | Key metadata, state, spec, usage, multi-region config |
| `get-key-policy` | Key policy document (JSON) |
| `list-grants` | All grants on a key |
| `list-retirable-grants` | Grants a principal can retire |
| `get-key-rotation-status` | Whether automatic rotation is enabled |
| `list-aliases` | Key aliases |
| `list-key-policies` | Policy names (usually just "default") |
| `list-resource-tags` | Tags on a key |
| `enable-key` / `disable-key` | Change key enabled/disabled state |
| `schedule-key-deletion` / `cancel-key-deletion` | Manage key deletion |
| `create-grant` / `retire-grant` / `revoke-grant` | Grant lifecycle |
| `encrypt` / `decrypt` / `generate-data-key` | Cryptographic operations |
| `re-encrypt` | Re-encrypt under a different key |
| `simulate-principal-policy` | Test IAM policy evaluation |

## Gotchas: AWS KMS

- Key policy is the PRIMARY access control. Unlike most AWS services, the KMS key policy is the primary authorization mechanism. IAM policies only work if the key policy explicitly allows the account (via the root principal statement). Without this statement, IAM policies are completely ignored regardless of what they grant.
- Default key policy enables IAM policies. The default key policy includes a statement granting the account root principal full access. This statement is what enables IAM policies to work for the key. Removing this statement locks out all IAM-based access — only principals explicitly named in the key policy can use the key.
- Key deletion has a 7-30 day waiting period. When you schedule key deletion, the key enters PendingDeletion state for 7-30 days (configurable). During this period, the key cannot be used for any cryptographic operations. Deletion can be cancelled during the waiting period. Once deleted, the key and all data encrypted with it are PERMANENTLY unrecoverable.
- Disabled keys fail ALL crypto operations. A disabled key cannot be used for encrypt, decrypt, generate-data-key, or any cryptographic operation. Services using the key (S3, EBS, RDS) will fail to read/write encrypted data. Re-enabling the key immediately restores functionality.
- Encryption context is like a condition — must match on decrypt. Encryption context is a set of key-value pairs that are cryptographically bound to the ciphertext. The SAME encryption context must be provided during decryption. It is logged in CloudTrail and can be used in key policy conditions. It is NOT secret — it is stored in plaintext.
- Grants are for temporary/programmatic access. Grants provide temporary, scoped permissions on a KMS key without modifying the key policy. They are commonly used by AWS services (EBS, RDS) to access keys on behalf of users. Grants can be retired by the grantee or revoked by the key administrator.
- AWS-managed keys cannot be modified/rotated/deleted by customer. Keys with alias prefix `aws/` (e.g., aws/s3, aws/ebs) are managed by AWS. Customers cannot view or modify their key policies, manually rotate them, disable them, or schedule them for deletion. They rotate automatically every year.
- Multi-region keys are NOT global — replicas in each region. Multi-region keys share the same key material across regions but are separate KMS keys with separate ARNs, policies, and grants in each region. Creating a multi-region key in one region does not automatically make it available in other regions — you must explicitly create replicas.
- Request rate limits vary by key type and operation. KMS has per-account, per-region request rate limits. Cryptographic operations (encrypt, decrypt, generate-data-key) share a combined quota. Symmetric keys have higher limits than asymmetric keys. Exceeding limits causes ThrottlingException. Use data key caching to reduce KMS API calls.

## Anti-hallucination rules

1. Always cite specific key IDs, policy documents, or API responses as evidence.
2. Key policy is the primary access control. Never suggest IAM policies alone can grant KMS access without the key policy allowing it.
3. AWS-managed keys (aws/*) cannot be modified by customers. Never suggest changing their policies or rotation.
4. Key deletion is permanent after the waiting period. Never suggest recovering a deleted key.
5. Encryption context must match on decrypt. Never suggest it is optional during decryption if it was used during encryption.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 26 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Key Policy | A1-A3 | Access denied, policy evaluation, default key policy |
| B — Grants | B1-B2 | Grant management, grant tokens |
| C — Cross-Account | C1-C2 | Cross-account key access, ViaService conditions |
| D — Key State | D1-D3 | Disabled key, pending deletion, key rotation |
| E — Encryption | E1-E3 | Encryption context, asymmetric keys, multi-region keys |
| F — Integration | F1-F3 | S3 SSE-KMS, EBS encryption, RDS encryption |
| G — Quotas | G1-G2 | Request rate limits, key quotas |
| H — Security | H1-H2 | Key policy vs IAM policy, audit logging |
| Z — Catch-All | Z1 | General troubleshooting |
