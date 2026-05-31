---
title: "B2 — Stalled Replication"
description: "Diagnose stalled or disconnected replication"
status: active
severity: CRITICAL
triggers:
  - "replication stalled"
  - "replication stopped"
  - "disconnected source"
owner: devops-agent
objective: "Restore stalled replication to active state"
context: "Replication can stall due to network interruptions, agent crashes, replication server issues, or staging area problems. A STALLED or DISCONNECTED state means data is NOT being replicated."
---

## Phase 1 — Triage

MUST:
- Check replication state: `aws drs describe-source-servers --filters 'sourceServerIDs=<id>' --query 'items[0].dataReplicationInfo.dataReplicationState'`
- Verify agent is running on source server
- Check network connectivity from source to replication server (TCP 1500)
- Check replication server EC2 instance status

SHOULD:
- Review agent logs for error messages
- Check CloudTrail for DisconnectFromService events
- Verify staging area subnet routing and security groups

MAY:
- Check if source server was rebooted or had network interruption
- Verify no AWS service disruption in the target region

## Phase 2 — Remediate

MUST:
- Restart agent on source server if it crashed
- Restore network connectivity if interrupted
- If replication server terminated, DRS should auto-replace — wait and monitor

SHOULD:
- Rescan disks if disk configuration changed: `aws drs retry-data-replication --source-server-id <id>`
- Check for and resolve any staging area capacity issues

MAY:
- Disconnect and reconnect source server as last resort

## Common Issues

- symptoms: "State shows DISCONNECTED after source server reboot"
  diagnosis: "Agent did not auto-start after reboot."
  resolution: "Ensure agent service is set to auto-start. Restart agent manually."

## Output Format

```yaml
root_cause: "stalled_replication — <specific_cause>"
evidence:
  - type: replication_state
    content: "<current state and last sync time>"
severity: CRITICAL
mitigation:
  immediate: "Restore replication to active state"
  long_term: "Configure agent auto-start and monitoring alerts for replication state"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | CRITICAL — Stalled replication means data is NOT being protected |
| Blast Radius | CRITICAL — Stalled replication means RPO is not being met |
| Rollback Complexity | MEDIUM — May require agent restart or rescan |
| Compliance Scope | CRITICAL — Stalled replication violates DR compliance requirements |

## Escalation Conditions

- Escalate IMMEDIATELY to DR coordinator — RPO is not being met
- Escalate to operations if agent restart doesn't resolve the stall
- Escalate to AWS Support if replication server issues persist

## Data Sensitivity

- **Replication state** indicates DR readiness — critical operational data
- **Source server data** is not being replicated — data loss risk

## PROHIBITED Actions

- Do NOT ignore STALLED or DISCONNECTED replication states
- Do NOT disconnect source servers without understanding the data loss implications
- Do NOT assume replication will auto-recover without investigation

## Phase 3 — Rollback

- Restart the agent on the source server
- Retry data replication: `aws drs retry-data-replication --source-server-id <id>`
- If replication cannot be restored, disconnect and reconnect the source server as last resort
- Document the replication gap for DR compliance records

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
