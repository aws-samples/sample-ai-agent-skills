---
title: "F2 — Latency"
description: "Diagnose latency issues in DMS migrations"
status: active
severity: MEDIUM
triggers:
  - "high latency"
  - "CDC delay"
  - "replication delay"
  - "target behind"
owner: devops-agent
objective: "Identify and reduce latency in DMS data migration"
context: "Latency in DMS has two components: source latency (time to read changes) and target latency (time to apply changes). High latency means the target is behind the source. Network distance, instance sizing, and apply efficiency all contribute."
---

## Phase 1 — Triage

MUST:
- Check source latency: `aws cloudwatch get-metric-statistics --namespace AWS/DMS --metric-name CDCLatencySource --dimensions Name=ReplicationInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check target latency: `aws cloudwatch get-metric-statistics --namespace AWS/DMS --metric-name CDCLatencyTarget --dimensions Name=ReplicationInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check instance resources: `aws cloudwatch get-metric-statistics --namespace AWS/DMS --metric-name CPUUtilization --dimensions Name=ReplicationInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Average`

SHOULD:
- Determine if latency is on source side or target side
- Check if source and replication instance are in the same region/AZ
- Verify target is not resource-constrained

MAY:
- Check network latency between replication instance and endpoints
- Review if large transactions are causing latency spikes

## Phase 2 — Remediate

MUST:
- If source latency high: check source connectivity, supplemental logging, log retention
- If target latency high: enable BatchApplyEnabled, optimize target indexes
- Scale up instance if resource-constrained

SHOULD:
- Place replication instance in the same region as source and target
- Increase ParallelApplyThreads for CDC
- Remove unnecessary target indexes during migration

MAY:
- Use separate tasks for high-volume tables
- Optimize source queries by tuning extra connection attributes

## Common Issues

- symptoms: "CDCLatencySource steadily increasing"
  diagnosis: "Source log reading falling behind change rate."
  resolution: "Scale up instance. Check source log retention. Optimize log reader settings."

- symptoms: "CDCLatencyTarget spikes during peak hours"
  diagnosis: "Target cannot apply changes fast enough during high-volume periods."
  resolution: "Enable BatchApplyEnabled. Scale target database. Add apply threads."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Scale up replication instance | YELLOW | Instance modification — brief downtime |
| Enable BatchApplyEnabled | YELLOW | Changes apply behavior — may affect ordering |
| Increase ParallelApplyThreads | GREEN | Task setting — applied on restart |
| Remove unnecessary target indexes | YELLOW | Schema change on target — rebuild after |
| Place instance in same region as endpoints | RED | Requires new instance — migrate tasks |

## Escalation Conditions

- Task replicates production database
- Fix requires restarting full load
- High latency delays production cutover timeline
- Target database serves live traffic and index changes affect performance

## Data Sensitivity

- **Classification: HIGH**
- Source/target latency metrics reveal replication health and data freshness
- Instance resource metrics expose infrastructure sizing
- Network latency reveals geographic distribution of source and target
- CDC throughput reveals data change rates in production databases

## Prohibited Actions

- **NEVER** suggest deleting a replication task without confirming target data is current
- **NEVER** suggest modifying source endpoint during active CDC
- **NEVER** suggest dropping target indexes on a database serving live traffic
- **NEVER** suggest reducing source change rate by modifying production applications

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Scaled up replication instance | Scale down after latency is resolved |
| Enabled BatchApplyEnabled | Disable batch apply if it causes ordering issues |
| Removed target indexes | Rebuild target indexes after migration completes |
| Created new instance in different region | Delete old instance after tasks are migrated |

## Output Format

```yaml
root_cause: "latency — <specific_cause>"
evidence:
  - type: source_latency
    content: "<CDCLatencySource metrics>"
  - type: target_latency
    content: "<CDCLatencyTarget metrics>"
severity: MEDIUM
mitigation:
  immediate: "Address the latency bottleneck"
  long_term: "Optimize task settings and infrastructure placement"
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
  - "NEVER suggest disabling SSL for replication endpoints"
  - "NEVER suggest public replication instances"
  - "NEVER suggest deleting replication tasks without data verification"
