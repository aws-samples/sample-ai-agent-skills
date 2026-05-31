---
title: "B1 — Automation Runbooks"
description: "Diagnose runbook association and configuration issues"
status: active
severity: HIGH
triggers: ["runbook not associated", "wrong runbook", "runbook config"]
owner: devops-agent
objective: "Resolve runbook association with OpsItems"
context: "OpsItems can have associated automation runbooks for guided remediation. Runbooks are SSM Automation documents."
---

## Phase 1 — Triage

MUST:
- Check OpsItem for associated runbook: `aws ssm get-ops-item --ops-item-id <id> --query 'OpsItem.OperationalData./aws/automations'`
- Verify runbook document exists: `aws ssm describe-document --name <runbook-name>`
- Check runbook status is Active

SHOULD:
- Verify runbook parameters match OpsItem context
- Check IAM role for runbook execution

MAY:
- List available runbooks: `aws ssm list-documents --filters "Key=DocumentType,Values=Automation"`

## Phase 2 — Remediate

MUST:
- Associate correct runbook with OpsItem
- Ensure runbook document is in Active status

SHOULD:
- Validate runbook parameters before execution
- Test runbook independently

MAY:
- Create custom runbooks for common OpsItem types

## Common Issues

- symptoms: "No runbook associated with OpsItem"
  diagnosis: "OpsItem created without automation association."
  resolution: "Update OpsItem operational data to include runbook reference."

## Output Format

```yaml
root_cause: "automation_runbooks — <specific_cause>"
evidence:
  - type: runbook_config
    content: "<runbook association and status>"
severity: HIGH
mitigation:
  immediate: "Fix runbook association"
  long_term: "Standardize runbook associations per OpsItem type"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: MEDIUM — runbooks may access and modify infrastructure resources
- Service Disruption Risk: HIGH — runbook execution can modify production resources
- Compliance Impact: HIGH — automated changes must follow change management procedures

## Escalation Conditions
- Runbook execution causes unintended resource modifications
- Runbook fails midway leaving resources in inconsistent state
- Runbook permissions allow access to unauthorized resources
- Runbook execution rate exceeds expected thresholds
- Critical runbook fails to execute during incident response

## Data Sensitivity
- Runbook content and logic — INTERNAL
- Execution parameters — INTERNAL (may contain CONFIDENTIAL data)
- Execution logs — INTERNAL
- IAM role configurations for runbooks — CONFIDENTIAL
- Never include hardcoded credentials in runbook parameters

## Prohibited Actions
- Do NOT execute untested runbooks on production resources
- Do NOT grant runbook execution roles overly broad permissions
- Do NOT bypass change management for runbook modifications
- Do NOT execute destructive runbooks without approval
- Do NOT ignore runbook execution failures
- Do NOT store credentials in runbook parameters

## Phase 3 — Rollback
- Cancel in-progress runbook executions if they are causing issues
- Revert resource changes made by failed runbook executions
- Restore previous runbook versions if modifications caused failures
- Revoke runbook execution permissions if they were overly broad
- Verify resource state after runbook execution rollback
- Document runbook execution issues and update runbook logic

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
