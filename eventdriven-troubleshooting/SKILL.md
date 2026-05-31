---
name: eventdriven-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon SQS, SNS, and
  EventBridge problems by analyzing queue configurations, topic policies,
  event rules, delivery failures, and following structured runbooks.
  Activate when: messages not received, visibility timeout issues, DLQ
  overflow, FIFO ordering problems, deduplication failures, SNS delivery
  failures, subscription filtering issues, EventBridge rule pattern
  matching failures, target invocation errors, scheduling problems,
  Pipes issues, archive/replay failures, cross-account event delivery,
  Lambda trigger failures, SQS-SNS fanout problems, access policy
  errors, encryption issues, or the user says something is wrong with
  SQS, SNS, or EventBridge without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with SQS, SNS, EventBridge, IAM, KMS,
  CloudTrail, CloudWatch, and optionally Lambda permissions.
---

# Event-Driven Architecture Diagnostics

## When to use

Any SQS, SNS, or EventBridge investigation where the console alone is insufficient — message delivery failures, DLQ analysis, FIFO ordering issues, SNS fanout problems, EventBridge rule debugging, cross-account event delivery, encryption errors, or integration troubleshooting.

## Investigation workflow

### Step 1 — Collect and triage

#### SQS
```
aws sqs get-queue-attributes --queue-url <queue-url> --attribute-names All
aws sqs get-queue-url --queue-name <queue-name>
aws sqs list-dead-letter-source-queues --queue-url <dlq-url>
aws sqs get-queue-attributes --queue-url <queue-url> --attribute-names QueueArn,Policy,RedrivePolicy,VisibilityTimeout,MessageRetentionPeriod
```

#### SNS
```
aws sns get-topic-attributes --topic-arn <topic-arn>
aws sns list-subscriptions-by-topic --topic-arn <topic-arn>
aws sns get-subscription-attributes --subscription-arn <subscription-arn>
```

#### EventBridge
```
aws events list-rules --event-bus-name <bus-name>
aws events describe-rule --name <rule-name> --event-bus-name <bus-name>
aws events list-targets-by-rule --rule <rule-name> --event-bus-name <bus-name>
aws events describe-event-bus --name <bus-name>
```

### Step 2 — Domain deep dive

```
aws cloudwatch get-metric-statistics --namespace AWS/SQS --metric-name NumberOfMessagesSent --dimensions Name=QueueName,Value=<queue> --start-time <start> --end-time <end> --period 300 --statistics Sum
aws cloudwatch get-metric-statistics --namespace AWS/SQS --metric-name ApproximateNumberOfMessagesVisible --dimensions Name=QueueName,Value=<queue> --start-time <start> --end-time <end> --period 300 --statistics Average
aws cloudwatch get-metric-statistics --namespace AWS/SNS --metric-name NumberOfNotificationsFailed --dimensions Name=TopicName,Value=<topic> --start-time <start> --end-time <end> --period 300 --statistics Sum
aws cloudwatch get-metric-statistics --namespace AWS/Events --metric-name FailedInvocations --dimensions Name=RuleName,Value=<rule> --start-time <start> --end-time <end> --period 300 --statistics Sum
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=sqs.amazonaws.com --max-results 10
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=sns.amazonaws.com --max-results 10
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=events.amazonaws.com --max-results 10
```

Read `references/eventdriven-guardrails.md` before concluding on any event-driven issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `sqs get-queue-attributes` | Queue configuration, policy, DLQ settings |
| `sqs list-dead-letter-source-queues` | Identify which queues feed a DLQ |
| `sqs receive-message` | Test message retrieval, inspect message attributes |
| `sqs purge-queue` | Clear all messages from a queue |
| `sns get-topic-attributes` | Topic policy, encryption, delivery settings |
| `sns list-subscriptions-by-topic` | All subscriptions for a topic |
| `sns get-subscription-attributes` | Filter policy, delivery policy, raw delivery |
| `events list-rules` | All rules on an event bus |
| `events describe-rule` | Rule pattern, state, schedule |
| `events list-targets-by-rule` | Targets attached to a rule |
| `events describe-event-bus` | Event bus policy |
| `events test-event-pattern` | Test if an event matches a rule pattern |
| `events list-archives` | EventBridge archives |
| `pipes list-pipes` | EventBridge Pipes |

## Gotchas: SQS / SNS / EventBridge

### SQS
- Visibility timeout vs message retention: VisibilityTimeout (default 30s, max 12h) controls how long a message is hidden after receive. MessageRetentionPeriod (default 4 days, max 14 days) controls how long undeleted messages are kept. If retention expires, messages are permanently lost.
- FIFO ordering is per MessageGroupId, NOT global. Messages with different MessageGroupIds can be processed in parallel and out of order relative to each other.
- FIFO deduplication: if ContentBasedDeduplication is disabled, you MUST provide MessageDeduplicationId. The deduplication interval is 5 minutes — identical IDs within 5 minutes are treated as duplicates.
- DLQ redrive policy: maxReceiveCount determines how many times a message can be received before being sent to the DLQ. Setting it too low causes premature DLQ routing; too high delays error detection.
- Long polling (WaitTimeSeconds > 0, max 20s) reduces empty responses and cost. Short polling (WaitTimeSeconds = 0) returns immediately, potentially with empty responses. Set at queue level or per ReceiveMessage call.
- SQS max receive count before DLQ: after maxReceiveCount receives without deletion, the message moves to the DLQ. The receive count increments on EVERY receive, including failed processing attempts.
- Standard queues guarantee at-least-once delivery but messages may be delivered out of order or duplicated. FIFO queues guarantee exactly-once processing and ordering within a MessageGroupId.
- SQS message size limit is 256 KB. For larger payloads, use the Extended Client Library with S3.
- Batch operations (SendMessageBatch, DeleteMessageBatch, ChangeMessageVisibilityBatch) support up to 10 messages per call. Total batch payload cannot exceed 256 KB.

### SNS
- SNS message filtering: filter policies on subscriptions reduce downstream processing. Filters can match on message attributes or message body. An empty filter policy accepts ALL messages.
- SNS fanout delivery failures: if one subscription fails, it does NOT affect other subscriptions. Each subscription has independent delivery and retry.
- SNS raw message delivery: when enabled, SNS sends the raw message without the SNS JSON wrapper. Required for SQS subscriptions that need the original message format. Only supported for SQS and HTTP/S endpoints.
- SNS delivery retries per protocol: HTTP/S gets 3 retries with backoff (total ~1 hour). SQS and Lambda get no retries (delivery is synchronous and reliable). Email/SMS have limited retries. Each protocol has its own delivery policy.
- SNS message size limit is 256 KB. For larger payloads, use the Extended Client Library with S3.
- Cross-account SNS: the topic policy must allow the external account to Subscribe and/or Publish. The subscriber's SQS queue policy must also allow SNS to SendMessage.

### EventBridge
- EventBridge rule pattern matching uses content-based filtering. Patterns match on event fields using exact match, prefix, numeric comparison, exists, and anything-but. Patterns are case-sensitive and must match the event structure exactly.
- EventBridge 256 KB event size limit: events larger than 256 KB are rejected. Use claim-check pattern — store payload in S3 and pass the reference in the event.
- EventBridge archive and replay: archives store events matching a pattern. Replay re-sends archived events to the same event bus. Replayed events have the original event time, not replay time.
- EventBridge Pipes vs Rules: Pipes provide point-to-point integration with optional filtering, enrichment, and transformation. Rules provide event routing to multiple targets with pattern matching. Pipes support batching from SQS/Kinesis/DynamoDB sources.
- Cross-account event delivery requires the target event bus policy to allow PutEvents from the source account. The source account needs events:PutEvents permission on the target bus ARN.
- EventBridge delivers events with at-least-once delivery. Duplicate events are possible. Design targets to be idempotent.
- EventBridge has a soft limit of 300 rules per event bus (adjustable). Each rule can have up to 5 targets.
- Scheduled rules use cron or rate expressions. Cron expressions in EventBridge use 6 fields (no seconds). Rate expressions support minutes, hours, and days.

## Anti-hallucination rules

1. Always cite specific queue attributes, topic policies, rule patterns, or API responses as evidence.
2. SQS FIFO ordering is per MessageGroupId, NOT global. Never claim FIFO guarantees total ordering across all messages.
3. EventBridge event size limit is 256 KB. Never claim larger events can be sent directly.
4. SNS filter policies can match on message attributes OR message body, but the FilterPolicyScope must be set correctly. Never assume default scope.
5. SQS visibility timeout and message retention are independent settings. Never conflate them.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 36 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — SQS Delivery | A1-A4 | Messages not received, visibility timeout, DLQ overflow, message loss |
| B — SQS FIFO | B1-B3 | Ordering issues, deduplication, throughput limits |
| C — SQS Performance | C1-C3 | Throttling, long polling, batch operations |
| D — SNS Delivery | D1-D4 | Subscription failures, delivery failures, filtering issues, cross-account |
| E — SNS Configuration | E1-E3 | Topic policy, encryption, raw message delivery |
| F — EventBridge Rules | F1-F4 | Pattern matching, target failures, scheduling, cross-account |
| G — EventBridge Advanced | G1-G3 | Pipes issues, archive/replay, schema registry |
| H — Integration | H1-H3 | Lambda triggers, SQS-SNS fanout, EventBridge-to-SQS/SNS |
| I — Security | I1-I2 | Access policies, encryption (SSE-SQS/SSE-KMS) |
| Z — Catch-All | Z1 | General troubleshooting |
