---
title: "A1 — Messages Not Received"
description: "Diagnose and resolve SQS/SNS messages not being received by consumers or subscribers"
status: active
severity: CRITICAL
triggers:
  - "messages not received"
  - "consumer not receiving"
  - "empty receive"
  - "no messages"
  - "queue empty"
owner: devops-agent
objective: "Identify why messages are not being received and restore message delivery"
context: "Messages not being received can stem from multiple causes: messages not being sent, queue/topic misconfiguration, consumer polling issues, visibility timeout hiding messages, filter policies blocking delivery, permission errors, or encryption issues. This is often the first symptom reported and requires systematic triage to identify the actual failure point in the message pipeline."
---

## Phase 1 — Triage

MUST:
- Check if messages exist in the queue: `aws sqs get-queue-attributes --queue-url <queue-url> --attribute-names ApproximateNumberOfMessages,ApproximateNumberOfMessagesNotVisible,ApproximateNumberOfMessagesDelayed`
- Verify the producer is sending messages: check CloudWatch NumberOfMessagesSent metric
- Verify the consumer is polling: check CloudWatch NumberOfEmptyReceives metric
- Check if the consumer is using the correct queue URL
- Verify IAM permissions for the consumer: `aws iam simulate-principal-policy --policy-source-arn <arn> --action-names sqs:ReceiveMessage`

SHOULD:
- Check if messages are in the DLQ instead: `aws sqs get-queue-attributes --queue-url <dlq-url> --attribute-names ApproximateNumberOfMessages`
- Verify the queue is not purged or recently recreated (different queue URL)
- Check if the visibility timeout is hiding messages (high ApproximateNumberOfMessagesNotVisible)
- For SNS→SQS: verify the subscription is confirmed and filter policy allows the messages

MAY:
- Check VPC endpoint policies if the consumer is in a VPC
- Review if the consumer is using long polling vs short polling
- Check if the queue has a delay configured that is hiding new messages

## Phase 2 — Remediate

MUST:
- If messages not being sent: fix the producer — check producer logs and permissions
- If consumer using wrong queue URL: update the consumer configuration
- If permission denied: add sqs:ReceiveMessage permission to the consumer's IAM policy
- If messages in DLQ: fix the consumer processing issue and redrive messages (see B1 runbook)

SHOULD:
- Enable long polling to reduce empty receives: `aws sqs set-queue-attributes --queue-url <queue-url> --attributes ReceiveMessageWaitTimeSeconds=20`
- Verify message flow end-to-end after remediation
- Set up CloudWatch alarms for NumberOfMessagesSent and NumberOfMessagesReceived

MAY:
- Implement dead letter queue monitoring
- Set up end-to-end message tracing
- Create a health check that verifies the message pipeline

## Common Issues

- symptoms: "Consumer polls but always gets empty responses"
  diagnosis: "Queue is empty — messages not being sent, or sent to wrong queue."
  resolution: "Verify the producer is sending to the correct queue URL. Check producer logs."

- symptoms: "Messages exist in queue but consumer doesn't receive them"
  diagnosis: "Messages are not visible — either delayed or being processed by another consumer."
  resolution: "Check ApproximateNumberOfMessagesNotVisible and ApproximateNumberOfMessagesDelayed."

- symptoms: "SNS subscriber not receiving messages"
  diagnosis: "Subscription not confirmed, filter policy blocking, or delivery failure."
  resolution: "Check subscription status. Review filter policy. See D1 and D2 runbooks."

- symptoms: "Messages received but appear empty or malformed"
  diagnosis: "Message encoding issue or producer sending empty messages."
  resolution: "Check producer message format. Verify encoding (Base64 for binary). Check message attributes."

## Output Format

```yaml
root_cause: "Messages not received — <specific_cause>"
evidence:
  - type: queue_metrics
    content: "<message count and CloudWatch metrics>"
  - type: consumer_status
    content: "<consumer polling behavior and permissions>"
severity: CRITICAL
mitigation:
  immediate: "Fix the identified delivery or reception issue"
  long_term: "Implement end-to-end message monitoring and alerting"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (get-queue-attributes/metrics): GREEN — read-only"
  - "Phase 2 queue attribute changes: YELLOW — affects message delivery behavior"
  - "Phase 2 IAM policy changes: YELLOW — affects access permissions"

## Escalation Conditions

escalation_conditions:
  - "Complete message delivery failure affecting production systems"
  - "Messages are being lost (not in queue or DLQ)"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Issue involves cross-account or cross-region message delivery"
  - "Message loss has compliance or financial implications"

## Data Sensitivity

data_sensitivity:
  - command: "get-queue-attributes"
    sensitivity: LOW
    contains: "Queue configuration and message counts"
  - command: "receive-message (for debugging)"
    sensitivity: HIGH
    contains: "Message body content — may contain sensitive data"
  - command: "CloudWatch metrics"
    sensitivity: LOW
    contains: "Aggregated queue metrics"

## Prohibited Actions

prohibited_actions:
  - "NEVER purge a queue to troubleshoot delivery issues"
  - "NEVER read production messages without data handling authorization"
  - "NEVER delete the queue and recreate it to fix issues (messages will be lost)"
  - "NEVER change queue type (Standard to FIFO) — this requires recreation"

## Phase 3 — Rollback

- Before changing queue attributes, document current settings: `aws sqs get-queue-attributes --queue-url <queue-url> --attribute-names All`
- If attribute changes worsen the issue, restore previous values
- If consumer configuration was changed, revert if it causes new issues
- If IAM policy changes were too broad, scope them down after fixing the issue
- Monitor message delivery rates after any changes
