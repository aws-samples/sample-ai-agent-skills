---
title: "A3 — MSK Storage Full"
description: "Diagnose and resolve MSK broker storage exhaustion"
status: active
severity: CRITICAL
triggers:
  - "storage full"
  - "disk full"
  - "KafkaDataLogsDiskUsed"
  - "broker storage"
  - "no space"
  - "disk usage 100%"
owner: devops-agent
objective: "Free or expand broker storage to restore cluster health"
context: "When MSK broker storage reaches 100%, the broker becomes unhealthy and stops accepting writes. Storage auto-scaling is NOT enabled by default. Once storage is increased, it cannot be decreased. The KafkaDataLogsDiskUsed metric should be alarmed at 85%. Common causes include insufficient retention policies, high producer throughput, disabled auto-scaling, or log compaction not configured."
---

## Phase 1 — Triage

MUST:
- Check disk usage per broker: `aws cloudwatch get-metric-statistics --namespace AWS/Kafka --metric-name KafkaDataLogsDiskUsed --dimensions Name=Cluster\ Name,Value=<cluster-name> Name=Broker\ ID,Value=<broker-id> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
- Check cluster storage config: `aws kafka describe-cluster-v2 --cluster-arn <cluster-arn> --query 'ClusterInfo.Provisioned.BrokerNodeGroupInfo.StorageInfo'`
- Check if auto-scaling is enabled: `aws kafka describe-cluster-v2 --cluster-arn <cluster-arn> --query 'ClusterInfo.Provisioned.StorageMode'`
- Check cluster status: `aws kafka describe-cluster-v2 --cluster-arn <cluster-arn> --query 'ClusterInfo.{State:State,StateInfo:StateInfo}'`

SHOULD:
- Check topic retention settings: `kafka-configs.sh --bootstrap-server <broker>:9092 --describe --entity-type topics --entity-name <topic> --all | grep retention`
- List topics by size (using Kafka CLI): `kafka-log-dirs.sh --bootstrap-server <broker>:9092 --describe`
- Check broker log segment sizes
- Check for topics with no retention limit (retention.ms=-1)

MAY:
- Check producer throughput trends: `aws cloudwatch get-metric-statistics --namespace AWS/Kafka --metric-name BytesInPerSec --dimensions Name=Cluster\ Name,Value=<cluster-name> --start-time <start> --end-time <end> --period 3600 --statistics Sum`
- Review compaction settings for compacted topics
- Check for abandoned topics consuming storage

## Phase 2 — Remediate

MUST:
- If auto-scaling not enabled: enable it immediately: `aws kafka update-storage --cluster-arn <cluster-arn> --provisioned-throughput '{"Enabled":false}' --storage-mode LOCAL --current-version <version> --target-broker-ebs-volume-info '[{"VolumeSizeGB":<new-size>}]'`
- If immediate space needed: reduce retention on non-critical topics: `kafka-configs.sh --bootstrap-server <broker>:9092 --alter --entity-type topics --entity-name <topic> --add-config retention.ms=<shorter-retention>`
- Delete unused or abandoned topics: `kafka-topics.sh --bootstrap-server <broker>:9092 --delete --topic <topic>`

SHOULD:
- Enable storage auto-scaling with appropriate max limit
- Set retention policies on all topics (avoid retention.ms=-1 in production)
- Set up CloudWatch alarm on KafkaDataLogsDiskUsed at 85%

MAY:
- Enable log compaction for topics where only latest value per key matters
- Consider tiered storage (if available) for cost-effective long-term retention
- Review partition count — more partitions can spread data more evenly

## Common Issues

- symptoms: "Broker unhealthy, KafkaDataLogsDiskUsed at 100%"
  diagnosis: "Storage exhausted. Auto-scaling not enabled or max limit reached."
  resolution: "Increase storage manually or enable auto-scaling. Reduce retention on large topics."

- symptoms: "Storage growing faster than expected"
  diagnosis: "Topics with retention.ms=-1 (infinite retention) or very long retention."
  resolution: "Set appropriate retention.ms and retention.bytes on all topics."

- symptoms: "Auto-scaling enabled but storage still full"
  diagnosis: "Auto-scaling max limit reached or cooldown period active."
  resolution: "Increase auto-scaling max limit. Manually increase storage if urgent."

## Output Format

```yaml
root_cause: "storage_full — <specific_cause>"
evidence:
  - type: disk_usage
    content: "<KafkaDataLogsDiskUsed per broker>"
  - type: storage_config
    content: "<storage size, auto-scaling status>"
  - type: retention_config
    content: "<topic retention settings>"
severity: CRITICAL
mitigation:
  immediate: "Increase storage or reduce retention to free space"
  long_term: "Enable auto-scaling, set retention policies, monitor disk usage"
```


## Safety Ratings
```
safety_ratings:
  - "Check disk usage metrics: GREEN — read-only monitoring query"
  - "Check cluster storage config: GREEN — read-only API call"
  - "Reduce topic retention: YELLOW — causes older messages to be deleted permanently"
  - "Increase broker storage: YELLOW — irreversible operation, storage cannot be decreased"
  - "Enable storage auto-scaling: GREEN — sets policy but does not immediately change storage"
  - "Delete unused topics: RED — permanently removes topic and all messages"
```

## Escalation Conditions
- Cluster serves production event streaming
- Broker storage at 100% causing write failures
- Auto-scaling max limit reached with storage still growing
- Multiple brokers simultaneously at storage capacity
- Fix requires deleting topics that may have active consumers

## Data Sensitivity
```
data_sensitivity:
  classification: MEDIUM
  sensitive_fields:
    - "Topic names and retention settings: business data flow identifiers"
    - "Storage utilization patterns: capacity planning data"
    - "Producer throughput metrics: business activity indicators"
  handling: "Do not expose topic names or retention policies externally. Mask cluster identifiers in shared reports."
```

## Prohibited Actions
- NEVER suggest deleting topics with active consumers
- NEVER suggest reducing broker count
- NEVER reduce retention to 0 on any topic without explicit confirmation
- NEVER delete topics without verifying consumer group status first
- NEVER assume storage increase can be reversed (it cannot)

## Phase 3 — Rollback
- If retention was reduced on a topic: increase retention.ms back to original value (already-deleted messages cannot be recovered)
- If storage was increased: CANNOT be rolled back — storage increases are permanent in MSK
- If auto-scaling was enabled: disable auto-scaling policy if not desired, but storage already scaled cannot shrink
- If topics were deleted: CANNOT be rolled back — recreate topic with same configuration, but data is lost
- If compaction was enabled on a topic: revert cleanup.policy to "delete" if compaction is not desired

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
