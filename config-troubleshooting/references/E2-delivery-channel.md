---
title: "E2 — Delivery Channel Issues"
description: "Diagnose AWS Config delivery channel configuration and delivery failures"
status: active
severity: HIGH
triggers:
  - "delivery channel error"
  - "delivery failed"
  - "S3 delivery"
  - "SNS notification"
  - "config delivery"
owner: devops-agent
objective: "Identify and resolve delivery channel issues preventing Config from persisting configuration items and sending notifications"
context: "The delivery channel defines where Config sends configuration items (S3) and notifications (SNS). Delivery failures prevent configuration history from being stored and can cause rule evaluation gaps. The delivery channel status shows last successful delivery timestamps."
---

## Phase 1 — Triage

MUST:
- Check delivery channel configuration: `aws configservice describe-delivery-channels`
- Check delivery channel status: `aws configservice describe-delivery-channel-status`
- Verify the S3 bucket exists: `aws s3api head-bucket --bucket <config-bucket>`
- Check S3 bucket policy: `aws s3api get-bucket-policy --bucket <config-bucket>`
- Verify the Config service role can write to S3: `aws iam get-role --role-name <config-role>`

SHOULD:
- Check SNS topic if configured: `aws sns get-topic-attributes --topic-arn <topic-arn>`
- Verify KMS key if S3 bucket uses encryption: `aws kms describe-key --key-id <key-id>`
- Check CloudTrail for delivery errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=config.amazonaws.com --max-results 20`

MAY:
- Check S3 bucket versioning and lifecycle rules
- Verify cross-account S3 bucket access if using a centralized bucket

## Phase 2 — Remediate

MUST:
- Fix S3 bucket policy to allow Config service role access
- Ensure the S3 bucket exists and is in the correct region
- Verify the Config service role has s3:PutObject permission

SHOULD:
- Update the delivery channel if the S3 bucket or SNS topic changed
- Fix KMS key policy if the bucket uses customer-managed encryption
- Test delivery by requesting a configuration snapshot: `aws configservice deliver-config-snapshot --delivery-channel-name <channel-name>`

MAY:
- Set up S3 bucket notifications for monitoring delivery
- Configure S3 lifecycle rules for configuration snapshot retention

## Common Issues

- symptoms: "Delivery channel status shows lastErrorCode with S3 access denied"
  diagnosis: "The S3 bucket policy does not allow the Config service role to write objects."
  resolution: "Update the S3 bucket policy to allow s3:PutObject and s3:GetBucketAcl for the Config service role."

- symptoms: "Delivery channel status shows stale lastSuccessfulTime"
  diagnosis: "Delivery has been failing silently. The S3 bucket may have been deleted or the policy changed."
  resolution: "Verify the S3 bucket exists and the bucket policy allows Config access. Request a manual snapshot delivery."

- symptoms: "SNS notifications not being sent"
  diagnosis: "The SNS topic does not exist, the topic policy denies Config, or the topic ARN is incorrect."
  resolution: "Verify the SNS topic exists and its policy allows config.amazonaws.com to publish."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix S3 bucket policy | YELLOW | Permission change; reversible |
| Fix KMS key policy | YELLOW | Permission change; reversible |
| Test manual snapshot delivery | GREEN | Triggers delivery; no destructive change |

## Escalation Conditions

- Recorder configuration changes
- Delivery channel changes
- S3 bucket policy modifications

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-delivery-channels` | LOW | Delivery channel configuration |
| `describe-delivery-channel-status` | LOW | Delivery status |
| `get-bucket-policy` | MEDIUM | S3 bucket policy |

## Prohibited Actions

- NEVER suggest stopping Config recorder
- NEVER suggest deleting delivery channel
- NEVER suggest disabling organization rules

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| S3 bucket policy fix | Revert bucket policy to previous version |
| KMS key policy fix | Revert key policy to previous version |
| Delivery channel update | Revert delivery channel configuration |

## Output Format

```yaml
root_cause: "delivery_channel_issue — <specific_cause>"
evidence:
  - type: delivery_channel
    content: "<describe-delivery-channels output>"
  - type: delivery_status
    content: "<describe-delivery-channel-status output>"
  - type: s3_bucket
    content: "<bucket policy and accessibility>"
severity: HIGH
mitigation:
  immediate: "Fix the delivery channel configuration and test with a manual snapshot"
  long_term: "Monitor delivery status and set up alerts for delivery failures"
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
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling Config rules to fix compliance"
  - "NEVER suggest deleting the configuration recorder"
  - "NEVER remove remediation actions without review"
