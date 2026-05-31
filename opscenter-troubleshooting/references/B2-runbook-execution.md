---
title: "B2 — Runbook Execution"
description: "Diagnose runbook execution failures from OpsItems"
status: active
severity: HIGH
triggers: ["runbook failed", "execution error", "automation failed"]
owner: devops-agent
objective: "Resolve runbook execution failures"
context: "Runbooks execute as SSM Automation. Failures are typically IAM, parameter, or resource access issues."
---

## Phase 1 — Triage

MUST:
- Check execution status: `aws ssm describe-automation-executions --filters "Key=ExecutionId,Values=<id>"`
- Review step failures: `aws ssm describe-automation-step-executions --automation-execution-id <id>`
- Verify AutomationAssumeRole has required permissions

SHOULD:
- Check target resource state
- Review CloudTrail for API call failures during execution

MAY:
- Test runbook with dry-run parameters

## Phase 2 — Remediate

MUST:
- Fix IAM role permissions for failed actions
- Ensure target resources are accessible

SHOULD:
- Add error handling to runbook steps
- Test runbooks in non-production first

MAY:
- Implement approval steps for destructive actions

## Common Issues

- symptoms: "Automation step fails with AccessDenied"
  diagnosis: "AutomationAssumeRole missing required service permissions."
  resolution: "Add missing permissions to the automation role."

## Output Format

```yaml
root_cause: "runbook_execution — <specific_cause>"
evidence:
  - type: execution_details
    content: "<step execution status and errors>"
severity: HIGH
mitigation:
  immediate: "Fix execution failure"
  long_term: "Test and validate runbooks regularly"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: MEDIUM — execution may access sensitive resource configurations
- Service Disruption Risk: HIGH — failed executions can leave resources in bad state
- Compliance Impact: HIGH — execution audit trail is required for compliance

## Escalation Conditions
- Runbook execution fails with permission errors on critical resources
- Execution hangs or times out without completing
- Execution produces unexpected results or side effects
- Multiple runbook executions fail simultaneously
- Execution causes cascading failures across resources

## Data Sensitivity
- Execution input parameters — INTERNAL (may contain CONFIDENTIAL data)
- Execution output and logs — INTERNAL
- Resource state changes — INTERNAL
- Execution role credentials — RESTRICTED
- Never expose execution parameters containing secrets

## Prohibited Actions
- Do NOT retry failed executions without understanding the failure cause
- Do NOT execute runbooks with elevated permissions without approval
- Do NOT ignore execution timeout warnings
- Do NOT run multiple conflicting runbooks simultaneously
- Do NOT bypass execution approval workflows
- Do NOT share execution logs containing sensitive data

## Phase 3 — Rollback
- Cancel stuck or hanging executions
- Revert resource changes from failed executions manually
- Restore execution role permissions to previous state
- Re-run the runbook after fixing the root cause
- Verify all affected resources are in correct state
- Document execution failures and update runbook error handling

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
