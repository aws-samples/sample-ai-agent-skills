---
title: "D1 — Runbook Execution"
description: "Diagnose runbook execution issues during changes"
status: active
severity: HIGH
triggers: ["runbook failed", "automation failed", "execution error"]
owner: devops-agent
objective: "Resolve runbook execution failures"
context: "Change requests execute SSM Automation runbooks. Failures can be due to IAM, resource access, or runbook logic errors."
---

## Phase 1 — Triage

MUST:
- Check automation execution: `aws ssm describe-automation-executions --filters "Key=ExecutionId,Values=<id>"`
- Review step execution details: `aws ssm describe-automation-step-executions --automation-execution-id <id>`
- Check which step failed and the failure message

SHOULD:
- Verify automation service role has required permissions
- Check target resource state

MAY:
- Review runbook document for logic errors

## Phase 2 — Remediate

MUST:
- Fix the specific step failure (IAM, resource, logic)
- Manually complete any partial changes if needed

SHOULD:
- Test runbook independently before using in change requests
- Add error handling to runbook steps

MAY:
- Implement rollback steps in the runbook

## Common Issues

- symptoms: "Automation step fails with AccessDenied"
  diagnosis: "Automation service role missing required permissions."
  resolution: "Add missing permissions to the automation assume role."

## Output Format

```yaml
root_cause: "runbook_execution — <specific_cause>"
evidence:
  - type: execution_details
    content: "<step execution status and errors>"
severity: HIGH
mitigation:
  immediate: "Fix execution failure and complete change"
  long_term: "Test runbooks thoroughly before production use"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Fix the specific step failure (IAM, resource, logic) | YELLOW |
| Manually complete any partial changes if needed | RED |
| Test runbook independently before using in change requests | GREEN |
| Add error handling to runbook steps | GREEN |
| Implement rollback steps in the runbook | GREEN |

## Escalation Conditions

- Fix requires bypassing approval workflow
- Change affects production infrastructure

## Data Sensitivity

- **MEDIUM**: change request details, approval workflows
- Runbook execution logs may contain resource identifiers and configuration details for production infrastructure

## Prohibited Actions

- NEVER suggest bypassing approval workflows

## Phase 3 — Rollback

1. If a runbook step partially completed, manually revert the changes made by completed steps
2. If IAM permissions were modified for the automation role, revert to the previous policy
3. If the runbook was modified, restore the previous version of the SSM document
4. Document all manual changes made during recovery for audit trail
5. Verify rollback by checking target resource state matches pre-change baseline

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
