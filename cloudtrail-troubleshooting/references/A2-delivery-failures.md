---
title: "A2 — Delivery Failures"
description: "Diagnose CloudTrail log delivery failures to S3"
status: active
severity: CRITICAL
triggers:
  - "logs not delivered"
  - "delivery failed"
  - "trail not logging"
  - "LatestDeliveryError"
owner: devops-agent
objective: "Restore CloudTrail log delivery to S3"
context: "Log delivery failures mean events are not being recorded to S3. Causes include S3 bucket policy changes, bucket deletion, KMS key issues, or trail stopped. Check trail status for delivery errors."
---

## Phase 1 — Triage

MUST:
- Check trail status: `aws cloudtrail get-trail-status --name <trail-name>`
- Check if logging is enabled: look for `IsLogging: true` in status
- Check latest delivery error: look for `LatestDeliveryError` in status
- Verify S3 bucket still exists: `aws s3api head-bucket --bucket <bucket-name>`

SHOULD:
- Check S3 bucket policy hasn't changed: `aws s3api get-bucket-policy --bucket <bucket-name>`
- Verify KMS key is still accessible: `aws kms describe-key --key-id <key-id>`
- Check latest delivery time: `LatestDeliveryTime` in trail status
- Review CloudTrail for trail modification events

MAY:
- Check S3 bucket versioning and lifecycle rules
- Verify no S3 bucket policy denies CloudTrail

## Phase 2 — Remediate

MUST:
- Start logging if stopped: `aws cloudtrail start-logging --name <trail-name>`
- Fix S3 bucket policy to allow CloudTrail writes
- Fix KMS key policy if encryption errors

SHOULD:
- Verify delivery resumes: `aws cloudtrail get-trail-status --name <trail-name>`
- Set up SNS notifications for delivery failures
- Monitor LatestDeliveryError regularly

MAY:
- Set up CloudWatch alarm on trail delivery metrics
- Create redundant trail to different S3 bucket

## Common Issues

- symptoms: "LatestDeliveryError: Access Denied"
  diagnosis: "S3 bucket policy changed or removed CloudTrail permissions."
  resolution: "Restore CloudTrail permissions in S3 bucket policy."

- symptoms: "IsLogging: false"
  diagnosis: "Trail logging was stopped manually or by automation."
  resolution: "Start logging: aws cloudtrail start-logging --name <trail-name>"

- symptoms: "KMS key disabled or pending deletion"
  diagnosis: "KMS key used for encryption is no longer accessible."
  resolution: "Re-enable KMS key or update trail to use a different key."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Start logging if stopped | GREEN | Restores event recording |
| Fix S3 bucket policy | YELLOW | Bucket policy changes affect all access to the bucket |
| Fix KMS key policy | YELLOW | Key policy changes affect all key usage |
| Set up SNS notifications | GREEN | Monitoring-only addition |

## Escalation Conditions

- Trail logging stopped (IsLogging: false) — escalate immediately to security operations team
- Log delivery gap exceeding 1 hour — escalate to security team for gap analysis
- S3 bucket policy modified by unknown principal — escalate to security team for investigation
- KMS key disabled or pending deletion — escalate to key administrator immediately

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws cloudtrail get-trail-status` | HIGH | Exposes delivery status, errors, and timestamps |
| `aws s3api get-bucket-policy` | HIGH | Exposes bucket access policy |
| `aws kms describe-key` | MEDIUM | Exposes key status and metadata |
| `aws s3api head-bucket` | LOW | Bucket existence check only |

## Prohibited Actions

- **NEVER** suggest stopping or deleting CloudTrail trails
- **NEVER** suggest disabling log file validation
- **NEVER** suggest removing S3 bucket policy for log delivery
- **NEVER** ignore delivery failures — they represent gaps in audit logging

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Started logging | Stop logging only if trail was intentionally stopped (use with extreme caution) |
| Fixed S3 bucket policy | Restore previous bucket policy from backup |
| Fixed KMS key policy | Restore previous key policy |
| Set up SNS notifications | Remove SNS configuration from trail |

## Output Format

```yaml
root_cause: "delivery_failure — <specific_cause>"
evidence:
  - type: trail_status
    content: "<trail status and delivery error>"
  - type: bucket_status
    content: "<S3 bucket accessibility>"
severity: CRITICAL
mitigation:
  immediate: "Restore log delivery"
  long_term: "Set up delivery monitoring and alerting"
```

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
  - command: "lookup-events"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-trail-status"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-trails"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling CloudTrail logging"
  - "NEVER suggest deleting trail logs"
  - "NEVER reduce logging scope in production"
