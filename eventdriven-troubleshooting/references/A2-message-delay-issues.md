---
title: "A2 — Message Delay Issues"
description: "Diagnose and resolve SQS message delivery delay and latency issues"
status: active
severity: HIGH
triggers:
  - "message delay"
  - "SQS latency"
  - "delayed delivery"
  - "message not appearing"
  - "delivery delay"
owner: devops-agent
objective: "Identify why SQS messages are delayed and restore expected delivery timing"
context: "SQS messages can be delayed by queue-level delay (DelaySeconds, 0-900s), message-level delay timers, or consumer-side processing delays. Standard queues deliver messages almost immediately (unless delay is configured). FIFO queues may have additional latency due to ordering guarantees. Delays can also be caused by visibility timeout interactions, long polling configuration, or consumer scaling issues."
---

## Phase 1 — Triage

MUST:
- Check queue delay configuration: `aws sqs get-queue-attributes --queue-url <queue-url> --attribute-names DelaySeconds`
- Check queue metrics for message age: `aws cloudwatch get-metric-statistics --namespace AWS/SQS --metric-name ApproximateAgeOfOldestMessage --dimensions Name=QueueName,Value=<queue-name> --start-time <time> --end-time <time> --period 60 --statistics Maximum`
- Check messages in flight and available: `aws sqs get-queue-attributes --queue-url <queue-url> --attribute-names ApproximateNumberOfMessages,ApproximateNumberOfMessagesNotVisible,ApproximateNumberOfMessagesDelayed`
- Verify consumer is polling the queue: check consumer application logs
- Check visibility timeout: `aws sqs get-queue-attributes --queue-url <queue-url> --attribute-names VisibilityTimeout`

SHOULD:
- Check if messages are being sent with per-message delay timers
- Verify consumer long polling configuration: `aws sqs get-queue-attributes --queue-url <queue-url> --attribute-names ReceiveMessageWaitTimeSeconds`
- Review the number of consumers vs message volume
- Check CloudWatch NumberOfMessagesSent vs NumberOfMessagesReceived

MAY:
- Check if the producer is batching messages (introducing send-side delay)
- Review consumer processing time per message
- Verify no SQS service issues on the AWS Health Dashboard

## Phase 2 — Remediate

MUST:
- If queue delay is too high: reduce it: `aws sqs set-queue-attributes --queue-url <queue-url> --attributes DelaySeconds=0`
- If messages are delayed by per-message timers: update the producer to remove or reduce the delay
- If consumer is too slow: scale up consumers or optimize processing
- If visibility timeout is too long: reduce it to allow faster reprocessing of failed messages

SHOULD:
- Enable long polling (ReceiveMessageWaitTimeSeconds=20) to reduce empty receives and improve responsiveness
- Monitor ApproximateAgeOfOldestMessage to detect delays early
- Verify message throughput matches expected rates after changes

MAY:
- Implement auto-scaling for consumers based on queue depth
- Use short polling for latency-sensitive workloads
- Set up CloudWatch alarms for message age thresholds

## Common Issues

- symptoms: "Messages appear in queue but consumers don't receive them for several minutes"
  diagnosis: "Queue-level DelaySeconds is configured, delaying all messages."
  resolution: "Set DelaySeconds to 0 if delay is not intentional."

- symptoms: "Some messages are delayed but others are not"
  diagnosis: "Per-message delay timers set by the producer on specific messages."
  resolution: "Review producer code for SendMessage calls with DelaySeconds parameter."

- symptoms: "Messages pile up — ApproximateAgeOfOldestMessage increasing"
  diagnosis: "Consumers cannot keep up with message volume."
  resolution: "Scale up consumers. Increase batch size. Optimize processing time."

- symptoms: "Messages received but reappear after visibility timeout"
  diagnosis: "Consumer is not deleting messages after processing, or processing takes longer than visibility timeout."
  resolution: "Ensure consumers delete messages after successful processing. Increase visibility timeout if processing is legitimately slow."

## Output Format

```yaml
root_cause: "Message delay — <specific_cause>"
evidence:
  - type: queue_attributes
    content: "<queue delay and visibility timeout settings>"
  - type: queue_metrics
    content: "<ApproximateAgeOfOldestMessage and message counts>"
severity: HIGH
mitigation:
  immediate: "Adjust delay settings or scale consumers"
  long_term: "Implement consumer auto-scaling and message age monitoring"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (get-queue-attributes/metrics): GREEN — read-only"
  - "Phase 2 queue attribute changes: YELLOW — affects message delivery behavior"
  - "Phase 2 consumer scaling: YELLOW — creates new compute resources"

## Escalation Conditions

escalation_conditions:
  - "Message delays are causing SLA violations"
  - "Queue depth is growing unboundedly"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Delay is caused by an AWS service issue"
  - "Issue affects FIFO queue ordering guarantees"

## Data Sensitivity

data_sensitivity:
  - command: "get-queue-attributes"
    sensitivity: LOW
    contains: "Queue configuration settings"
  - command: "receive-message (for debugging)"
    sensitivity: HIGH
    contains: "Message body content — may contain sensitive data"
  - command: "CloudWatch metrics"
    sensitivity: LOW
    contains: "Aggregated queue metrics"

## Prohibited Actions

prohibited_actions:
  - "NEVER purge a production queue to resolve delay issues"
  - "NEVER set visibility timeout to 0 in production"
  - "NEVER read and discard messages to reduce queue depth"
  - "NEVER change FIFO queue settings without understanding ordering impact"

## Phase 3 — Rollback

- Before changing queue attributes, document current settings: `aws sqs get-queue-attributes --queue-url <queue-url> --attribute-names All`
- If delay changes cause issues, restore the previous DelaySeconds value
- If visibility timeout changes cause duplicate processing, revert to the previous value
- If consumer scaling causes issues, scale back to the previous count
- Monitor message age and processing rates after any changes
