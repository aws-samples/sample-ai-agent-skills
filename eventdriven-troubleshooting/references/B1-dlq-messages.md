---
title: "B1 — DLQ Messages"
description: "Diagnose and resolve issues with messages landing in Dead Letter Queues"
status: active
severity: HIGH
triggers:
  - "dead letter queue"
  - "DLQ messages"
  - "maxReceiveCount"
  - "failed messages"
  - "poison pill"
owner: devops-agent
objective: "Identify why messages are being sent to the DLQ and resolve the underlying processing failures"
context: "A Dead Letter Queue (DLQ) receives messages that cannot be successfully processed after a configured number of attempts (maxReceiveCount). Messages land in the DLQ when consumers fail to delete them within the visibility timeout, repeatedly. Common causes: message format errors, downstream dependency failures, consumer bugs, or poison pill messages. SQS, SNS, and EventBridge all support DLQ configurations."
---

## Phase 1 — Triage

MUST:
- Check DLQ message count: `aws sqs get-queue-attributes --queue-url <dlq-url> --attribute-names ApproximateNumberOfMessages`
- Check the redrive policy on the source queue: `aws sqs get-queue-attributes --queue-url <source-queue-url> --attribute-names RedrivePolicy`
- Sample messages from the DLQ to identify patterns: `aws sqs receive-message --queue-url <dlq-url> --max-number-of-messages 5 --attribute-names All --message-attribute-names All`
- Check the source queue's maxReceiveCount in the redrive policy
- Review consumer application logs for processing errors

SHOULD:
- Check CloudWatch metrics for the source queue: NumberOfMessagesReceived vs NumberOfMessagesDeleted
- Verify the consumer is properly deleting messages after successful processing
- Check if the DLQ has its own redrive policy (DLQ of DLQ)
- Review the message attributes for error context

MAY:
- Check if the DLQ messages correlate with specific time periods or events
- Review the consumer's error handling and retry logic
- Check if downstream dependencies were unavailable during the failure period

## Phase 2 — Remediate

MUST:
- If consumer bug: fix the consumer code and redeploy
- If downstream dependency failure: verify the dependency is restored, then redrive messages
- If poison pill messages: identify and remove the malformed messages from the DLQ
- If maxReceiveCount too low: increase it to allow more retries: update the RedrivePolicy on the source queue

SHOULD:
- Use SQS DLQ redrive to move messages back to the source queue: `aws sqs start-message-move-task --source-arn <dlq-arn> --destination-arn <source-queue-arn>`
- Implement better error handling in the consumer to distinguish transient vs permanent failures
- Set up CloudWatch alarms for DLQ message count

MAY:
- Implement a DLQ processor that logs failed messages for analysis
- Set up automated redrive with backoff for transient failures
- Create a dashboard for DLQ monitoring across all queues

## Common Issues

- symptoms: "Messages appearing in DLQ shortly after being sent"
  diagnosis: "maxReceiveCount is set to 1 — messages go to DLQ after a single failure."
  resolution: "Increase maxReceiveCount to allow retries (typically 3-5)."

- symptoms: "All messages going to DLQ"
  diagnosis: "Consumer is crashing or not deleting messages, causing all messages to exceed maxReceiveCount."
  resolution: "Fix the consumer. Check for unhandled exceptions. Verify the consumer deletes messages after processing."

- symptoms: "Specific message types consistently go to DLQ"
  diagnosis: "Poison pill — certain message formats cause consumer failures."
  resolution: "Add message validation in the consumer. Handle malformed messages gracefully."

- symptoms: "DLQ messages have no useful error information"
  diagnosis: "Consumer does not log errors before message goes to DLQ."
  resolution: "Implement error logging in the consumer. Consider adding error attributes to messages before they reach DLQ."

## Output Format

```yaml
root_cause: "DLQ messages — <specific_cause>"
evidence:
  - type: dlq_messages
    content: "<sample DLQ message attributes and patterns>"
  - type: redrive_policy
    content: "<source queue redrive policy>"
severity: HIGH
mitigation:
  immediate: "Fix consumer, redrive messages, or remove poison pills"
  long_term: "Implement DLQ monitoring, error handling, and automated redrive"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (get-queue-attributes/receive-message): GREEN — read-only (receive with visibility timeout)"
  - "Phase 2 message redrive: YELLOW — reprocesses failed messages"
  - "Phase 2 DLQ purge: RED — permanently deletes failed messages"

## Escalation Conditions

escalation_conditions:
  - "DLQ contains messages with sensitive data that needs special handling"
  - "Message redrive could cause duplicate processing in production"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "DLQ message volume is growing rapidly"
  - "Failed messages represent financial transactions or critical data"

## Data Sensitivity

data_sensitivity:
  - command: "receive-message from DLQ"
    sensitivity: HIGH
    contains: "Message body content — may contain sensitive business data"
  - command: "get-queue-attributes"
    sensitivity: LOW
    contains: "Queue configuration and redrive policy"
  - command: "consumer logs"
    sensitivity: MEDIUM
    contains: "Processing errors and message details"

## Prohibited Actions

prohibited_actions:
  - "NEVER purge a DLQ without analyzing the messages first"
  - "NEVER redrive messages without fixing the underlying consumer issue"
  - "NEVER share DLQ message contents without verifying data sensitivity"
  - "NEVER remove the DLQ configuration from a production queue"
  - "NEVER set maxReceiveCount higher than 10 without understanding the retry impact"

## Phase 3 — Rollback

- Before redriving messages, document the DLQ message count and sample messages
- If redrive causes processing issues, stop the consumer and investigate
- If maxReceiveCount was changed, revert if it causes too many retries
- If consumer code was updated, roll back if the new version introduces new failures
- Monitor both the source queue and DLQ after any changes
