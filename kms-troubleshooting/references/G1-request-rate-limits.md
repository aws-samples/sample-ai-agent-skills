---
title: "G1 — Request Rate Limit Issues"
description: "Diagnose KMS request throttling and rate limit problems"
status: active
severity: HIGH
triggers:
  - "ThrottlingException"
  - "rate exceeded"
  - "request limit"
  - "KMS throttling"
  - "too many requests"
owner: devops-agent
objective: "Resolve KMS throttling and optimize request patterns"
context: "KMS enforces per-account, per-region request rate limits. Cryptographic operations share a combined quota (typically 5,500-30,000 req/sec for symmetric keys). Asymmetric operations have lower limits. Exceeding limits causes ThrottlingException. Use envelope encryption and data key caching to reduce API calls."
---

## Phase 1 — Triage

MUST:
- Check KMS throttling metrics: `aws cloudwatch get-metric-statistics --namespace AWS/KMS --metric-name ThrottleCount --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check current quota: `aws service-quotas get-service-quota --service-code kms --quota-code L-6E3AF000`
- Check CloudTrail for throttled events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=kms.amazonaws.com --max-results 50`
- Identify which operations are being throttled (Encrypt, Decrypt, GenerateDataKey)

SHOULD:
- Identify the source of high request volume (which service or application)
- Check if S3 bucket keys are enabled for S3 SSE-KMS workloads
- Review application code for unnecessary KMS API calls

MAY:
- Check per-key request metrics if available
- Compare current usage against the quota limit

## Phase 2 — Remediate

MUST:
- Implement exponential backoff and retry for throttled requests
- Enable S3 bucket keys for S3 SSE-KMS workloads (reduces KMS calls by up to 99%)
- Use envelope encryption with data key caching for application-level encryption

SHOULD:
- Request a quota increase if legitimate workload exceeds limits: `aws service-quotas request-service-quota-increase --service-code kms --quota-code L-6E3AF000 --desired-value <value>`
- Distribute workload across multiple KMS keys if possible
- Implement client-side data key caching using the AWS Encryption SDK

MAY:
- Use CloudWatch alarms to alert before hitting rate limits
- Optimize application to batch encryption operations

## Common Issues

- symptoms: "ThrottlingException on Decrypt calls from S3"
  diagnosis: "High-traffic S3 bucket with SSE-KMS generates a KMS call per object operation."
  resolution: "Enable S3 bucket keys to dramatically reduce KMS API calls."

- symptoms: "Intermittent ThrottlingException during peak hours"
  diagnosis: "Request rate exceeds the account/region quota during traffic spikes."
  resolution: "Implement data key caching, request a quota increase, or distribute across keys."

## Output Format

```yaml
root_cause: "request_rate_limit — <specific_cause>"
evidence:
  - type: throttle_metrics
    content: "<CloudWatch throttle count>"
  - type: quota
    content: "<current quota and usage>"
  - type: source
    content: "<primary source of KMS requests>"
severity: HIGH
mitigation:
  immediate: "Implement retry with backoff and enable S3 bucket keys"
  long_term: "Use data key caching and request quota increases"
```


## Safety Ratings

- `get-metric-statistics` (CloudWatch): GREEN — Read-only analysis of KMS throttle metrics
- `get-service-quota`: GREEN — Read-only retrieval of account quota limits
- `lookup-events` (CloudTrail): GREEN — Read-only audit log analysis of throttled events
- `describe-key`: GREEN — Read-only key metadata retrieval
- `request-service-quota-increase`: GREEN — Quota increase requests are reviewed by AWS and do not change key configuration
- `enable-key-rotation`: YELLOW — Rotation is additive; old key material is retained for decryption

## Escalation Conditions

- Remediation requires modifying a KMS key policy
- Fix involves granting cross-account key access
- Key is used for encryption of production data (S3, EBS, RDS)
- Key state change affects multiple services
- Throttling is causing production service degradation
- Quota increase request is urgent and requires AWS Support engagement

## Rollback

- Pre-change: "Save current key policy before modification"
- Verification: "Test encrypt/decrypt with the specific service after policy change"
- Revert: "Restore previous key policy if change breaks dependent services"

## Data Sensitivity

- HIGH: "Key policies reveal all authorized principals and services"
- HIGH: "Grant details expose cross-account relationships"
- MEDIUM: "Key metadata reveals encryption strategy"
- LOW: "Throttle metrics and quota limits are operational data"

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
