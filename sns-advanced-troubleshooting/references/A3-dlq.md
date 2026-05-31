---
title: "A3 — Dead-Letter Queue (DLQ) Issues"
description: "Diagnose SNS dead-letter queue configuration and message handling"
status: active
severity: MEDIUM
triggers:
  - "dead-letter queue"
  - "DLQ"
  - "undeliverable messages"
  - "redrive policy"
owner: devops-agent
objective: "Configure and troubleshoot SNS dead-letter queues for failed deliveries"
context: "SNS DLQs capture messages that cannot be delivered after all retries are exhausted. DLQs are configured per-subscription (not per-topic) using the RedrivePolicy attribute. The DLQ must be an SQS queue."
---

## Phase 1 — Triage

MUST:
- Check subscription redrive policy: `aws sns get-subscription-attributes --subscription-arn <sub-arn> --query 'Attributes.RedrivePolicy'`
- Check DLQ message count: `aws sqs get-queue-attributes --queue-url <dlq-url> --attribute-names ApproximateNumberOfMessages`
- Verify DLQ SQS queue exists and is accessible
- Check DLQ SQS queue policy allows SNS to send messages

SHOULD:
- Read DLQ messages for failure details: `aws sqs receive-message --queue-url <dlq-url> --max-number-of-messages 5`
- Verify the DLQ is in the same Region as the subscription
- Check for DLQ SQS queue encryption (KMS key must allow SNS)

MAY:
- Set up CloudWatch alarms on DLQ message count
- Implement DLQ message processing/redriving

## Phase 2 — Remediate

MUST:
- If no DLQ configured: set RedrivePolicy on the subscription: `aws sns set-subscription-attributes --subscription-arn <sub-arn> --attribute-name RedrivePolicy --attribute-value '{"deadLetterTargetArn":"<sqs-arn>"}'`
- If DLQ permission denied: add SNS send permission to SQS queue policy
- If DLQ encrypted: add KMS permissions for SNS service principal

SHOULD:
- Configure DLQ on all subscriptions for production topics
- Set up monitoring on DLQ depth
- Implement DLQ message processing pipeline

MAY:
- Use SQS DLQ redrive to replay failed messages
- Implement automated alerting on DLQ messages

## Common Issues

- symptoms: "DLQ configured but no messages appearing"
  diagnosis: "SQS queue policy doesn't allow SNS to send messages."
  resolution: "Add sqs:SendMessage permission for sns.amazonaws.com in the queue policy."

- symptoms: "Messages in DLQ but no failure details"
  diagnosis: "DLQ messages contain the original SNS message. Enable delivery status logging for failure reasons."
  resolution: "Enable delivery status logging. DLQ messages show what failed, logs show why."

## Output Format

```yaml
root_cause: "dlq — <specific_cause>"
evidence:
  - type: redrive_policy
    content: "<subscription DLQ configuration>"
  - type: dlq_messages
    content: "<DLQ message count and samples>"
severity: MEDIUM
mitigation:
  immediate: "Fix DLQ configuration or permissions"
  long_term: "Configure DLQ on all subscriptions with monitoring"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Message payloads may contain sensitive application data
- Subscription endpoints reveal integration architecture
- SMS messages may contain customer PII or authentication codes
- Classification: CONFIDENTIAL - message content and subscriber endpoint data

## Prohibited Actions

- NEVER delete SNS topics with active subscriptions in production
- NEVER modify filter policies on production subscriptions without testing
- NEVER change access policies that could disrupt cross-account message delivery
- NEVER purge dead-letter queues without analyzing failed messages

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
