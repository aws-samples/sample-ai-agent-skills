---
title: "A2 — Instance Issues"
description: "Diagnose Neptune instance health and availability problems"
status: active
severity: HIGH
triggers:
  - "instance not available"
  - "instance failed"
  - "neptune instance error"
  - "instance rebooting"
owner: devops-agent
objective: "Identify and resolve Neptune instance health issues"
context: "Neptune instances can experience failures due to hardware issues, maintenance events, storage full, out-of-memory conditions, or configuration problems. Instance class determines performance characteristics."
---

## Phase 1 — Triage

MUST:
- Describe instances: `aws neptune describe-db-instances --filters Name=db-cluster-id,Values=<cluster-id>`
- Check instance status and pending maintenance
- Check CPU utilization: `aws cloudwatch get-metric-statistics --namespace AWS/Neptune --metric-name CPUUtilization --dimensions Name=DBInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check freeable memory: `aws cloudwatch get-metric-statistics --namespace AWS/Neptune --metric-name FreeableMemory --dimensions Name=DBInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Minimum`

SHOULD:
- Check instance events: `aws neptune describe-events --source-type db-instance --source-identifier <instance-id> --duration 1440`
- Verify instance class is appropriate for workload
- Check for pending modifications: `aws neptune describe-db-instances --db-instance-identifier <instance-id>`
- Monitor network throughput: `aws cloudwatch get-metric-statistics --namespace AWS/Neptune --metric-name NetworkThroughput --dimensions Name=DBInstanceIdentifier,Value=<instance-id> --start-time <start> --end-time <end> --period 300 --statistics Average`

MAY:
- Check cluster status endpoint: `curl https://<cluster-endpoint>:8182/status`
- Review pending maintenance actions: `aws neptune describe-pending-maintenance-actions`
- Compare instance metrics across replicas

## Phase 2 — Remediate

MUST:
- Address the root cause (reboot, resize, or replace instance)
- Verify instance returns to available state
- Confirm cluster endpoint connectivity

SHOULD:
- Scale instance class if resource constrained
- Apply pending maintenance during planned window
- Ensure read replicas are healthy

MAY:
- Set up enhanced monitoring
- Configure CloudWatch alarms for instance metrics

## Common Issues

- symptoms: "Instance stuck in modifying state"
  diagnosis: "A modification operation is in progress or failed."
  resolution: "Wait for completion or contact AWS Support if stuck for extended period."

- symptoms: "High CPU with degraded performance"
  diagnosis: "Instance class is undersized for the workload or runaway query."
  resolution: "Scale to larger instance class or identify and optimize expensive queries."

- symptoms: "Instance rebooting repeatedly"
  diagnosis: "Out-of-memory condition or storage issue causing crash loop."
  resolution: "Scale to instance with more memory. Check VolumeBytesUsed metric."

## Output Format

```yaml
root_cause: "instance_issue — <specific_cause>"
evidence:
  - type: instance
    content: "<instance status and configuration>"
  - type: metrics
    content: "<CPU, memory, network metrics>"
  - type: events
    content: "<recent instance events>"
severity: HIGH
mitigation:
  immediate: "Resolve instance health issue"
  long_term: "Right-size instances and set up monitoring"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — instance issues are operational concerns
- Service Disruption Risk: HIGH — instance failures directly affect database availability
- Compliance Impact: LOW — instance issues do not directly affect compliance

## Escalation Conditions
- Primary instance fails and failover does not occur automatically
- Instance enters incompatible-parameters or incompatible-network state
- Instance performance degrades significantly without clear cause
- Instance class change causes unexpected downtime
- Multiple instances in the cluster become unhealthy simultaneously

## Data Sensitivity
- Instance endpoint addresses — INTERNAL
- Instance performance metrics — INTERNAL
- Parameter group configurations — INTERNAL
- Instance class and storage details — INTERNAL
- Never expose database credentials or connection strings

## Prohibited Actions
- Do NOT delete the primary instance without promoting a replica first
- Do NOT change instance class during peak query periods
- Do NOT modify instance parameters that require reboot without a maintenance window
- Do NOT ignore instance health check failures
- Do NOT run production workloads on unsupported instance classes
- Do NOT disable enhanced monitoring on production instances

## Phase 3 — Rollback
- Revert instance class changes if they caused performance issues
- Restore parameter group to previous configuration
- Failover to a healthy replica if the primary instance is degraded
- Re-enable monitoring if accidentally disabled
- Verify cluster health and replication status after instance changes
- Document instance changes and their impact on cluster performance

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
  - "NEVER suggest disabling IAM authentication"
  - "NEVER suggest public access"
  - "NEVER suggest disabling encryption at rest"
