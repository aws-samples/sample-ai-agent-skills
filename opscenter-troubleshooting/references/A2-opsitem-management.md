---
title: "A2 — OpsItem Management"
description: "Diagnose OpsItem management and status issues"
status: active
severity: MEDIUM
triggers: ["OpsItem stuck", "cannot update", "status not changing"]
owner: devops-agent
objective: "Resolve OpsItem management issues"
context: "OpsItems have status (Open/InProgress/Resolved), severity, and operational data. Management issues include status updates, filtering, and lifecycle."
---

## Phase 1 — Triage

MUST:
- Get OpsItem details: `aws ssm get-ops-item --ops-item-id <id>`
- Check current status and severity
- Verify IAM permissions for ssm:UpdateOpsItem

SHOULD:
- Review OpsItem history for status changes
- Check if OpsItem is locked or restricted

MAY:
- List all open OpsItems: `aws ssm describe-ops-items --ops-item-filters "Key=Status,Values=Open,Operator=Equal"`

## Phase 2 — Remediate

MUST:
- Update status: `aws ssm update-ops-item --ops-item-id <id> --status Resolved`
- Ensure IAM permissions allow updates

SHOULD:
- Implement OpsItem lifecycle automation
- Set up regular review of open OpsItems

MAY:
- Create dashboards for OpsItem metrics

## Common Issues

- symptoms: "Cannot update OpsItem status"
  diagnosis: "IAM policy missing ssm:UpdateOpsItem permission."
  resolution: "Add ssm:UpdateOpsItem to IAM policy."

## Output Format

```yaml
root_cause: "opsitem_management — <specific_cause>"
evidence:
  - type: opsitem_status
    content: "<current status and update attempts>"
severity: MEDIUM
mitigation:
  immediate: "Fix management issue"
  long_term: "Automate OpsItem lifecycle"
```


## Safety Ratings
- Overall Risk: LOW
- Data Exposure Risk: MEDIUM — OpsItems may contain operational details about infrastructure
- Service Disruption Risk: LOW — OpsItem management does not affect running resources
- Compliance Impact: MEDIUM — OpsItem tracking supports audit and compliance requirements

## Escalation Conditions
- OpsItems cannot be updated or resolved
- OpsItem status transitions fail
- Bulk OpsItem operations cause service throttling
- OpsItem data integrity issues detected
- Cross-account OpsItem sharing fails

## Data Sensitivity
- OpsItem operational data — INTERNAL
- Resolution notes and remediation steps — INTERNAL
- Related resource details — INTERNAL
- Cross-account sharing configurations — INTERNAL
- Never expose internal operational details in shared OpsItems

## Prohibited Actions
- Do NOT delete OpsItems that are part of active incident investigations
- Do NOT modify OpsItem severity without proper assessment
- Do NOT share OpsItems containing sensitive data across accounts without review
- Do NOT bypass OpsItem workflow status transitions
- Do NOT ignore unresolved critical OpsItems
- Do NOT use OpsItems to store credentials or secrets

## Phase 3 — Rollback
- Restore OpsItem status to previous state if incorrectly modified
- Revert OpsItem data changes if incorrect updates were applied
- Re-enable cross-account sharing if accidentally disabled
- Restore OpsItem notification configurations
- Verify OpsItem workflows function correctly after changes
- Document OpsItem management changes and their rationale

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
