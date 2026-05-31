---
title: "C1 — FIFO Ordering"
description: "Diagnose and resolve SQS FIFO queue message ordering issues"
status: active
severity: HIGH
triggers:
  - "FIFO ordering"
  - "message order"
  - "MessageGroupId"
  - "out of order"
  - "FIFO throughput"
owner: devops-agent
objective: "Resolve FIFO queue ordering issues and ensure messages are processed in the correct sequence"
context: "SQS FIFO queues guarantee message ordering within a MessageGroupId. Messages with the same MessageGroupId are delivered in the order they were sent. Different MessageGroupIds can be processed in parallel. FIFO queues have a throughput limit of 300 messages/second (3000 with high throughput mode). Ordering issues arise from incorrect MessageGroupId usage, consumer parallelism conflicts, or visibility timeout interactions."
---

## Phase 1 — Triage

MUST:
- Verify the queue is FIFO: `aws sqs get-queue-attributes --queue-url <queue-url> --attribute-names FifoQueue`
- Check if high throughput mode is enabled: `aws sqs get-queue-attributes --queue-url <queue-url> --attribute-names DeduplicationScope,FifoThroughputLimit`
- Review how the producer sets MessageGroupId — is it consistent for related messages?
- Check the number of consumers vs message groups
- Verify messages are being processed in order by reviewing consumer logs

SHOULD:
- Check if multiple consumers are processing the same MessageGroupId (breaks ordering)
- Verify the visibility timeout is long enough for sequential processing within a group
- Review CloudWatch metrics: NumberOfMessagesSent, NumberOfMessagesReceived
- Check if messages are being received with ReceiveMessage MaxNumberOfMessages > 1

MAY:
- Analyze MessageGroupId distribution for hot partitions
- Check if the producer is sending messages to the correct MessageGroupId
- Review if batch sending preserves the intended order

## Phase 2 — Remediate

MUST:
- If ordering broken by parallel consumers on same group: ensure only one consumer processes each MessageGroupId at a time
- If MessageGroupId is incorrect: fix the producer to use consistent group IDs for related messages
- If throughput limited: enable high throughput mode: `aws sqs set-queue-attributes --queue-url <queue-url> --attributes FifoThroughputLimit=perMessageGroupId,DeduplicationScope=messageGroup`
- If visibility timeout too short: increase it to prevent redelivery within a group

SHOULD:
- Use distinct MessageGroupIds for independent message streams
- Implement single-consumer-per-group pattern for strict ordering
- Monitor per-group processing to detect ordering violations

MAY:
- Implement a message sequencing check in the consumer
- Use high throughput FIFO for workloads with many message groups
- Document the MessageGroupId strategy for the application

## Common Issues

- symptoms: "Messages processed out of order"
  diagnosis: "Multiple consumers processing the same MessageGroupId simultaneously."
  resolution: "Ensure only one consumer processes each MessageGroupId. Use a consumer-group mapping."

- symptoms: "FIFO queue throughput is limited to 300 msg/s"
  diagnosis: "Standard FIFO throughput limit. High throughput mode not enabled."
  resolution: "Enable high throughput mode with per-message-group throughput limit."

- symptoms: "Messages from different groups appear interleaved"
  diagnosis: "This is expected — FIFO only guarantees ordering within a MessageGroupId."
  resolution: "If global ordering is needed, use a single MessageGroupId (limits throughput to 300 msg/s)."

- symptoms: "Message group is blocked — no messages delivered"
  diagnosis: "A message in the group has exceeded visibility timeout and is being redelivered, blocking subsequent messages."
  resolution: "Process or delete the blocking message. Increase visibility timeout."

## Output Format

```yaml
root_cause: "FIFO ordering issue — <specific_cause>"
evidence:
  - type: queue_config
    content: "<FIFO queue attributes and throughput settings>"
  - type: message_groups
    content: "<MessageGroupId usage patterns>"
severity: HIGH
mitigation:
  immediate: "Fix MessageGroupId usage or consumer parallelism"
  long_term: "Implement proper message group strategy and ordering verification"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (get-queue-attributes/metrics): GREEN — read-only"
  - "Phase 2 queue attribute changes: YELLOW — affects throughput and deduplication behavior"
  - "Phase 2 consumer architecture changes: YELLOW — affects processing patterns"

## Escalation Conditions

escalation_conditions:
  - "Message ordering violations are causing data inconsistency"
  - "FIFO throughput limits are blocking production message flow"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Ordering fix requires architectural changes to the consumer"
  - "Issue affects financial transactions or audit-critical message sequences"

## Data Sensitivity

data_sensitivity:
  - command: "get-queue-attributes"
    sensitivity: LOW
    contains: "Queue configuration settings"
  - command: "receive-message"
    sensitivity: HIGH
    contains: "Message body and MessageGroupId"
  - command: "producer logs"
    sensitivity: MEDIUM
    contains: "Message sending patterns and group IDs"

## Prohibited Actions

prohibited_actions:
  - "NEVER change a FIFO queue to a standard queue (not supported — must recreate)"
  - "NEVER use a single MessageGroupId for all messages unless global ordering is required"
  - "NEVER ignore ordering violations in FIFO queues"
  - "NEVER purge a FIFO queue without understanding the impact on in-flight message groups"

## Phase 3 — Rollback

- Before changing FIFO queue attributes, document current settings
- If high throughput mode causes unexpected behavior, revert to standard FIFO throughput
- If consumer changes break ordering, roll back the consumer deployment
- If MessageGroupId strategy was changed, coordinate with producers to revert
- Monitor message ordering and throughput after any changes
