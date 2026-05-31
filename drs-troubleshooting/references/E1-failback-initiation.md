---
title: "E1 — Failback Initiation"
description: "Diagnose failback initiation failures"
status: active
severity: HIGH
triggers:
  - "failback not starting"
  - "cannot initiate failback"
  - "failback client error"
owner: devops-agent
objective: "Resolve failback initiation issues"
context: "Failback returns workloads from AWS recovery instances back to the original source infrastructure. It requires the Failback Client installed on the recovery instance and connectivity to the original source."
---

## Phase 1 — Triage

MUST:
- Check recovery instance failback state: `aws drs describe-recovery-instances --filters 'sourceServerIDs=<id>' --query 'items[0].failback'`
- Verify Failback Client is installed on recovery instance
- Check connectivity from recovery instance to original source infrastructure

SHOULD:
- Verify original source infrastructure is available and accessible
- Check IAM permissions for failback operations
- Review CloudTrail for StartFailbackLaunch events

MAY:
- Verify network routing between AWS VPC and on-premises network
- Check VPN/Direct Connect connectivity if applicable

## Phase 2 — Remediate

MUST:
- Install Failback Client on recovery instance if not present
- Ensure network path from recovery instance to source is available
- Verify source infrastructure has capacity for failback

SHOULD:
- Test connectivity before initiating failback
- Plan failback during maintenance window

MAY:
- Configure bandwidth throttling for failback replication

## Common Issues

- symptoms: "Failback client cannot connect to source"
  diagnosis: "Network connectivity between AWS and on-premises is broken."
  resolution: "Verify VPN/DX connectivity and firewall rules allow failback traffic."

## Output Format

```yaml
root_cause: "failback_initiation — <specific_cause>"
evidence:
  - type: failback_state
    content: "<failback status and client connectivity>"
severity: HIGH
mitigation:
  immediate: "Restore connectivity and retry failback"
  long_term: "Include failback testing in regular DR drills"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — Failback transfers data from AWS back to on-premises |
| Blast Radius | HIGH — Failback affects production workload location |
| Rollback Complexity | HIGH — Failback is a major operational event |
| Compliance Scope | HIGH — Failback restores normal operations after DR event |

## Escalation Conditions

- Escalate to management before initiating failback — it's a major operational decision
- Escalate to network team if connectivity to on-premises is not available
- Escalate to application teams to coordinate failback timing

## Data Sensitivity

- **Failback data** is production data being transferred back to on-premises
- **Network connectivity** between AWS and on-premises must be secure

## PROHIBITED Actions

- Do NOT initiate failback without management approval
- Do NOT failback during business hours without coordination
- Do NOT assume on-premises infrastructure is ready without verification

## Phase 3 — Rollback

- If failback was initiated incorrectly, stop the failback process
- Continue running on AWS recovery instances until failback can be properly planned
- Verify on-premises infrastructure readiness before retrying

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
