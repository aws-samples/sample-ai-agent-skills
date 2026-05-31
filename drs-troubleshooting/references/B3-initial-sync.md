---
title: "B3 — Initial Sync Issues"
description: "Diagnose initial replication sync problems"
status: active
severity: HIGH
triggers:
  - "initial sync slow"
  - "initial sync stuck"
  - "first replication not completing"
owner: devops-agent
objective: "Resolve initial sync completion issues"
context: "Initial sync copies all source disk data to the target. Duration depends on total data volume and bandwidth. A 1TB disk on 100Mbps takes ~24 hours. Initial sync must complete before continuous replication begins."
---

## Phase 1 — Triage

MUST:
- Check sync state: `aws drs describe-source-servers --filters 'sourceServerIDs=<id>' --query 'items[0].dataReplicationInfo'`
- Verify dataReplicationState is INITIAL_SYNC (not STALLED)
- Check etaDateTime for estimated completion
- Verify replicatedDisks shows progress for each disk

SHOULD:
- Calculate expected duration based on data volume and bandwidth
- Check replication server network throughput

MAY:
- Monitor data transfer rate over time to detect degradation

## Phase 2 — Remediate

MUST:
- If state is STALLED, follow B2-stalled-replication runbook
- Ensure adequate bandwidth for the data volume

SHOULD:
- Use larger replication server instance type for faster initial sync
- Schedule initial sync during low-traffic periods

MAY:
- Temporarily increase bandwidth allocation for initial sync

## Common Issues

- symptoms: "Initial sync at 10% for 3 days on 500GB server"
  diagnosis: "Bandwidth limited — likely 10Mbps effective throughput."
  resolution: "Check and increase network bandwidth. Consider larger replication server."

## Output Format

```yaml
root_cause: "initial_sync — <specific_cause>"
evidence:
  - type: sync_progress
    content: "<sync state, progress percentage, ETA>"
severity: HIGH
mitigation:
  immediate: "Address bandwidth or connectivity bottleneck"
  long_term: "Pre-calculate sync times and plan accordingly"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — Initial sync copies all source server data to AWS |
| Blast Radius | MEDIUM — Slow initial sync delays DR readiness |
| Rollback Complexity | LOW — Bandwidth and replication server changes can be adjusted |
| Compliance Scope | HIGH — Initial sync must complete before DR protection is active |

## Escalation Conditions

- Escalate to DR coordinator if initial sync is not completing within expected timeframes
- Escalate to network team if bandwidth is the bottleneck
- Escalate to management if replication server upgrades are needed for faster sync

## Data Sensitivity

- **Replicated data** is a complete copy of source server disks — highest sensitivity
- **Sync progress** reveals data volume and transfer rates

## PROHIBITED Actions

- Do NOT assume DR protection is active until initial sync completes
- Do NOT interrupt initial sync without understanding the restart implications
- Do NOT throttle bandwidth during initial sync unless network saturation is a concern

## Phase 3 — Rollback

- If initial sync is stalled, follow B2-stalled-replication runbook
- Adjust bandwidth throttling if it's limiting sync speed
- Upgrade replication server instance type for faster sync

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
