---
title: "D2 — MSK Partition Rebalancing"
description: "Diagnose and resolve partition rebalancing and reassignment issues"
status: active
severity: MEDIUM
triggers:
  - "partition rebalance"
  - "partition reassignment"
  - "uneven partitions"
  - "leader imbalance"
  - "preferred leader"
  - "hot partition"
owner: devops-agent
objective: "Achieve balanced partition distribution across brokers for optimal performance"
context: "Partition imbalance occurs when partition leaders or replicas are unevenly distributed across brokers. This causes some brokers to be overloaded while others are underutilized. Imbalance can result from broker additions, broker replacements, topic creation patterns, or failed preferred leader elections. MSK does not automatically rebalance partitions — manual reassignment is required."
---

## Phase 1 — Triage

MUST:
- Check per-broker partition count and leader count: `kafka-topics.sh --bootstrap-server <broker>:9092 --describe --topic <topic>`
- Check per-broker CPU utilization to identify hot brokers
- Check per-broker BytesInPerSec and BytesOutPerSec for throughput imbalance
- Check per-broker disk usage for storage imbalance

SHOULD:
- List all topics and their partition assignments: `kafka-topics.sh --bootstrap-server <broker>:9092 --describe`
- Check LeaderCount per broker via JMX or enhanced monitoring
- Check PartitionCount per broker
- Identify brokers with disproportionate leader assignments

MAY:
- Generate a reassignment plan: `kafka-reassign-partitions.sh --bootstrap-server <broker>:9092 --topics-to-move-json-file topics.json --broker-list <broker-ids> --generate`
- Check for ongoing reassignments: `kafka-reassign-partitions.sh --bootstrap-server <broker>:9092 --reassignment-json-file reassignment.json --verify`

## Phase 2 — Remediate

MUST:
- If leader imbalance: trigger preferred leader election: `kafka-leader-election.sh --bootstrap-server <broker>:9092 --election-type preferred --all-topic-partitions`
- If partition distribution uneven after adding brokers: reassign partitions to include new brokers
- If hot partitions: consider increasing partition count for the hot topic

SHOULD:
- Execute reassignment during low-traffic periods to minimize impact
- Throttle reassignment to avoid overwhelming brokers: use `--throttle` flag
- Monitor reassignment progress and broker metrics during rebalancing
- Verify balance after reassignment completes

MAY:
- Use Cruise Control (if deployed) for automated rebalancing
- Create a custom reassignment plan for fine-grained control
- Consider rack-aware assignment for multi-AZ resilience

## Common Issues

- symptoms: "New broker added but has no partitions"
  diagnosis: "MSK does not automatically reassign existing partitions to new brokers."
  resolution: "Generate and execute a partition reassignment plan including the new broker."

- symptoms: "One broker has 80% CPU while others are at 20%"
  diagnosis: "Most partition leaders assigned to one broker."
  resolution: "Run preferred leader election. If still imbalanced, reassign partitions."

- symptoms: "Reassignment taking very long"
  diagnosis: "Large partitions being moved without throttling, or throttle too low."
  resolution: "Adjust throttle rate. Monitor ReassignmentBytesInPerSec and ReassignmentBytesOutPerSec."

## Output Format

```yaml
root_cause: "partition_rebalancing — <specific_cause>"
evidence:
  - type: partition_distribution
    content: "<partition and leader count per broker>"
  - type: broker_metrics
    content: "<CPU, throughput, disk per broker>"
  - type: reassignment_status
    content: "<reassignment progress if applicable>"
severity: MEDIUM
mitigation:
  immediate: "Rebalance partitions or trigger preferred leader election"
  long_term: "Monitor partition distribution, plan for rebalancing after scaling"
```


## Safety Ratings
```
safety_ratings:
  - "Check partition distribution: GREEN — read-only Kafka CLI command"
  - "Check per-broker metrics: GREEN — read-only monitoring query"
  - "Trigger preferred leader election: YELLOW — causes brief leader transitions, minimal impact"
  - "Generate reassignment plan: GREEN — generates plan without executing"
  - "Execute partition reassignment: YELLOW — moves data between brokers, impacts throughput during transfer"
  - "Increase partition count: YELLOW — irreversible, triggers consumer rebalance"
```

## Escalation Conditions
- Cluster serves production event streaming
- Severe partition imbalance causing broker overload
- Reassignment required during business hours with production traffic
- New brokers added but not receiving any partitions
- Hot partitions causing cascading performance issues

## Data Sensitivity
```
data_sensitivity:
  classification: MEDIUM
  sensitive_fields:
    - "Partition assignment maps: topic-to-broker mapping"
    - "Broker utilization metrics: capacity data"
    - "Reassignment plans: infrastructure change details"
  handling: "Do not expose partition assignment details or reassignment plans externally."
```

## Prohibited Actions
- NEVER suggest deleting topics with active consumers
- NEVER suggest reducing broker count
- NEVER execute partition reassignment without throttling on production clusters
- NEVER reassign all partitions simultaneously — do it in batches
- NEVER increase partition count without understanding consumer group impact

## Phase 3 — Rollback
- If preferred leader election was triggered: no rollback needed — election is non-destructive
- If partition reassignment was executed: generate and execute a reverse reassignment plan using the original assignment
- If partition count was increased: CANNOT be rolled back — partitions cannot be decreased
- If throttle was set too high during reassignment: reduce throttle rate immediately to limit impact
- If reassignment is in progress and causing issues: cancel by removing the reassignment throttle and waiting for completion, or contact AWS Support

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
