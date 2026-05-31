# Event-Driven Architecture Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any SQS, SNS, or EventBridge issue.

## Guardrail 1: SQS Visibility Timeout vs Message Retention Are Independent
VisibilityTimeout (default 30s, max 12h) controls how long a message is hidden after a receive. MessageRetentionPeriod (default 4 days, max 14 days) controls how long undeleted messages persist. Confusing these leads to misdiagnosis of message loss vs duplicate processing.

## Guardrail 2: FIFO Ordering Is Per MessageGroupId, Not Global
SQS FIFO queues guarantee ordering ONLY within a single MessageGroupId. Messages with different MessageGroupIds can be processed in parallel and arrive out of order relative to each other. Never claim FIFO provides total ordering.

## Guardrail 3: FIFO Deduplication Has a 5-Minute Window
MessageDeduplicationId deduplication applies within a 5-minute interval. After 5 minutes, the same deduplication ID can be reused. If ContentBasedDeduplication is disabled, you MUST provide MessageDeduplicationId or sends will fail.

## Guardrail 4: DLQ maxReceiveCount Increments on Every Receive
The receive count increments on EVERY ReceiveMessage call that returns the message, including failed processing attempts and visibility timeout expirations. A message moves to the DLQ after maxReceiveCount receives without deletion.

## Guardrail 5: SNS Filter Policies Require Correct FilterPolicyScope
SNS filter policies can match on MessageAttributes (default) or MessageBody. The FilterPolicyScope attribute must be set to "MessageBody" to filter on the message body. Mismatched scope causes filters to silently pass or reject all messages.

## Guardrail 6: SNS Delivery Failures Are Per-Subscription
If one SNS subscription fails delivery, it does NOT affect other subscriptions. Each subscription has independent delivery, retry, and DLQ settings. Never assume one failure blocks the entire fanout.

## Guardrail 7: EventBridge Patterns Are Case-Sensitive
EventBridge rule patterns perform case-sensitive matching on event field values. A pattern matching "EC2" will NOT match "ec2". Always verify the exact casing of event fields.

## Guardrail 8: EventBridge Event Size Limit Is 256 KB
Events larger than 256 KB are rejected by EventBridge PutEvents. The 256 KB limit includes the entire event entry (source, detail-type, detail, resources). Use the claim-check pattern with S3 for larger payloads.

## Guardrail 9: Cross-Account Event Delivery Requires Both Sides
Cross-account EventBridge delivery requires the target event bus policy to allow PutEvents from the source account AND the source account IAM policy to allow events:PutEvents on the target bus ARN. Missing either side causes delivery failure.

## Guardrail 10: EventBridge Delivers At-Least-Once
EventBridge provides at-least-once delivery. Duplicate events are possible. Target applications must be idempotent. Never claim EventBridge guarantees exactly-once delivery.

## Guardrail 11: SNS Raw Message Delivery Changes the Payload Format
When RawMessageDelivery is enabled on an SQS or HTTP/S subscription, SNS sends the raw message without the SNS JSON envelope. Consumers expecting the SNS wrapper will break. Only SQS and HTTP/S subscriptions support raw delivery.

## Guardrail 12: SQS Standard Queues Allow Duplicates and Reordering
Standard SQS queues provide at-least-once delivery. Messages may be delivered more than once and in a different order than sent. Only FIFO queues guarantee exactly-once processing and ordering (per MessageGroupId).
