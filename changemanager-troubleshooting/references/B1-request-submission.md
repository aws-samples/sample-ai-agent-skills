---
title: "B1 — Request Submission"
description: "Diagnose change request submission failures"
status: active
severity: HIGH
triggers: ["cannot submit request", "request failed", "submission error"]
owner: devops-agent
objective: "Resolve change request submission issues"
context: "Change requests are submitted using approved templates. Submission requires valid template, parameters, and IAM permissions."
---

## Phase 1 — Triage

MUST:
- Verify template is approved: `aws ssm describe-document --name <template> --query 'Document.ReviewStatus'`
- Check IAM permissions for ssm:StartChangeRequestExecution
- Verify request parameters match template requirements

SHOULD:
- Check Change Calendar state: `aws ssm get-calendar-state --calendar-names <name>`
- Verify target resources exist

MAY:
- Review CloudTrail for submission errors

## Phase 2 — Remediate

MUST:
- Use an approved template
- Provide all required parameters
- Ensure IAM permissions are sufficient

SHOULD:
- Validate parameters before submission

MAY:
- Use console for better error messages during submission

## Common Issues

- symptoms: "StartChangeRequestExecution returns InvalidDocument"
  diagnosis: "Template not approved or doesn't exist."
  resolution: "Verify template is in APPROVED status."

## Output Format

```yaml
root_cause: "request_submission — <specific_cause>"
evidence:
  - type: submission_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix submission issue"
  long_term: "Validate prerequisites before submission"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Use an approved template | GREEN |
| Provide all required parameters | GREEN |
| Ensure IAM permissions are sufficient | YELLOW |
| Validate parameters before submission | GREEN |
| Use console for better error messages during submission | GREEN |

## Escalation Conditions

- Fix requires bypassing approval workflow
- Change affects production infrastructure

## Data Sensitivity

- **MEDIUM**: change request details, approval workflows
- Request parameters may contain target resource identifiers and configuration values

## Prohibited Actions

- NEVER suggest bypassing approval workflows

## Phase 3 — Rollback

1. If a change request was submitted in error, reject it through the approval workflow
2. If IAM permissions were broadened for submission, revert the IAM policy
3. If incorrect parameters were used, cancel the request and resubmit with correct values
4. Verify rollback by checking request status: `aws ssm get-ops-item --ops-item-id <id>`

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
