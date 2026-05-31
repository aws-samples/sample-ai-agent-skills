---
title: "B3 — Finding Workflow Issues"
description: "Diagnose Security Hub finding workflow status and lifecycle issues"
status: active
severity: MEDIUM
triggers:
  - "finding workflow"
  - "finding status"
  - "finding archived"
  - "finding resolved"
  - "workflow status"
owner: devops-agent
objective: "Identify and resolve finding workflow status and lifecycle management issues"
context: "Security Hub findings have a workflow status (NEW, NOTIFIED, SUPPRESSED, RESOLVED) that tracks investigation progress. Issues arise from incorrect status transitions, automated status changes conflicting with manual updates, or findings not aging out as expected."
---

## Phase 1 — Triage

MUST:
- Get findings with specific workflow status: `aws securityhub get-findings --filters '{"WorkflowStatus":[{"Value":"NEW","Comparison":"EQUALS"}]}' --max-results 20`
- Check finding details including workflow status and record state
- Verify automation rules that may change workflow status: `aws securityhub list-automation-rules`
- Check if findings are being updated by their source product

SHOULD:
- Review finding update history (UpdatedAt timestamps)
- Check for automation rules that auto-suppress or auto-resolve findings
- Verify the finding's RecordState (ACTIVE vs ARCHIVED)

MAY:
- Check CloudTrail for BatchUpdateFindings events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=BatchUpdateFindings --max-results 10`
- Review finding retention (90 days after last update)

## Phase 2 — Remediate

MUST:
- Update finding workflow status as appropriate: `aws securityhub batch-update-findings --finding-identifiers '[{"Id":"<id>","ProductArn":"<arn>"}]' --workflow '{"Status":"RESOLVED"}'`
- Fix automation rules that incorrectly change workflow status
- Ensure finding lifecycle management aligns with security processes

SHOULD:
- Create automation rules for common workflow transitions
- Document the finding workflow process for the team

MAY:
- Set up notifications for finding status changes
- Create dashboards for finding workflow metrics

## Common Issues

- symptoms: "Findings stuck in NEW status despite being investigated"
  diagnosis: "No one or no automation is updating the workflow status."
  resolution: "Update workflow status manually or create automation rules to transition findings based on criteria."

- symptoms: "Findings being auto-suppressed unexpectedly"
  diagnosis: "An automation rule is matching and suppressing findings that should remain active."
  resolution: "Review automation rules and narrow their criteria to avoid suppressing legitimate findings."

- symptoms: "Resolved findings reappearing as NEW"
  diagnosis: "The source product is generating new findings for the same issue, or the finding is being updated."
  resolution: "Fix the underlying issue so the source stops generating findings. Use suppression for known false positives."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Update finding workflow status | YELLOW | State change; reversible |
| Fix automation rules | YELLOW | Rule modification; reversible |

## Escalation Conditions

- Cross-account configuration changes
- Automation rule changes affecting finding lifecycle

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-findings` (workflow status) | HIGH | Finding details |
| `list-automation-rules` | LOW | Rule names and ARNs |
| CloudTrail events | MEDIUM | API call history |

## Prohibited Actions

- NEVER suggest disabling Security Hub
- NEVER suggest disabling all security standards
- NEVER suggest auto-archiving all findings

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Workflow status update | Revert via `batch-update-findings` |
| Automation rule fix | Revert via `batch-update-automation-rules` |

## Output Format

```yaml
root_cause: "finding_workflow — <specific_cause>"
evidence:
  - type: findings
    content: "<finding workflow status details>"
  - type: automation_rules
    content: "<automation rules affecting workflow>"
severity: MEDIUM
mitigation:
  immediate: "Update finding workflow status appropriately"
  long_term: "Implement automated workflow management with proper rules"
```

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
  - "NEVER suggest disabling security standards"
  - "NEVER suppress findings without investigation"
  - "NEVER remove security integrations"
