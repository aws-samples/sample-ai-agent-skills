---
title: "B1 — Replication Lag"
description: "Diagnose replication lag exceeding RPO requirements"
status: active
severity: HIGH
triggers:
  - "replication lag"
  - "RPO exceeded"
  - "data not synced"
owner: devops-agent
objective: "Identify and resolve replication lag issues"
context: "DRS continuously replicates changed blocks from source to target. Lag indicates the time difference between source writes and target replication. Sustained lag exceeding RPO is a DR readiness concern."
---

## Phase 1 — Triage

MUST:
- Check lag duration: `aws drs describe-source-servers --filters 'sourceServerIDs=<id>' --query 'items[0].dataReplicationInfo.lagDuration'`
- Check replication state: `aws drs describe-source-servers --filters 'sourceServerIDs=<id>' --query 'items[0].dataReplicationInfo.dataReplicationState'`
- Verify network bandwidth between source and staging area

SHOULD:
- Check source server disk I/O rate (high write rate increases lag)
- Verify replication server instance type is adequate
- Check staging area S3 bucket for throttling

MAY:
- Monitor CloudWatch metrics for DRS replication
- Check for network congestion on source server

## Phase 2 — Remediate

MUST:
- If bandwidth is insufficient, configure bandwidth throttling to prevent saturation
- Upgrade replication server instance type if CPU/network bound

SHOULD:
- Reduce source server write rate during initial catch-up
- Verify no security group or NACL changes blocking TCP 1500

MAY:
- Consider dedicated replication network path
- Adjust replication configuration for better performance

## Common Issues

- symptoms: "Lag steadily increasing over time"
  diagnosis: "Source write rate exceeds replication bandwidth."
  resolution: "Increase bandwidth or upgrade replication server instance type."

## Output Format

```yaml
root_cause: "replication_lag — <specific_cause>"
evidence:
  - type: lag_duration
    content: "<current lag and trend>"
severity: HIGH
mitigation:
  immediate: "Address bandwidth or replication server bottleneck"
  long_term: "Right-size replication infrastructure for sustained write rates"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — Replication lag indicates potential data loss during failover |
| Blast Radius | HIGH — Sustained lag means RPO is not being met |
| Rollback Complexity | LOW — Bandwidth and replication server changes can be reverted |
| Compliance Scope | CRITICAL — RPO compliance requires minimal replication lag |

## Escalation Conditions

- Escalate to DR coordinator if lag exceeds RPO thresholds
- Escalate to network team if bandwidth is the bottleneck
- Escalate to management if replication server upgrades are needed

## Data Sensitivity

- **Lag duration** indicates DR readiness — critical operational data
- **Bandwidth utilization** reveals network capacity

## PROHIBITED Actions

- Do NOT ignore sustained replication lag — it means data loss during failover
- Do NOT throttle replication bandwidth below what's needed to maintain RPO
- Do NOT change replication server instance type during active replication without planning

## Phase 3 — Rollback

- Restore previous bandwidth throttling settings
- Revert replication server instance type if the change caused issues
- Monitor lag after rollback to confirm it returns to acceptable levels

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
