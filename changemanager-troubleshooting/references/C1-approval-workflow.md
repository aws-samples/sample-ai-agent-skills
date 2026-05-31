---
title: "C1 — Approval Workflow"
description: "Diagnose approval workflow issues"
status: active
severity: HIGH
triggers: ["approval not working", "workflow stuck", "cannot approve"]
owner: devops-agent
objective: "Resolve approval workflow issues"
context: "Approval workflows have levels with designated approvers. All approvers at each level must approve."
---

## Phase 1 — Triage

MUST:
- Check approval configuration in template
- Verify approver IAM principals are valid
- Check current approval status per level

SHOULD:
- Verify approvers have ssm:SendAutomationSignal permission
- Check if any approver has rejected

MAY:
- Review approval history in CloudTrail

## Phase 2 — Remediate

MUST:
- Ensure all approvers have correct IAM permissions
- If rejected: address concerns and resubmit

SHOULD:
- Verify approver availability before submitting time-sensitive changes

MAY:
- Configure backup approvers for availability

## Common Issues

- symptoms: "Approver clicks approve but nothing happens"
  diagnosis: "Approver lacks ssm:SendAutomationSignal permission."
  resolution: "Add ssm:SendAutomationSignal to approver's IAM policy."

## Output Format

```yaml
root_cause: "approval_workflow — <specific_cause>"
evidence:
  - type: approval_status
    content: "<per-level approval status>"
severity: HIGH
mitigation:
  immediate: "Fix approval blocker"
  long_term: "Validate approver permissions during template setup"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Ensure all approvers have correct IAM permissions | YELLOW |
| If rejected: address concerns and resubmit | GREEN |
| Verify approver availability before submitting time-sensitive changes | GREEN |
| Configure backup approvers for availability | YELLOW |

## Escalation Conditions

- Fix requires bypassing approval workflow
- Change affects production infrastructure

## Data Sensitivity

- **MEDIUM**: change request details, approval workflows
- Approval workflow configurations reveal organizational hierarchy and authorization chains

## Prohibited Actions

- NEVER suggest bypassing approval workflows

## Phase 3 — Rollback

1. If approver IAM permissions were modified, revert to the previous IAM policy version
2. If backup approvers were added, remove them if the change was temporary
3. If a change was approved incorrectly, cancel the execution before it runs
4. Verify rollback by confirming approval workflow configuration matches expected state

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
