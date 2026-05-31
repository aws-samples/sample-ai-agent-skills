---
title: "B2 — MSK Consumer Lag"
description: "Diagnose and resolve high consumer group lag"
status: active
severity: HIGH
triggers:
  - "consumer lag"
  - "offset lag"
  - "MaxOffsetLag"
  - "SumOffsetLag"
  - "consumer behind"
  - "consumer slow"
  - "EstimatedMaxTimeLag"
owner: devops-agent
objective: "Identify causes of consumer lag and restore consumers to near-real-time processing"
context: "Consumer lag is the difference between the latest offset in a partition and the consumer's committed offset. High lag means consumers cannot keep up with producers. Causes include slow message processing, insufficient consumer instances, frequent rebalancing, large messages, or producer bursts. MSK provides per-consumer-group lag metrics in CloudWatch when enhanced monitoring is enabled."
---

## Phase 1 — Triage

MUST:
- Check consumer group lag: `kafka-consumer-groups.sh --bootstrap-server <broker>:9092 --describe --group <consumer-group>`
- Check MaxOffsetLag: `aws cloudwatch get-metric-statistics --namespace AWS/Kafka --metric-name MaxOffsetLag --dimensions Name=Cluster\ Name,Value=<cluster-name> Name=Consumer\ Group,Value=<group> Name=Topic,Value=<topic> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
- Check EstimatedMaxTimeLag: `aws cloudwatch get-metric-statistics --namespace AWS/Kafka --metric-name EstimatedMaxTimeLag --dimensions Name=Cluster\ Name,Value=<cluster-name> Name=Consumer\ Group,Value=<group> Name=Topic,Value=<topic> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
- Check consumer group state: `kafka-consumer-groups.sh --bootstrap-server <broker>:9092 --describe --group <consumer-group> --state`

SHOULD:
- Check BytesOutPerSec per broker (consumer fetch rate)
- Check FetchMessageConversionsPerSec (message format conversion overhead)
- Check if consumer group is rebalancing frequently
- Compare producer rate (BytesInPerSec) vs consumer rate (BytesOutPerSec)

MAY:
- Check consumer-side metrics (records-consumed-rate, fetch-latency-avg, rebalance-rate)
- Check partition assignment across consumer instances
- Review consumer configuration (max.poll.records, fetch.max.bytes, session.timeout.ms)

## Phase 2 — Remediate

MUST:
- If consumer processing is slow: optimize message processing logic or increase consumer instances
- If insufficient consumers: add consumer instances (up to partition count)
- If frequent rebalancing: increase session.timeout.ms and heartbeat.interval.ms
- If producer burst caused lag: lag should recover naturally if consumers have sufficient capacity

SHOULD:
- Increase max.poll.records if processing is fast per-record but poll overhead is high
- Increase fetch.min.bytes and fetch.max.wait.ms to batch fetches
- Ensure consumer count does not exceed partition count (excess consumers are idle)
- Use cooperative rebalancing (partition.assignment.strategy=CooperativeStickyAssignor)

MAY:
- Consider parallel processing within each consumer (thread pool per partition)
- Reset consumer offsets to skip backlog if data is stale: `kafka-consumer-groups.sh --bootstrap-server <broker>:9092 --group <group> --topic <topic> --reset-offsets --to-latest --execute`
- Increase topic partition count to allow more parallel consumers

## Common Issues

- symptoms: "Consumer lag growing steadily over time"
  diagnosis: "Producer rate exceeds consumer processing capacity."
  resolution: "Add consumer instances or optimize processing. Increase partitions if consumer count is at partition limit."

- symptoms: "Consumer lag spikes during rebalancing"
  diagnosis: "Stop-the-world rebalancing pauses all consumers."
  resolution: "Use CooperativeStickyAssignor. Increase session.timeout.ms. Reduce max.poll.interval.ms if processing is fast."

- symptoms: "Some partitions have high lag, others have zero"
  diagnosis: "Uneven partition assignment or hot partitions with more data."
  resolution: "Check partition assignment. Consider custom partitioner for even distribution."

## Output Format

```yaml
root_cause: "consumer_lag — <specific_cause>"
evidence:
  - type: consumer_group
    content: "<consumer group describe output with lag per partition>"
  - type: cloudwatch_metrics
    content: "<MaxOffsetLag, EstimatedMaxTimeLag>"
  - type: throughput_comparison
    content: "<BytesInPerSec vs BytesOutPerSec>"
severity: HIGH
mitigation:
  immediate: "Scale consumers or optimize processing to reduce lag"
  long_term: "Right-size consumer fleet, use cooperative rebalancing, monitor lag metrics"
```


## Safety Ratings
```
safety_ratings:
  - "Describe consumer group: GREEN — read-only Kafka CLI command"
  - "Check CloudWatch lag metrics: GREEN — read-only monitoring query"
  - "Add consumer instances: GREEN — scales consumer fleet without cluster changes"
  - "Increase max.poll.records: GREEN — client-side configuration change"
  - "Reset consumer offsets to latest: RED — permanently skips unprocessed messages"
  - "Increase topic partition count: YELLOW — irreversible, requires consumer rebalance"
```

## Escalation Conditions
- Cluster serves production event streaming
- Consumer lag growing continuously with no sign of recovery
- EstimatedMaxTimeLag exceeds SLA thresholds
- Lag causing downstream system data staleness
- Fix requires resetting offsets (data loss) or partition changes

## Data Sensitivity
```
data_sensitivity:
  classification: MEDIUM
  sensitive_fields:
    - "Consumer group names: application identifiers"
    - "Topic names and offsets: data flow state"
    - "Lag metrics: processing capacity indicators"
  handling: "Do not expose consumer group names or offset positions externally. Mask topic names in shared reports."
```

## Prohibited Actions
- NEVER suggest deleting topics with active consumers
- NEVER suggest reducing broker count
- NEVER reset consumer offsets without explicit confirmation of acceptable data loss
- NEVER add consumers beyond the partition count (they will be idle)
- NEVER change partition.assignment.strategy on a running consumer group without a rolling restart plan

## Phase 3 — Rollback
- If consumer offsets were reset to latest: CANNOT be rolled back — skipped messages are permanently missed unless replayed from another source
- If max.poll.records was changed: revert client configuration and redeploy consumers
- If partition count was increased: CANNOT be rolled back — adjust consumer count to match
- If CooperativeStickyAssignor was enabled: revert to previous assignment strategy with rolling consumer restart
- If consumer instances were added: scale down consumer fleet to previous count if lag is resolved

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling encryption in transit"
  - "NEVER suggest removing authentication"
  - "NEVER suggest public access to brokers"
