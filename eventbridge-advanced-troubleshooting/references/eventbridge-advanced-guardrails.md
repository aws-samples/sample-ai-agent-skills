# Amazon EventBridge Advanced Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any EventBridge Scheduler, Pipes, or advanced feature issue.

## Guardrail 1: Scheduler Is NOT EventBridge Rules
EventBridge Scheduler is a separate service from EventBridge rules. Scheduler uses the scheduler API namespace (aws scheduler), not the events API. Schedules have their own IAM execution roles, delivery configurations, and retry policies. Do not confuse schedule configuration with rule configuration. Scheduler supports features rules don't have (flexible time windows, one-time schedules).

## Guardrail 2: Pipes Are Point-to-Point — No Event Bus Involved
EventBridge Pipes connect a source directly to a target without going through an event bus. Pipes do NOT use rules for routing. Each pipe has exactly one source and one target with optional filtering and enrichment. Pipes have their own IAM execution role. Do not suggest configuring event bus rules for pipe-based integrations.

## Guardrail 3: Pipe Sources Are Limited to Five Services
EventBridge Pipe sources are limited to: Amazon SQS, Amazon Kinesis Data Streams, Amazon DynamoDB Streams, Amazon MSK, and Amazon MQ. You cannot use S3, SNS, CloudWatch Events, or other services as pipe sources. For other event sources, use EventBridge rules with event buses instead.

## Guardrail 4: Enrichment Is Optional and Adds Latency
Pipe enrichment (Lambda, Step Functions, API Gateway, API Destination) is optional. When configured, it processes events between filtering and target delivery. Enrichment adds latency and cost to the pipe. Enrichment failures are handled by the pipe's retry policy and DLQ. The enrichment function receives filtered events and must return the transformed events.

## Guardrail 5: Archive Replay Delivers to the SAME Event Bus
When replaying archived events, they are delivered to the same event bus they were originally archived from. You cannot replay events to a different event bus. All rules on the bus will process replayed events. Replayed events include a replay-name header that can be used for filtering. Plan for duplicate processing if rules don't filter replayed events.

## Guardrail 6: Schema Discovery Has Cost Implications
Schema discovery must be explicitly enabled on an event bus. It analyzes events flowing through the bus and creates schemas in the discovered-schemas registry. Schema discovery incurs costs per event analyzed ($0.10 per million events). It adds minimal latency. Disable discovery when not actively developing to avoid unnecessary costs.

## Guardrail 7: Cross-Account Event Bus Requires Resource Policy
Sending events to an event bus in another account requires a resource policy on the destination bus granting events:PutEvents to the source account. The source account must also have IAM permissions to call events:PutEvents on the destination bus ARN. Both accounts must be in the same region (or use global endpoints for cross-region).

## Guardrail 8: DLQ Must Be Configured Per-Target
Dead letter queues are configured per-target (for rules) or per-pipe (for Pipes). There is no global DLQ for an event bus. Each target or pipe can have its own SQS queue as a DLQ. The DLQ receives events that fail delivery after all retries are exhausted. The IAM role must have sqs:SendMessage permission on the DLQ.

## Guardrail 9: One-Time Schedules Are Auto-Deleted by Default
EventBridge Scheduler one-time schedules (at expressions) execute once and are automatically deleted after execution by default. This behavior can be changed by setting ActionAfterCompletion to NONE (keep) or DELETE (default). If set to NONE, the schedule remains in COMPLETE state but does not execute again.

## Guardrail 10: Flexible Time Windows Are Not Exact Execution Times
Scheduler flexible time windows allow execution within a window (1-15 minutes or OFF) rather than at the exact scheduled time. This helps distribute load and improve reliability. With a flexible window, the schedule executes at a random time within the window after the scheduled time. Use OFF for exact timing requirements.

## Guardrail 11: Pipe Batching Depends on Source Type
Pipe batch size and behavior depend on the source type. SQS supports batch sizes 1-10000. Kinesis and DynamoDB Streams support batch sizes 1-10000 with configurable parallelization. MSK supports batch sizes 1-10000. Batch window (MaximumBatchingWindowInSeconds) controls how long to wait for a full batch. Partial batches are delivered when the window expires.

## Guardrail 12: Partner Event Sources Have a Lifecycle
Partner event sources (SaaS integrations) go through a lifecycle: pending → active → deleted. A partner event source must be associated with an event bus before it becomes active. If the partner event source is deleted, the associated event bus stops receiving events but is not automatically deleted. Partner event sources are region-specific.
