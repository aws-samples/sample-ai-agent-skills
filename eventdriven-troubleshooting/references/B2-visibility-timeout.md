---
title: "B2 — Visibility Timeout"
description: "Diagnose and resolve SQS visibility timeout configuration and duplicate processing issues"
status: active
severity: MEDIUM
triggers:
  - "visibility timeout"
  - "duplicate messages"
  - "message reappears"
  - "message processing timeout"
  - "ChangeMessageVisibility"
owner: devops-agent
objective: "Resolve visibility timeout issues to prevent duplicate processing and message loss"
context: "SQS visibility timeout determines how long a message is hidden from other consumers after being received. Default is 30 seconds, maximum is 12 hours. If a consumer does not delete the message before the visibility timeout expires, the message becomes visible again and can be received by another consumer, causing duplicate processing. Consumers can extend the timeout using ChangeMessageVisibility."
---

## Phase 1 — Triage

MUST:
- Check queue visibility timeout: `aws sqs get-queue-attributes --queue-url <queue-url> --attribute-names VisibilityTimeout`
- Check messages not visible (in-flight): `aws sqs get-queue-attributes --queue-url <queue-url> --attribute-names ApproximateNumberOfMessagesNotVisible`
- Review consumer processing time — is it exceeding the visibility timeout?
- Check CloudWatch metric NumberOfMessagesReceived vs NumberOfMessagesDeleted (gap indicates reprocessing)
- Review consumer logs for duplicate processing evidence

SHOULD:
- Check if consumers are using ChangeMessageVisibility to extend the timeout
- Verify the consumer's message deletion logic (is it deleting after successful processing?)
- Check the ApproximateReceiveCount attribute on messages (>1 indicates redelivery)

MAY:
- Profile consumer processing time distribution
- Check if specific message types take longer to process
- Review if batch processing is causing timeout issues

## Phase 2 — Remediate

MUST:
- If processing exceeds timeout: increase visibility timeout: `aws sqs set-queue-attributes --queue-url <queue-url> --attributes VisibilityTimeout=<seconds>`
- If consumer not deleting messages: fix the consumer to delete messages after successful processing
- If variable processing time: implement ChangeMessageVisibility in the consumer to extend timeout dynamically
- If duplicate processing causes issues: implement idempotency in the consumer

SHOULD:
- Set visibility timeout to at least 6x the average processing time
- Implement a heartbeat pattern: periodically extend visibility timeout during long processing
- Add message deduplication logic in the consumer for at-least-once delivery

MAY:
- Use FIFO queues with exactly-once processing for critical workloads
- Implement a processing timeout in the consumer that is shorter than the visibility timeout
- Set up monitoring for ApproximateReceiveCount to detect redelivery patterns

## Common Issues

- symptoms: "Messages processed multiple times"
  diagnosis: "Consumer processing time exceeds visibility timeout, causing message redelivery."
  resolution: "Increase visibility timeout or implement ChangeMessageVisibility heartbeat."

- symptoms: "Messages disappear and reappear"
  diagnosis: "Consumer receives the message (it becomes invisible) but fails to process and delete it."
  resolution: "Fix consumer error handling. Ensure messages are deleted after successful processing."

- symptoms: "High ApproximateNumberOfMessagesNotVisible but low processing"
  diagnosis: "Consumers are receiving messages but not processing or deleting them (consumer stalled)."
  resolution: "Check consumer health. Restart stalled consumers. Messages will become visible after timeout."

- symptoms: "ChangeMessageVisibility returns error"
  diagnosis: "Receipt handle expired or message was already deleted."
  resolution: "Use the most recent receipt handle. Implement error handling for expired handles."

## Output Format

```yaml
root_cause: "Visibility timeout issue — <specific_cause>"
evidence:
  - type: queue_attributes
    content: "<visibility timeout and message counts>"
  - type: processing_metrics
    content: "<consumer processing time vs timeout>"
severity: MEDIUM
mitigation:
  immediate: "Adjust visibility timeout or fix consumer processing"
  long_term: "Implement idempotency, heartbeat pattern, and processing monitoring"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (get-queue-attributes/metrics): GREEN — read-only"
  - "Phase 2 visibility timeout changes: YELLOW — affects message redelivery behavior"
  - "Phase 2 consumer code changes: YELLOW — affects message processing logic"

## Escalation Conditions

escalation_conditions:
  - "Duplicate processing is causing data corruption or financial impact"
  - "Visibility timeout changes affect multiple consumers"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Message redelivery is causing cascading failures"
  - "Issue affects FIFO queue exactly-once processing guarantees"

## Data Sensitivity

data_sensitivity:
  - command: "get-queue-attributes"
    sensitivity: LOW
    contains: "Queue configuration settings"
  - command: "receive-message"
    sensitivity: HIGH
    contains: "Message body content"
  - command: "consumer logs"
    sensitivity: MEDIUM
    contains: "Processing details and error messages"

## Prohibited Actions

prohibited_actions:
  - "NEVER set visibility timeout to 0 in production (causes immediate redelivery)"
  - "NEVER set visibility timeout to 12 hours without understanding the impact on failed message reprocessing"
  - "NEVER ignore duplicate processing — implement idempotency"
  - "NEVER change visibility timeout during peak processing without a rollback plan"

## Phase 3 — Rollback

- Before changing visibility timeout, document the current value
- If timeout increase causes messages to be hidden too long after failures, reduce it
- If timeout decrease causes duplicate processing, increase it back
- If consumer code changes introduce bugs, roll back the consumer deployment
- Monitor duplicate processing rates and message age after any changes
