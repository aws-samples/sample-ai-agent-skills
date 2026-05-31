---
title: "H2 — S3 Delivery Issues"
description: "Diagnose AWS Config S3 delivery failures for configuration snapshots and history"
status: active
severity: HIGH
triggers:
  - "S3 delivery failed"
  - "config snapshot delivery"
  - "configuration history S3"
  - "config S3 bucket"
  - "delivery to S3"
owner: devops-agent
objective: "Identify and resolve S3 delivery failures preventing Config from storing configuration data"
context: "Config delivers configuration snapshots and history files to an S3 bucket. Delivery failures can be caused by bucket policy issues, KMS encryption problems, bucket deletion, cross-account access issues, or IAM role permissions. The delivery channel status provides the last delivery timestamp and error details."
---

## Phase 1 — Triage

MUST:
- Check delivery channel status: `aws configservice describe-delivery-channel-status`
- Verify the S3 bucket exists: `aws s3api head-bucket --bucket <config-bucket>`
- Check S3 bucket policy: `aws s3api get-bucket-policy --bucket <config-bucket>`
- Verify the Config service role: `aws iam get-role --role-name <config-role>`
- Check if the bucket uses KMS encryption: `aws s3api get-bucket-encryption --bucket <config-bucket>`

SHOULD:
- If KMS encrypted, verify key policy: `aws kms describe-key --key-id <key-id>`
- Check for S3 bucket public access block: `aws s3api get-public-access-block --bucket <config-bucket>`
- Verify the bucket is in the same region as Config (or cross-region delivery is configured)
- Test manual snapshot delivery: `aws configservice deliver-config-snapshot --delivery-channel-name <channel-name>`

MAY:
- Check S3 server access logs for denied requests
- Verify no S3 bucket lifecycle rules are deleting Config files prematurely
- Check for cross-account bucket access if using a centralized logging bucket

## Phase 2 — Remediate

MUST:
- Fix S3 bucket policy to allow Config service role access
- Ensure the Config role has s3:PutObject, s3:GetBucketAcl permissions
- If KMS encrypted, grant the Config role kms:GenerateDataKey and kms:Decrypt

SHOULD:
- Test delivery with a manual snapshot after fixing
- Update the delivery channel if the bucket name or prefix changed

MAY:
- Set up S3 event notifications to monitor Config deliveries
- Configure S3 lifecycle rules for appropriate retention of Config files

## Common Issues

- symptoms: "Delivery channel lastErrorCode shows InsufficientDeliveryPolicyException"
  diagnosis: "The S3 bucket policy does not grant the Config service role write access."
  resolution: "Add a bucket policy statement allowing s3:PutObject for the Config service role ARN with the correct prefix."

- symptoms: "Delivery fails with KMS AccessDeniedException"
  diagnosis: "The S3 bucket uses KMS encryption and the Config role lacks KMS permissions."
  resolution: "Update the KMS key policy to grant the Config service role kms:GenerateDataKey and kms:Decrypt."

- symptoms: "Delivery channel shows no recent successful delivery but no error"
  diagnosis: "The S3 bucket was deleted or the delivery channel references a non-existent bucket."
  resolution: "Create the S3 bucket or update the delivery channel to point to an existing bucket."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix S3 bucket policy | YELLOW | Permission change; reversible |
| Fix KMS key policy | YELLOW | Permission change; reversible |
| Test manual snapshot delivery | GREEN | Triggers delivery; no destructive change |
| Update delivery channel | YELLOW | Configuration change; reversible |

## Escalation Conditions

- Recorder configuration changes
- S3 bucket policy modifications
- KMS key policy modifications

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-delivery-channel-status` | LOW | Delivery status |
| `get-bucket-policy` | MEDIUM | S3 bucket policy |
| `get-bucket-encryption` | LOW | Encryption configuration |
| `describe-key` (KMS) | MEDIUM | Key policy details |

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
root_cause: "s3_delivery — <specific_cause>"
evidence:
  - type: delivery_status
    content: "<describe-delivery-channel-status output>"
  - type: s3_bucket
    content: "<bucket policy and encryption configuration>"
  - type: iam_role
    content: "<Config service role permissions>"
severity: HIGH
mitigation:
  immediate: "Fix S3 bucket policy or KMS key policy and test delivery"
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
