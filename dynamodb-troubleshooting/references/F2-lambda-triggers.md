---
title: "F2 — Lambda Trigger Issues"
description: "Diagnose Lambda event source mapping errors, batch size, error handling, and bisect on batch failure"
status: active
severity: HIGH
triggers:
  - "Lambda trigger"
  - "event source mapping"
  - "Lambda DynamoDB"
  - "batch failure"
  - "bisect batch"
owner: devops-agent
objective: "Identify and resolve Lambda trigger issues with DynamoDB Streams"
context: "Lambda event source mappings poll DynamoDB Streams and invoke Lambda functions with batches of records. Batch size (1-10000) controls records per invocation. On failure, the entire batch is retried by default, which can cause a poison pill scenario. BisectBatchOnFunctionError splits failed batches to isolate the problematic record. MaximumRetryAttempts and MaximumRecordAgeInSeconds control retry behavior. A failed batch blocks the shard until resolved."
---

## Phase 1 — Triage

MUST:
- List event source mappings: `aws lambda list-event-source-mappings --function-name <function> --event-source-arn <stream-arn>`
- Check event source mapping state: `aws lambda get-event-source-mapping --uuid <mapping-uuid>`
- Check Lambda errors: `aws cloudwatch get-metric-statistics --namespace AWS/Lambda --metric-name Errors --dimensions Name=FunctionName,Value=<function> --start-time <start> --end-time <end> --period 60 --statistics Sum`
- Check iterator age (indicates processing lag): `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name IteratorAge --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 60 --statistics Maximum`

SHOULD:
- Check Lambda function timeout and memory configuration: `aws lambda get-function-configuration --function-name <function>`
- Check if BisectBatchOnFunctionError is enabled in the event source mapping
- Check MaximumRetryAttempts and MaximumRecordAgeInSeconds settings
- Review Lambda function logs: `aws logs filter-log-events --log-group-name /aws/lambda/<function> --start-time <epoch-ms> --end-time <epoch-ms>`

MAY:
- Check if a destination is configured for failed records: `aws lambda get-event-source-mapping --uuid <mapping-uuid> --query 'DestinationConfig'`
- Check Lambda concurrent execution limits: `aws lambda get-account-settings`
- Review if the function handles partial batch failures with ReportBatchItemFailures

## Phase 2 — Remediate

MUST:
- If a poison pill record is blocking the shard, enable BisectBatchOnFunctionError: `aws lambda update-event-source-mapping --uuid <mapping-uuid> --bisect-batch-on-function-error`
- Set MaximumRetryAttempts to prevent infinite retries: `aws lambda update-event-source-mapping --uuid <mapping-uuid> --maximum-retry-attempts 3`
- Configure a dead-letter destination for failed records: `aws lambda update-event-source-mapping --uuid <mapping-uuid> --destination-config '{"OnFailure":{"Destination":"<sqs-arn>"}}'`

SHOULD:
- Enable ReportBatchItemFailures in the function response to only retry failed records
- Set MaximumRecordAgeInSeconds to skip old records: `aws lambda update-event-source-mapping --uuid <mapping-uuid> --maximum-record-age-in-seconds 3600`
- Optimize batch size based on processing time and record size
- Increase Lambda timeout and memory if processing is slow

MAY:
- Implement idempotent processing to handle retries safely
- Use parallelization factor to process multiple batches per shard: `aws lambda update-event-source-mapping --uuid <mapping-uuid> --parallelization-factor 5`
- Add error handling in the Lambda function to catch and log individual record failures

## Common Issues

- symptoms: "Lambda function keeps retrying the same batch"
  diagnosis: "A single bad record (poison pill) causes the entire batch to fail. Without BisectBatchOnFunctionError, the same batch retries indefinitely."
  resolution: "Enable BisectBatchOnFunctionError and set MaximumRetryAttempts. Implement ReportBatchItemFailures for granular error handling."

- symptoms: "Iterator age increasing despite Lambda running"
  diagnosis: "Lambda processing time exceeds the rate of incoming records. Or Lambda is hitting concurrency limits."
  resolution: "Increase parallelization factor. Optimize Lambda code. Increase batch size to process more records per invocation."

- symptoms: "Event source mapping in disabled state"
  diagnosis: "Too many consecutive failures caused the mapping to disable, or it was manually disabled."
  resolution: "Fix the underlying Lambda error. Re-enable: `aws lambda update-event-source-mapping --uuid <mapping-uuid> --enabled`"

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Enable BisectBatchOnFunctionError | GREEN | Improves error isolation; no data impact |
| Set MaximumRetryAttempts | GREEN | Prevents infinite retries; failed records go to DLQ |
| Configure dead-letter destination | GREEN | Captures failed records; no data loss |
| Set MaximumRecordAgeInSeconds | YELLOW | Old records will be skipped; potential data loss if set too low |
| Increase parallelization factor | GREEN | Processes more batches concurrently; no data risk |
| Enable ReportBatchItemFailures | GREEN | Application-level improvement; granular retry control |

## Escalation Conditions

- Poison pill record blocking a shard for more than 1 hour
- Event source mapping auto-disabled due to consecutive failures
- Iterator age exceeding 12 hours due to Lambda failures
- Lambda function hitting concurrency limits affecting stream processing
- Failed records accumulating in dead-letter queue without resolution

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `list-event-source-mappings` | LOW | Mapping configuration only |
| `get-function-configuration` | LOW | Lambda configuration only |
| `get-metric-statistics` (Lambda errors) | LOW | Operational metrics only |
| `filter-log-events` (Lambda logs) | HIGH | May contain actual record data in error messages |

## Prohibited Actions

- NEVER suggest deleting a table to fix Lambda trigger issues
- NEVER suggest disabling encryption on the stream or Lambda function
- NEVER suggest removing auto-scaling without understanding traffic patterns
- NEVER suggest switching capacity mode during peak traffic hours
- NEVER set MaximumRecordAgeInSeconds too low on critical data pipelines (risk of data loss)

## Phase 3 — Rollback

If Lambda trigger changes cause issues:
1. Revert event source mapping configuration: `aws lambda update-event-source-mapping --uuid <mapping-uuid> --batch-size <previous> --bisect-batch-on-function-error --maximum-retry-attempts <previous> --parallelization-factor <previous>`
2. If MaximumRecordAgeInSeconds was set too low, increase it: `aws lambda update-event-source-mapping --uuid <mapping-uuid> --maximum-record-age-in-seconds <higher-value>`
3. If event source mapping was disabled, re-enable: `aws lambda update-event-source-mapping --uuid <mapping-uuid> --enabled`
4. If dead-letter destination was misconfigured, fix the SQS/SNS ARN
5. Process any records accumulated in the dead-letter queue

## Output Format

```yaml
root_cause: "lambda_trigger — <specific_cause>"
evidence:
  - type: event_source_mapping
    content: "<mapping configuration and state>"
  - type: lambda_errors
    content: "<error count and log entries>"
  - type: iterator_age
    content: "<IteratorAge metric>"
severity: HIGH
lambda_trigger_analysis:
  mapping_state: "Enabled | Disabled | Creating | Updating"
  batch_size: "<size>"
  bisect_on_error: "true | false"
  max_retry_attempts: "<count>"
  max_record_age_seconds: "<seconds>"
  parallelization_factor: "<factor>"
  destination_configured: "yes | no"
mitigation:
  immediate: "Enable BisectBatchOnFunctionError and set retry limits"
  long_term: "Implement ReportBatchItemFailures and dead-letter destination"
```

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
  - "NEVER suggest disabling encryption at rest"
  - "NEVER suggest deleting a table to fix issues"
  - "NEVER run full table scans on production tables"
