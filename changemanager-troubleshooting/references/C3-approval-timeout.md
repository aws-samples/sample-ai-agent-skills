---
title: "C3 — Approval Timeout"
description: "Diagnose approval timeout issues"
status: active
severity: MEDIUM
triggers: ["approval expired", "timeout", "approval window closed"]
owner: devops-agent
objective: "Resolve approval timeout issues"
context: "Change requests can have approval expiration configured. Expired requests must be resubmitted."
---

## Phase 1 — Triage

MUST:
- Check request status for expiration
- Review template approval timeout configuration
- Verify when request was submitted vs current time

SHOULD:
- Check if approvers were notified in time
- Review approval SLA expectations

MAY:
- Check historical approval response times

## Phase 2 — Remediate

MUST:
- Resubmit expired change requests
- Ensure approvers are available before submission

SHOULD:
- Adjust approval timeout in template if too short
- Notify approvers proactively before submission

MAY:
- Configure escalation for approaching timeouts

## Common Issues

- symptoms: "Change request expired before all approvers responded"
  diagnosis: "Approval timeout too short or approvers unavailable."
  resolution: "Increase timeout in template or coordinate with approvers before submission."

## Output Format

```yaml
root_cause: "approval_timeout — <specific_cause>"
evidence:
  - type: timeout_details
    content: "<submission time, timeout, expiration>"
severity: MEDIUM
mitigation:
  immediate: "Resubmit with coordinated approvers"
  long_term: "Adjust timeout and notification settings"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Resubmit expired change requests | GREEN |
| Ensure approvers are available before submission | GREEN |
| Adjust approval timeout in template if too short | YELLOW |
| Notify approvers proactively before submission | GREEN |
| Configure escalation for approaching timeouts | GREEN |

## Escalation Conditions

- Fix requires bypassing approval workflow
- Change affects production infrastructure

## Data Sensitivity

- **MEDIUM**: change request details, approval workflows
- Timeout patterns may reveal operational bottlenecks and approver responsiveness

## Prohibited Actions

- NEVER suggest bypassing approval workflows

## Phase 3 — Rollback

1. If approval timeout was changed in the template, restore the previous timeout value
2. If an expired request was resubmitted, cancel it if no longer needed
3. If escalation rules were added, remove them if the change was temporary
4. Verify rollback by confirming template configuration matches expected state

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
