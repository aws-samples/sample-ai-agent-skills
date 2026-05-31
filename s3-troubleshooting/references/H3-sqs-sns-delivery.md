---
title: "H3 — SQS/SNS Delivery from S3"
description: "Diagnose S3 event notification delivery failures to SQS and SNS"
status: active
severity: MEDIUM
triggers:
  - "SQS not receiving S3 events"
  - "SNS not receiving S3 events"
  - "Queue policy S3"
  - "Topic policy S3"
owner: devops-agent
objective: "Fix S3 event delivery to SQS queues and SNS topics"
context: "S3 event notifications to SQS and SNS require the queue/topic resource policy to allow s3.amazonaws.com to publish. If the queue uses SSE-KMS, the KMS key must allow S3 to use it. Cross-account delivery requires additional policy configuration."
---

## Phase 1 — Triage

MUST:
- Check S3 notification configuration: `aws s3api get-bucket-notification-configuration --bucket <bucket>`
- For SQS: check queue policy: `aws sqs get-queue-attributes --queue-url <url> --attribute-names Policy`
- For SNS: check topic policy: `aws sns get-topic-attributes --topic-arn <arn> --query 'Attributes.Policy'`
- Verify the queue/topic is in the same region as the bucket

SHOULD:
- Check if the SQS queue uses SSE-KMS encryption
- If SSE-KMS: verify the KMS key policy allows S3 to use it
- Check for messages in the dead-letter queue if configured

MAY:
- Send a test message to verify the queue/topic is working
- Check CloudTrail for notification delivery errors

## Phase 2 — Remediate

MUST:
- Update SQS queue policy to allow S3:
  ```json
  {
    "Effect": "Allow",
    "Principal": {"Service": "s3.amazonaws.com"},
    "Action": "sqs:SendMessage",
    "Resource": "<queue-arn>",
    "Condition": {"ArnEquals": {"aws:SourceArn": "arn:aws:s3:::<bucket>"}}
  }
  ```
- Update SNS topic policy similarly with sns:Publish
- If SSE-KMS on SQS: add kms:GenerateDataKey and kms:Decrypt for s3.amazonaws.com to the key policy

SHOULD:
- Use aws:SourceAccount condition for additional security
- Test with a sample upload after fixing the policy
- Set up a dead-letter queue for failed message delivery

MAY:
- Consider EventBridge as an alternative for more flexible routing
- Enable SQS/SNS delivery logging for ongoing monitoring

## Common Issues

- symptoms: "S3 notification configuration save fails with invalid SQS queue"
  diagnosis: "SQS queue policy does not allow s3.amazonaws.com to send messages."
  resolution: "Update the queue policy to allow sqs:SendMessage from s3.amazonaws.com."

- symptoms: "Events delivered but messages are empty or malformed"
  diagnosis: "SNS raw message delivery is not enabled, wrapping the S3 event in SNS envelope."
  resolution: "Enable raw message delivery on the SNS subscription if the consumer expects raw S3 events."

- symptoms: "Events stop after enabling SSE-KMS on the SQS queue"
  diagnosis: "S3 cannot encrypt messages with the KMS key."
  resolution: "Update the KMS key policy to allow s3.amazonaws.com to use kms:GenerateDataKey."

## Output Format

```yaml
root_cause: "sqs_sns_delivery — <specific_cause>"
evidence:
  - type: queue_or_topic_policy
    content: "<resource policy>"
  - type: notification_config
    content: "<S3 notification configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix the queue/topic resource policy"
  long_term: "Add dead-letter queues and consider EventBridge for flexibility"
```


## Safety Ratings

| Rating | Justification |
|--------|--------------|
| 🟡 YELLOW | Involves modifying SQS queue policies, SNS topic policies, and KMS key policies to allow S3 event delivery. Resource policy changes are state-changing but recoverable. Uses get-queue-attributes and get-topic-attributes for diagnosis. |

## Escalation Conditions
- Remediation requires modifying bucket policy in a production account
- Fix involves changing Block Public Access settings
- Remediation affects cross-account access
- Change impacts encryption configuration
- Queue/topic policy changes affect event delivery to production consumers

## Rollback
- Pre-change: "Save current bucket policy/ACL/CORS before modification"
- Verification: "Test access with the specific operation after change"
- Revert: "Restore previous configuration if change causes unintended access"

## Data Sensitivity
- HIGH: "Bucket policies reveal all authorized principals"
- HIGH: "ACLs expose cross-account grants"
- MEDIUM: "Queue/topic policies reveal integration architecture"
- LOW: "Bucket metrics and storage class distribution"

## Prohibited Actions
- NEVER suggest disabling S3 Block Public Access as a remediation
- NEVER suggest `"Principal": "*"` without restrictive Condition keys
- NEVER suggest removing bucket encryption
- NEVER suggest `s3:*` in any policy fix
- NEVER suggest deleting a bucket to resolve configuration issues

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
  - command: "get-bucket-policy"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-bucket-acl"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-public-access-block"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling S3 Block Public Access"
  - "NEVER suggest Principal: * without Condition keys"
  - "NEVER suggest removing bucket encryption"

## Phase 3 � Rollback

- Document the current state before making any changes
- If remediation worsens the issue, revert to the documented pre-change state
- Remove any temporary resources created during investigation
- Verify the system returns to its previous functional state after rollback
