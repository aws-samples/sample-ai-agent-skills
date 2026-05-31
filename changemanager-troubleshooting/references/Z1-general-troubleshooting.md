---
title: "Z1 — General Change Manager Troubleshooting"
description: "Catch-all runbook for Change Manager issues"
status: active
severity: MEDIUM
triggers: ["Change Manager error", "change management issue"]
owner: devops-agent
objective: "Systematic investigation of unclassified Change Manager issues"
context: "Use when the issue doesn't clearly match a specific runbook."
---

## Phase 1 — Triage

MUST:
- List templates: `aws ssm list-documents --filters "Key=DocumentType,Values=ChangeTemplate"`
- Check recent change requests: `aws ssm list-ops-item-events --filters "Key=OpsItemType,Values=/aws/changerequest"`
- Review CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=ssm.amazonaws.com --max-results 20`

SHOULD:
- Verify OpsCenter is enabled
- Check SNS topic configuration

MAY:
- Check Change Calendar state

## Phase 2 — Remediate

MUST:
- Identify problem domain and pivot to specific runbook

SHOULD:
- Document findings

MAY:
- Set up monitoring for Change Manager operations

## Output Format

```yaml
root_cause: "general_changemanager — <specific_cause>"
evidence:
  - type: investigation_summary
    content: "<findings>"
severity: MEDIUM
mitigation:
  immediate: "Address identified issue"
  long_term: "Implement monitoring"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Identify problem domain and pivot to specific runbook | GREEN |
| Document findings | GREEN |
| Set up monitoring for Change Manager operations | GREEN |

## Escalation Conditions

- Fix requires bypassing approval workflow
- Change affects production infrastructure

## Data Sensitivity

- **MEDIUM**: change request details, approval workflows
- General troubleshooting may expose templates, requests, and approval configurations

## Prohibited Actions

- NEVER suggest bypassing approval workflows

## Phase 3 — Rollback

1. Rollback steps depend on the specific runbook identified during triage — follow the Phase 3 rollback of the matched runbook
2. If multiple changes were made during investigation, revert them in reverse order
3. Document all changes made during troubleshooting for audit purposes
4. Verify rollback by re-running the Phase 1 triage commands and confirming expected state

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
