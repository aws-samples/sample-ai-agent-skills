---
title: "B3 — MSK High Latency"
description: "Diagnose and resolve high end-to-end latency in MSK"
status: active
severity: HIGH
triggers:
  - "high latency"
  - "slow Kafka"
  - "produce latency"
  - "fetch latency"
  - "request latency"
  - "ProduceTotalTimeMs"
owner: devops-agent
objective: "Identify sources of latency and reduce end-to-end message delivery time"
context: "End-to-end Kafka latency includes producer batching time, network transit, broker processing, replication, and consumer fetch time. High latency can be caused by large batch sizes with high linger.ms, broker overload, cross-AZ replication, slow disk I/O, network congestion, or consumer fetch configuration. MSK provides ProduceTotalTimeMs and FetchConsumerTotalTimeMs metrics."
---

## Phase 1 — Triage

MUST:
- Check ProduceTotalTimeMs: `aws cloudwatch get-metric-statistics --namespace AWS/Kafka --metric-name ProduceTotalTimeMs --dimensions Name=Cluster\ Name,Value=<cluster-name> Name=Broker\ ID,Value=<broker-id> --start-time <start> --end-time <end> --period 300 --statistics Average,p99`
- Check FetchConsumerTotalTimeMs: `aws cloudwatch get-metric-statistics --namespace AWS/Kafka --metric-name FetchConsumerTotalTimeMs --dimensions Name=Cluster\ Name,Value=<cluster-name> Name=Broker\ ID,Value=<broker-id> --start-time <start> --end-time <end> --period 300 --statistics Average,p99`
- Check broker CPU: `aws cloudwatch get-metric-statistics --namespace AWS/Kafka --metric-name CpuUser --dimensions Name=Cluster\ Name,Value=<cluster-name> Name=Broker\ ID,Value=<broker-id> --start-time <start> --end-time <end> --period 300 --statistics Average,Maximum`
- Check UnderReplicatedPartitions (replication lag): `aws cloudwatch get-metric-statistics --namespace AWS/Kafka --metric-name UnderReplicatedPartitions --dimensions Name=Cluster\ Name,Value=<cluster-name> Name=Broker\ ID,Value=<broker-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum`

SHOULD:
- Check RequestHandlerAvgIdlePercent (< 0.3 indicates thread saturation)
- Check NetworkProcessorAvgIdlePercent
- Check disk I/O metrics (if enhanced monitoring enabled)
- Review producer linger.ms and batch.size settings

MAY:
- Check cross-AZ data transfer metrics
- Review consumer fetch.min.bytes and fetch.max.wait.ms settings
- Check for message format conversions (ProduceMessageConversionsPerSec)

## Phase 2 — Remediate

MUST:
- If broker CPU > 80%: scale instance type or add brokers
- If RequestHandlerAvgIdlePercent < 0.3: add brokers or scale instance type
- If linger.ms too high for latency requirements: reduce linger.ms (trade throughput for latency)
- If acks=all causing latency: ensure min.insync.replicas is not equal to replication factor

SHOULD:
- For low-latency: set linger.ms=0 or 1, reduce batch.size
- For low-latency consumers: set fetch.min.bytes=1 and fetch.max.wait.ms=100
- Ensure brokers are not disk I/O bound (use gp3 or io1 volumes)
- Consider rack-aware replica placement to minimize cross-AZ replication latency

MAY:
- Use compression to reduce network transfer time (lz4 is fastest)
- Consider reducing replication factor for non-critical low-latency topics
- Enable client-side metrics to identify which phase contributes most to latency

## Common Issues

- symptoms: "p99 produce latency spikes periodically"
  diagnosis: "Broker garbage collection pauses or log segment rolling."
  resolution: "Scale broker instance type for more memory. Check log.segment.bytes and log.roll.ms settings."

- symptoms: "Consistently high produce latency with acks=all"
  diagnosis: "Cross-AZ replication adds latency for each ack from remote replicas."
  resolution: "This is expected for multi-AZ. Consider acks=1 for latency-sensitive, loss-tolerant topics."

- symptoms: "Consumer fetch latency high despite low broker load"
  diagnosis: "fetch.max.wait.ms too high or fetch.min.bytes too large."
  resolution: "Reduce fetch.max.wait.ms and fetch.min.bytes for lower latency."

## Output Format

```yaml
root_cause: "high_latency — <specific_cause>"
evidence:
  - type: latency_metrics
    content: "<ProduceTotalTimeMs, FetchConsumerTotalTimeMs>"
  - type: broker_metrics
    content: "<CPU, RequestHandlerAvgIdlePercent>"
  - type: client_config
    content: "<linger.ms, batch.size, acks, fetch settings>"
severity: HIGH
mitigation:
  immediate: "Tune client configuration or scale brokers"
  long_term: "Right-size cluster, optimize client settings, monitor latency percentiles"
```


## Safety Ratings
```
safety_ratings:
  - "Check latency metrics: GREEN — read-only monitoring query"
  - "Check broker CPU and idle percentages: GREEN — read-only monitoring query"
  - "Reduce linger.ms for lower latency: GREEN — client-side configuration change"
  - "Scale broker instance type: RED — triggers rolling restart, temporary partition unavailability"
  - "Change acks setting: YELLOW — reducing acks trades durability for latency"
  - "Reduce replication factor: RED — reduces data durability, risk of data loss on broker failure"
```

## Escalation Conditions
- Cluster serves production event streaming
- p99 latency exceeding SLA thresholds
- Latency spikes correlated with broker instability
- Fix requires instance type change during business hours
- Latency impacting real-time processing pipelines

## Data Sensitivity
```
data_sensitivity:
  classification: MEDIUM
  sensitive_fields:
    - "Latency percentile metrics: performance SLA data"
    - "Client configuration: acks, linger.ms, batch.size settings"
    - "Broker resource utilization: capacity planning data"
  handling: "Do not expose latency SLA details or client configuration externally."
```

## Prohibited Actions
- NEVER suggest deleting topics with active consumers
- NEVER suggest reducing broker count
- NEVER set acks=0 on topics carrying critical production data
- NEVER reduce replication factor below 2 for production topics
- NEVER disable TLS encryption to reduce latency

## Phase 3 — Rollback
- If linger.ms was reduced: increase linger.ms back to previous value if throughput drops unacceptably
- If acks was changed from all to 1: revert to acks=all if durability requirements demand it
- If broker instance type was scaled: scale back during maintenance window (triggers rolling restart)
- If fetch.max.wait.ms was reduced: increase back to previous value if broker load increases
- If compression was changed: revert to previous compression setting in producer configuration

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
