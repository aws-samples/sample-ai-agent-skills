---
title: "B2 — Request Status"
description: "Diagnose change request status issues"
status: active
severity: MEDIUM
triggers: ["request status", "request stuck", "request not progressing"]
owner: devops-agent
objective: "Resolve change request status issues"
context: "Change requests progress through states: Pending → Approved → Scheduled/Executing → Completed/Failed/Rejected."
---

## Phase 1 — Triage

MUST:
- Check request status: `aws ssm get-ops-item --ops-item-id <id>`
- Review approval status for each level
- Check if blocked by Change Calendar

SHOULD:
- Verify all approvers have responded
- Check execution status if approved

MAY:
- Review request timeline and history

## Phase 2 — Remediate

MUST:
- If pending approval: follow up with approvers
- If blocked by calendar: wait for open window or use emergency template
- If execution failed: check runbook execution logs

SHOULD:
- Set up monitoring for request state changes

MAY:
- Configure EventBridge rules for status notifications

## Common Issues

- symptoms: "Request approved but not executing"
  diagnosis: "Change Calendar is in CLOSED state blocking execution."
  resolution: "Wait for calendar open window or use emergency change template."

## Output Format

```yaml
root_cause: "request_status — <specific_cause>"
evidence:
  - type: request_state
    content: "<current status and blockers>"
severity: MEDIUM
mitigation:
  immediate: "Address specific blocker"
  long_term: "Implement status monitoring and alerting"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| If pending approval: follow up with approvers | GREEN |
| If blocked by calendar: wait for open window or use emergency template | YELLOW |
| If execution failed: check runbook execution logs | GREEN |
| Set up monitoring for request state changes | GREEN |
| Configure EventBridge rules for status notifications | GREEN |

## Escalation Conditions

- Fix requires bypassing approval workflow
- Change affects production infrastructure

## Data Sensitivity

- **MEDIUM**: change request details, approval workflows
- Request status history reveals change management processes and timelines

## Prohibited Actions

- NEVER suggest bypassing approval workflows

## Phase 3 — Rollback

1. If an emergency template was used to bypass calendar restrictions, document the justification and review
2. If EventBridge rules were created, delete them if no longer needed
3. If a request was force-progressed, review the change outcome and revert infrastructure changes if needed
4. Verify rollback by confirming request status matches expected state

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
