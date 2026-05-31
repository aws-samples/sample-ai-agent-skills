---
title: "F1 — Stream Processing Failures"
description: "Diagnose DynamoDB Streams processing errors including iterator age and shard management"
status: active
severity: HIGH
triggers:
  - "stream processing failure"
  - "iterator age"
  - "shard iterator expired"
  - "stream records missing"
  - "DynamoDB Streams error"
owner: devops-agent
objective: "Identify and resolve DynamoDB Streams processing failures"
context: "DynamoDB Streams captures item-level changes in near real-time. Stream records are available for 24 hours. Each stream has shards that are processed independently. Iterator age indicates how far behind a consumer is. If iterator age approaches 24 hours, records will be lost. Shards split when partitions split. Consumers must handle shard lineage (parent-child relationships)."
---

## Phase 1 — Triage

MUST:
- Check stream status: `aws dynamodb describe-table --table-name <table> --query 'Table.StreamSpecification'`
- List stream shards: `aws dynamodbstreams describe-stream --stream-arn <stream-arn>`
- Check iterator age metric: `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name IteratorAge --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 60 --statistics Maximum`
- Check for Lambda trigger errors if using Lambda: `aws cloudwatch get-metric-statistics --namespace AWS/Lambda --metric-name Errors --dimensions Name=FunctionName,Value=<function> --start-time <start> --end-time <end> --period 60 --statistics Sum`

SHOULD:
- Check if the stream has the correct view type (NEW_IMAGE, OLD_IMAGE, NEW_AND_OLD_IMAGES, KEYS_ONLY)
- Verify consumer application is handling shard splits (parent-child shard lineage)
- Check for TrimmedDataAccessException (attempting to read expired records)
- Review consumer application logs for processing errors

MAY:
- Check if multiple consumers are reading the same shard (max 2 simultaneous readers per shard)
- Verify the stream ARN hasn't changed (streams are recreated when disabled/re-enabled)
- Check if the consumer handles empty GetRecords responses correctly

## Phase 2 — Remediate

MUST:
- If iterator age is high, scale up consumer processing capacity
- If shard iterator expired, get a new iterator from the shard: `aws dynamodbstreams get-shard-iterator --stream-arn <stream-arn> --shard-id <shard-id> --shard-iterator-type TRIM_HORIZON`
- Handle TrimmedDataAccessException by starting from LATEST or TRIM_HORIZON

SHOULD:
- Implement checkpointing to track processed records per shard
- Handle shard splits by detecting closed shards and processing child shards
- Use Lambda event source mappings for managed stream processing

MAY:
- Implement a dead-letter queue for records that fail processing
- Use Kinesis Client Library (KCL) adapter for complex stream processing
- Set up CloudWatch alarms on IteratorAge to detect lag early

## Common Issues

- symptoms: "Stream records are missing or incomplete"
  diagnosis: "Stream was disabled and re-enabled, creating a new stream ARN. Old stream records are lost."
  resolution: "Never disable streams if you need continuous processing. Use the new stream ARN."

- symptoms: "TrimmedDataAccessException when reading stream"
  diagnosis: "Attempting to read records older than 24 hours. Records have expired."
  resolution: "Use LATEST shard iterator type to start from current records. Implement faster processing to prevent lag."

- symptoms: "Consumer processes same records repeatedly"
  diagnosis: "Consumer is not checkpointing progress. After restart, it reprocesses from the beginning."
  resolution: "Implement checkpointing using DynamoDB or another store. Track the last processed sequence number per shard."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Scale up consumer processing | GREEN | Application-level scaling; no data impact |
| Get new shard iterator | GREEN | Resets read position; no data modification |
| Implement checkpointing | GREEN | Application-level improvement; no infrastructure change |
| Use Lambda event source mapping | GREEN | Managed processing; no direct data risk |
| Set up dead-letter queue | GREEN | Captures failed records; no data loss |

## Escalation Conditions

- Iterator age exceeds 12 hours (records at risk of 24-hour expiry)
- Stream was disabled and re-enabled (data loss occurred)
- TrimmedDataAccessException indicating records already lost
- Multiple consumers failing simultaneously on the same stream
- Stream processing failure impacting downstream data pipelines

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-table` (StreamSpecification) | LOW | Stream configuration only |
| `describe-stream` | LOW | Shard metadata only |
| `get-metric-statistics` (IteratorAge) | LOW | Operational metrics only |
| `get-records` (stream records) | HIGH | Contains actual item change data |

## Prohibited Actions

- NEVER suggest deleting a table to fix stream processing issues
- NEVER suggest disabling encryption on the stream
- NEVER suggest removing auto-scaling without understanding traffic patterns
- NEVER suggest switching capacity mode during peak traffic hours
- NEVER disable DynamoDB Streams if downstream consumers depend on them

## Phase 3 — Rollback

If stream processing changes cause issues:
1. If shard iterator was reset to LATEST, note: records between old position and LATEST are skipped (cannot recover)
2. If Lambda event source mapping was modified, revert configuration: `aws lambda update-event-source-mapping --uuid <mapping-uuid> --batch-size <previous> --parallelization-factor <previous>`
3. If consumer scaling caused resource contention, reduce consumer count
4. If checkpointing was implemented incorrectly, reset checkpoint to last known good position
5. Monitor IteratorAge to verify processing is catching up after rollback

## Output Format

```yaml
root_cause: "stream_processing — <specific_cause>"
evidence:
  - type: stream_status
    content: "<stream configuration and status>"
  - type: iterator_age
    content: "<IteratorAge metric>"
  - type: consumer_errors
    content: "<processing error details>"
severity: HIGH
stream_analysis:
  stream_enabled: "yes | no"
  stream_view_type: "<view type>"
  iterator_age_ms: "<age>"
  shard_count: "<count>"
  consumer_type: "Lambda | KCL | custom"
  records_at_risk: "yes | no"
mitigation:
  immediate: "Scale up consumer processing or reset shard iterators"
  long_term: "Implement checkpointing and shard split handling"
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
