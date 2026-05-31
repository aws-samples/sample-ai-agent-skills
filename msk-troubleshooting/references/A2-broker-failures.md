---
title: "A2 — MSK Broker Failures"
description: "Diagnose and resolve MSK broker failures and unhealthy brokers"
status: active
severity: CRITICAL
triggers:
  - "broker failure"
  - "broker unhealthy"
  - "broker down"
  - "ActiveControllerCount 0"
  - "OfflinePartitionsCount"
  - "broker not responding"
owner: devops-agent
objective: "Identify failed or unhealthy brokers, determine root cause, and restore cluster health"
context: "MSK broker failures can be caused by storage exhaustion, CPU overload, memory pressure, network issues, or underlying infrastructure problems. AWS automatically replaces failed brokers, but recovery takes time. During broker failure, partitions led by that broker become unavailable unless replicas exist on other brokers. ActiveControllerCount dropping to 0 indicates a severe cluster issue."
---

## Phase 1 — Triage

MUST:
- Check cluster status: `aws kafka describe-cluster-v2 --cluster-arn <cluster-arn>`
- List broker nodes: `aws kafka list-nodes --cluster-arn <cluster-arn>`
- Check ActiveControllerCount: `aws cloudwatch get-metric-statistics --namespace AWS/Kafka --metric-name ActiveControllerCount --dimensions Name=Cluster\ Name,Value=<cluster-name> --start-time <start> --end-time <end> --period 60 --statistics Minimum`
- Check OfflinePartitionsCount: `aws cloudwatch get-metric-statistics --namespace AWS/Kafka --metric-name OfflinePartitionsCount --dimensions Name=Cluster\ Name,Value=<cluster-name> --start-time <start> --end-time <end> --period 60 --statistics Maximum`
- Check per-broker CPU: `aws cloudwatch get-metric-statistics --namespace AWS/Kafka --metric-name CpuUser --dimensions Name=Cluster\ Name,Value=<cluster-name> Name=Broker\ ID,Value=<broker-id> --start-time <start> --end-time <end> --period 300 --statistics Average,Maximum`

SHOULD:
- Check per-broker disk usage: `aws cloudwatch get-metric-statistics --namespace AWS/Kafka --metric-name KafkaDataLogsDiskUsed --dimensions Name=Cluster\ Name,Value=<cluster-name> Name=Broker\ ID,Value=<broker-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
- Check UnderReplicatedPartitions per broker: `aws cloudwatch get-metric-statistics --namespace AWS/Kafka --metric-name UnderReplicatedPartitions --dimensions Name=Cluster\ Name,Value=<cluster-name> Name=Broker\ ID,Value=<broker-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
- Check recent cluster operations: `aws kafka list-cluster-operations --cluster-arn <cluster-arn>`
- Check CloudTrail for events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=kafka.amazonaws.com --max-results 20`

MAY:
- Check per-broker memory usage: `aws cloudwatch get-metric-statistics --namespace AWS/Kafka --metric-name MemoryUsed --dimensions Name=Cluster\ Name,Value=<cluster-name> Name=Broker\ ID,Value=<broker-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
- Check network throughput per broker
- Review broker logs if enabled (CloudWatch Logs, S3, or Firehose)

## Phase 2 — Remediate

MUST:
- If storage full: increase broker storage or enable auto-scaling (see A3)
- If CPU overloaded: scale to larger instance type or add brokers and rebalance partitions
- If ActiveControllerCount = 0: this is critical — wait for AWS automatic recovery or contact AWS Support
- If broker being replaced: monitor replacement progress via list-nodes

SHOULD:
- Ensure replication factor >= 2 for all critical topics to survive broker failures
- Enable enhanced monitoring for better visibility
- Set up CloudWatch alarms for ActiveControllerCount < 1 and OfflinePartitionsCount > 0

MAY:
- Rebalance partitions after broker recovery using `kafka-reassign-partitions.sh`
- Consider multi-AZ deployment for higher availability
- Review instance type sizing for the workload

## Common Issues

- symptoms: "ActiveControllerCount drops to 0 intermittently"
  diagnosis: "Controller election in progress, possibly due to broker instability or ZooKeeper issues."
  resolution: "Monitor for recovery. If sustained, check broker resource utilization and contact AWS Support."

- symptoms: "Single broker shows high CPU while others are idle"
  diagnosis: "Uneven partition distribution — hot partitions concentrated on one broker."
  resolution: "Rebalance partitions across brokers. Consider increasing partition count for hot topics."

- symptoms: "Broker replaced but partitions not reassigned"
  diagnosis: "New broker has a different broker ID. Partitions need to be reassigned."
  resolution: "Use kafka-reassign-partitions.sh to move partitions to the new broker."

## Output Format

```yaml
root_cause: "broker_failure — <specific_cause>"
evidence:
  - type: cluster_status
    content: "<cluster and node status>"
  - type: cloudwatch_metrics
    content: "<ActiveControllerCount, OfflinePartitionsCount, per-broker CPU/disk>"
  - type: cluster_operations
    content: "<recent operations>"
severity: CRITICAL
mitigation:
  immediate: "Restore broker health or wait for automatic replacement"
  long_term: "Ensure adequate replication, monitoring, and instance sizing"
```


## Safety Ratings
```
safety_ratings:
  - "Describe cluster and list nodes: GREEN — read-only API calls"
  - "Check CloudWatch metrics: GREEN — read-only monitoring query"
  - "Check CloudTrail events: GREEN — read-only audit log query"
  - "Scale to larger instance type: RED — triggers rolling restart, causes temporary partition unavailability"
  - "Add brokers to cluster: YELLOW — adds capacity but requires partition rebalancing"
  - "Rebalance partitions after recovery: YELLOW — moves data between brokers, impacts throughput during transfer"
```

## Escalation Conditions
- Cluster serves production event streaming
- ActiveControllerCount drops to 0 (critical cluster health issue)
- OfflinePartitionsCount > 0 causing data unavailability
- Multiple brokers failing simultaneously
- Broker replacement by AWS taking longer than expected

## Data Sensitivity
```
data_sensitivity:
  classification: MEDIUM
  sensitive_fields:
    - "Topic configurations: consumer group names and offsets"
    - "Broker metrics: throughput patterns revealing business activity"
    - "Cluster ARN: resource identifier"
  handling: "Do not expose consumer group names or topic names externally. Mask cluster ARN in shared logs."
```

## Prohibited Actions
- NEVER suggest deleting topics with active consumers
- NEVER suggest reducing broker count
- NEVER force-stop a broker replacement in progress
- NEVER modify cluster configuration during an active broker failure (wait for stabilization)
- NEVER reduce replication factor on production topics during an incident

## Phase 3 — Rollback
- If instance type was scaled up: scale back to previous instance type during a maintenance window (triggers rolling restart)
- If brokers were added: brokers cannot be removed — plan partition rebalancing to utilize new capacity
- If partition reassignment was executed: re-run reassignment with the original assignment plan to restore previous distribution
- If monitoring configuration was changed: revert CloudWatch alarm thresholds to previous values
- If enhanced monitoring was enabled: can be disabled, but recommended to keep for visibility

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
