---
title: "F2 — Drill Cleanup"
description: "Diagnose drill instance cleanup issues"
status: active
severity: MEDIUM
triggers:
  - "drill instances still running"
  - "cleanup drill"
  - "terminate drill instances"
owner: devops-agent
objective: "Properly clean up drill test instances"
context: "Drill instances are real EC2 instances that incur costs. They must be explicitly terminated after testing. DRS does not auto-terminate drill instances."
---

## Phase 1 — Triage

MUST:
- List recovery instances: `aws drs describe-recovery-instances`
- Identify drill instances (check isDrill flag in job)
- Check EC2 instances with DRS tags: `aws ec2 describe-instances --filters "Name=tag:aws:elasticdisasterrecovery:source-server-id,Values=*" --query 'Reservations[*].Instances[*].{Id:InstanceId,State:State.Name,Launch:LaunchTime}'`

SHOULD:
- Verify which instances are drill vs actual recovery
- Check associated EBS volumes and snapshots

MAY:
- Calculate ongoing costs of running drill instances

## Phase 2 — Remediate

MUST:
- Terminate drill instances: `aws drs terminate-recovery-instances --recovery-instance-ids <id>`
- Verify termination completed successfully

SHOULD:
- Clean up associated EBS snapshots if no longer needed
- Document drill results before cleanup

MAY:
- Set up automated cleanup with Lambda/EventBridge
- Tag drill instances with expiration dates

## Common Issues

- symptoms: "Multiple old drill instances running for weeks"
  diagnosis: "Drill instances not cleaned up after testing."
  resolution: "Terminate via DRS API. Implement cleanup automation."

## Output Format

```yaml
root_cause: "drill_cleanup — <specific_cause>"
evidence:
  - type: drill_instances
    content: "<list of running drill instances and ages>"
severity: MEDIUM
mitigation:
  immediate: "Terminate stale drill instances"
  long_term: "Automate drill cleanup with Lambda triggers"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — Drill instances contain production data |
| Blast Radius | LOW — Cleanup affects only drill instances |
| Rollback Complexity | LOW — Drill instances can be relaunched if needed |
| Compliance Scope | MEDIUM — Proper cleanup prevents unnecessary costs and data exposure |

## Escalation Conditions

- Escalate to finance if stale drill instances are incurring significant costs
- Escalate to security if drill instances with production data are running without security controls

## Data Sensitivity

- **Drill instances** contain replicated production data — secure until terminated
- **EBS volumes** from drill instances may persist after termination

## PROHIBITED Actions

- Do NOT leave drill instances running indefinitely — they contain production data and incur costs
- Do NOT terminate drill instances without documenting drill results first
- Do NOT leave EBS snapshots from drills without cleanup

## Phase 3 — Rollback

- If drill instances were terminated prematurely, relaunch a new drill
- Clean up associated EBS volumes and snapshots after termination
- Implement automated cleanup with Lambda/EventBridge for future drills

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
