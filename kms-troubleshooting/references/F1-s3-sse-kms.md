---
title: "F1 — S3 SSE-KMS Integration Issues"
description: "Diagnose S3 server-side encryption with KMS key problems"
status: active
severity: HIGH
triggers:
  - "S3 SSE-KMS"
  - "S3 encryption error"
  - "S3 KMS key"
  - "bucket encryption"
  - "S3 access denied KMS"
owner: devops-agent
objective: "Resolve S3 SSE-KMS encryption and decryption failures"
context: "S3 SSE-KMS uses KMS keys to encrypt objects. Each PutObject generates a data key via GenerateDataKey, and each GetObject calls Decrypt. This means KMS permissions are needed for both read and write. S3 adds encryption context automatically (aws:s3:arn). High-traffic buckets can hit KMS request rate limits."
---

## Phase 1 — Triage

MUST:
- Check bucket encryption configuration: `aws s3api get-bucket-encryption --bucket <bucket-name>`
- Describe the KMS key used: `aws kms describe-key --key-id <key-id>`
- Check key policy for S3 access: `aws kms get-key-policy --key-id <key-id> --policy-name default`
- Verify the caller has both S3 and KMS permissions

SHOULD:
- Check for KMS throttling: `aws cloudwatch get-metric-statistics --namespace AWS/KMS --metric-name ThrottleCount --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check CloudTrail for KMS errors related to S3: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=kms.amazonaws.com --max-results 20`

MAY:
- Check S3 bucket policy for KMS-related conditions
- Verify cross-account access if bucket and key are in different accounts

## Phase 2 — Remediate

MUST:
- Grant the caller kms:GenerateDataKey and kms:Decrypt on the key
- Ensure the key policy allows the S3 service or the caller principal
- Fix cross-account permissions if bucket and key are in different accounts

SHOULD:
- Use S3 bucket keys to reduce KMS API calls (reduces cost and throttling)
- Monitor KMS request rates for high-traffic buckets
- Use the AWS-managed key (aws/s3) for simpler permission management

MAY:
- Request a KMS quota increase for high-traffic workloads
- Implement client-side caching of data keys

## Common Issues

- symptoms: "Access Denied when uploading to S3 with SSE-KMS"
  diagnosis: "Caller lacks kms:GenerateDataKey permission on the encryption key."
  resolution: "Grant kms:GenerateDataKey and kms:Decrypt to the caller via key policy or IAM."

- symptoms: "KMS ThrottlingException on high-traffic S3 bucket"
  diagnosis: "Too many KMS API calls from S3 operations exceeding the request rate limit."
  resolution: "Enable S3 bucket keys to reduce KMS calls. Request a quota increase if needed."

- symptoms: "Cross-account S3 access fails with KMS error"
  diagnosis: "The KMS key policy does not grant access to the external account."
  resolution: "Add the external account to the key policy and ensure their IAM policy includes KMS permissions."

## Output Format

```yaml
root_cause: "s3_sse_kms_issue — <specific_cause>"
evidence:
  - type: bucket_encryption
    content: "<bucket encryption configuration>"
  - type: key_policy
    content: "<KMS key policy>"
  - type: throttling
    content: "<KMS throttle metrics>"
severity: HIGH
mitigation:
  immediate: "Fix KMS permissions for S3 operations"
  long_term: "Enable S3 bucket keys and monitor KMS request rates"
```


## Safety Ratings

- `get-bucket-encryption`: GREEN — Read-only retrieval of S3 bucket encryption configuration
- `describe-key`: GREEN — Read-only key metadata retrieval
- `get-key-policy`: GREEN — Read-only retrieval of key policy document
- `get-metric-statistics` (CloudWatch): GREEN — Read-only throttle metrics analysis
- `put-key-policy` (S3 access): YELLOW — Affects S3 service access to the key
- `put-bucket-encryption`: YELLOW — Changes how new objects are encrypted in the bucket
- `enable-key-rotation`: YELLOW — Rotation is additive; old key material is retained for decryption

## Escalation Conditions

- Remediation requires modifying a KMS key policy
- Fix involves granting cross-account key access
- Key is used for encryption of production data (S3, EBS, RDS)
- Key state change affects multiple services
- Encryption configuration change affects cross-account S3 access

## Rollback

- Pre-change: "Save current key policy before modification"
- Verification: "Test encrypt/decrypt with the specific service after policy change"
- Revert: "Restore previous key policy if change breaks dependent services"

## Data Sensitivity

- HIGH: "Key policies reveal all authorized principals and services"
- HIGH: "Grant details expose cross-account relationships"
- MEDIUM: "Key metadata reveals encryption strategy"
- MEDIUM: "Bucket encryption configuration reveals the KMS key ARN used"

## Prohibited Actions

- NEVER suggest `"kms:*"` in a key policy
- NEVER suggest making a key policy public with `"Principal": "*"`
- NEVER suggest disabling key rotation as a fix
- NEVER suggest scheduling key deletion without confirming no active usage
- NEVER suggest creating a new key to bypass policy issues (data becomes inaccessible)

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "get-key-policy"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-key"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-grants"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest kms:* on key policies"
  - "NEVER suggest disabling key rotation"
  - "NEVER suggest making key policies public"

## Phase 3 � Rollback

- Document the current state before making any changes
- If remediation worsens the issue, revert to the documented pre-change state
- Remove any temporary resources created during investigation
- Verify the system returns to its previous functional state after rollback
