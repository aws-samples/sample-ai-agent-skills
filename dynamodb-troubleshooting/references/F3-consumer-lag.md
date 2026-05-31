---
title: "F3 — Consumer Lag"
description: "Monitor and resolve stream consumer lag including iterator age, 24-hour expiry, and scaling consumers"
status: active
severity: MEDIUM
triggers:
  - "consumer lag"
  - "iterator age increasing"
  - "stream behind"
  - "24 hour expiry"
  - "records expiring"
owner: devops-agent
objective: "Reduce stream consumer lag to prevent record loss from 24-hour expiry"
context: "DynamoDB Streams retains records for 24 hours. IteratorAge metric shows how far behind a consumer is in milliseconds. If IteratorAge approaches 86,400,000ms (24 hours), records will expire before being processed. Consumer lag increases when processing speed is slower than the write rate. Each shard can be read by at most 2 processes simultaneously. Scaling requires processing more shards in parallel."
---

## Phase 1 — Triage

MUST:
- Check iterator age: `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name IteratorAge --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 60 --statistics Maximum`
- Check stream shard count: `aws dynamodbstreams describe-stream --stream-arn <stream-arn> --query 'StreamDescription.Shards | length(@)'`
- Check write rate to the table: `aws cloudwatch get-metric-statistics --namespace AWS/DynamoDB --metric-name ConsumedWriteCapacityUnits --dimensions Name=TableName,Value=<table> --start-time <start> --end-time <end> --period 60 --statistics Sum`
- If using Lambda, check concurrent executions and errors

SHOULD:
- Check if the consumer is processing all shards (shards may have split)
- Verify consumer processing speed per record
- Check if there are error-induced retries slowing down processing
- Review if the consumer handles empty GetRecords responses efficiently

MAY:
- Check Lambda parallelization factor if using Lambda triggers
- Analyze write rate trends to predict future lag
- Check if the consumer can be horizontally scaled

## Phase 2 — Remediate

MUST:
- If iterator age exceeds 12 hours, take immediate action to prevent record loss
- Scale up consumer processing: increase Lambda concurrency, add more KCL workers, or increase parallelization factor
- Fix any processing errors that cause retries and slow down consumption

SHOULD:
- Increase Lambda parallelization factor: `aws lambda update-event-source-mapping --uuid <mapping-uuid> --parallelization-factor 10`
- Increase batch size to process more records per invocation: `aws lambda update-event-source-mapping --uuid <mapping-uuid> --batch-size 1000`
- Optimize consumer code to reduce per-record processing time

MAY:
- Implement batch processing instead of per-record processing
- Use multiple consumers with shard-level parallelism
- Set up CloudWatch alarms on IteratorAge with thresholds at 1 hour and 12 hours

## Common Issues

- symptoms: "Iterator age steadily increasing over time"
  diagnosis: "Write rate exceeds consumer processing rate. The gap grows continuously."
  resolution: "Increase consumer throughput: more workers, higher parallelization, larger batches, faster processing."

- symptoms: "Iterator age spikes then recovers"
  diagnosis: "Temporary write spike or consumer slowdown. Consumer catches up after the spike."
  resolution: "This may be acceptable. Set up alarms to detect if recovery doesn't happen. Consider auto-scaling consumers."

- symptoms: "Iterator age at 24 hours — records being lost"
  diagnosis: "Consumer is 24 hours behind. Records are expiring before being processed."
  resolution: "Emergency: skip to LATEST iterator to stop further loss. Investigate and fix the root cause. Accept that 24 hours of records are lost."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Increase Lambda parallelization factor | GREEN | Processes more batches concurrently; no data risk |
| Increase batch size | GREEN | Processes more records per invocation; no data risk |
| Optimize consumer code | GREEN | Application-level improvement |
| Skip to LATEST iterator | RED | Permanently skips all unprocessed records; data loss |
| Set up CloudWatch alarms | GREEN | Monitoring only; no operational impact |

## Escalation Conditions

- Iterator age exceeds 12 hours (records at risk of 24-hour expiry)
- Iterator age at 24 hours (records actively being lost)
- Consumer lag increasing despite maximum parallelization
- Write rate permanently exceeds consumer processing capacity
- Consumer lag impacting downstream data consistency or SLAs

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-metric-statistics` (IteratorAge) | LOW | Operational metrics only |
| `describe-stream` (shard count) | LOW | Stream metadata only |
| `get-metric-statistics` (consumed WCU) | LOW | Operational metrics only |

## Prohibited Actions

- NEVER suggest deleting a table to fix consumer lag
- NEVER suggest disabling encryption to improve processing speed
- NEVER suggest removing auto-scaling without understanding traffic patterns
- NEVER suggest switching capacity mode during peak traffic hours
- NEVER skip to LATEST iterator without explicit acknowledgment of data loss

## Phase 3 — Rollback

If consumer lag remediation causes issues:
1. If parallelization factor was increased and causes resource contention, reduce it: `aws lambda update-event-source-mapping --uuid <mapping-uuid> --parallelization-factor <previous>`
2. If batch size was increased and causes Lambda timeouts, reduce it: `aws lambda update-event-source-mapping --uuid <mapping-uuid> --batch-size <previous>`
3. If LATEST iterator was used, note: skipped records cannot be recovered
4. If consumer code was optimized and introduced bugs, revert to previous version
5. Monitor IteratorAge to verify lag is stabilizing after rollback

## Output Format

```yaml
root_cause: "consumer_lag — <specific_cause>"
evidence:
  - type: iterator_age
    content: "<IteratorAge metric over time>"
  - type: write_rate
    content: "<table write rate>"
  - type: consumer_throughput
    content: "<records processed per second>"
severity: MEDIUM
consumer_lag_analysis:
  iterator_age_ms: "<current age>"
  iterator_age_hours: "<hours behind>"
  write_rate_per_second: "<rate>"
  consumer_processing_rate: "<rate>"
  records_at_risk: "yes | no"
  time_to_24h_expiry: "<hours remaining>"
mitigation:
  immediate: "Scale up consumer processing capacity"
  long_term: "Implement auto-scaling consumers and monitoring alarms"
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
