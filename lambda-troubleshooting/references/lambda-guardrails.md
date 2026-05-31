# Lambda Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Lambda issue.

## Guardrail 1: Memory Controls CPU Proportionally
Lambda allocates CPU proportional to memory. 1769 MB = 1 full vCPU. At 128 MB you get ~7% of a vCPU. Never claim CPU is independent of memory. If a function is CPU-bound, increasing memory increases CPU.

## Guardrail 2: Reserved Concurrency is a CAP, Not a Guarantee
Reserved concurrency limits the maximum concurrent executions AND removes those units from the unreserved pool. It does NOT pre-warm environments. Setting reserved concurrency to 5 means max 5 concurrent executions, not 5 warm instances.

## Guardrail 3: VPC Lambda Needs NAT for Internet
Lambda functions in a VPC do NOT get public IPs. They need a NAT gateway (or NAT instance) in a public subnet for internet access. VPC endpoints can provide access to AWS services without NAT. Never claim VPC Lambda has internet by default.

## Guardrail 4: /tmp is Ephemeral Across Cold Starts
/tmp persists across warm invocations of the SAME execution environment. It is wiped on cold starts. Max 10 GB (configurable). Never claim /tmp is reliable persistent storage.

## Guardrail 5: Timeout Max is 900 Seconds
Lambda maximum timeout is 15 minutes (900 seconds). If a workload needs longer, Lambda is the wrong service. Consider Step Functions, ECS, or EC2.

## Guardrail 6: Synchronous vs Asynchronous Retry Behavior
Synchronous invocations do NOT retry — the error is returned to the caller. Asynchronous invocations retry twice (configurable) then send to DLQ/OnFailure destination. Event source mappings have their own retry semantics. Never apply one invocation type's retry behavior to another.

## Guardrail 7: Cold Start is NOT Configurable (Except Provisioned Concurrency)
You cannot eliminate cold starts without Provisioned Concurrency. Reducing package size, using lighter runtimes, and moving initialization outside the handler can reduce cold start duration but not eliminate it.

## Guardrail 8: Deployment Package Size Limits
Direct upload: 50 MB zipped. Unzipped (function + all layers): 250 MB. Container image: 10 GB. These are hard limits. Exceeding them fails the deployment, not the invocation.

## Guardrail 9: Environment Variable Size Limit
Total environment variable size is 4 KB. This includes keys and values. For larger configuration, use SSM Parameter Store or Secrets Manager (accessed at runtime, not deployment).

## Guardrail 10: Execution Role vs Resource Policy
Execution role: what the function CAN DO (call DynamoDB, write to S3). Resource policy: who CAN INVOKE the function (API Gateway, S3 events, other accounts). These are completely different. Never confuse them.

## Guardrail 11: Event Source Mapping Concurrency
SQS event source: Lambda scales up to the number of SQS message groups (FIFO) or up to 1000 batches/minute (standard). Kinesis/DynamoDB: one concurrent invocation per shard by default (parallelization factor increases this). Never claim event source mappings scale the same way as direct invocations.

## Guardrail 12: Layer Compatibility
Layers are extracted to /opt. They must be built for the correct runtime and architecture (x86_64 vs arm64). A Python layer built for x86 will NOT work on a Graviton (arm64) function if it contains compiled binaries.
