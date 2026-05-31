---
title: "E2 — Failback Replication"
description: "Diagnose failback replication issues"
status: active
severity: HIGH
triggers:
  - "failback replication slow"
  - "failback sync not completing"
  - "failback data transfer"
owner: devops-agent
objective: "Resolve failback replication problems"
context: "Failback replication copies data from the AWS recovery instance back to the original source. Similar to forward replication, it depends on bandwidth and data volume."
---

## Phase 1 — Triage

MUST:
- Check failback replication state: `aws drs describe-recovery-instances --filters 'sourceServerIDs=<id>' --query 'items[0].failback.state'`
- Monitor failback replication progress
- Verify bandwidth between AWS and source infrastructure

SHOULD:
- Check for network interruptions during failback
- Verify source infrastructure disk space is sufficient

MAY:
- Monitor data transfer rates over time

## Phase 2 — Remediate

MUST:
- Ensure stable network connectivity throughout failback
- Verify source has sufficient disk space for restored data

SHOULD:
- Schedule failback during low-traffic periods
- Monitor and address any bandwidth bottlenecks

MAY:
- Increase VPN/DX bandwidth for faster failback

## Common Issues

- symptoms: "Failback replication stalls at 80%"
  diagnosis: "Network interruption or source disk full."
  resolution: "Check connectivity and disk space on source. Resume failback."

## Output Format

```yaml
root_cause: "failback_replication — <specific_cause>"
evidence:
  - type: failback_progress
    content: "<failback replication state and progress>"
severity: HIGH
mitigation:
  immediate: "Address the specific replication blocker"
  long_term: "Ensure adequate bandwidth and storage for failback operations"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — Failback replication transfers production data to on-premises |
| Blast Radius | HIGH — Failback replication failures delay return to normal operations |
| Rollback Complexity | MEDIUM — Failback can be restarted but requires stable connectivity |
| Compliance Scope | HIGH — Failback completion restores normal DR posture |

## Escalation Conditions

- Escalate to network team if connectivity issues stall failback
- Escalate to operations if source infrastructure disk space is insufficient
- Escalate to management if failback timeline is at risk

## Data Sensitivity

- **Failback data** is production data — ensure encrypted transfer
- **Source infrastructure** must be secured before receiving data

## PROHIBITED Actions

- Do NOT interrupt failback replication without understanding the restart implications
- Do NOT failback to infrastructure that hasn't been verified as ready
- Do NOT reduce VPN/DX bandwidth during active failback

## Phase 3 — Rollback

- If failback replication fails, continue running on AWS recovery instances
- Fix the connectivity or capacity issue and restart failback
- Verify data integrity on the source after failback completes

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
