---
title: "A1 — Template Creation"
description: "Diagnose change template creation failures"
status: active
severity: HIGH
triggers: ["template creation failed", "cannot create template", "template error"]
owner: devops-agent
objective: "Resolve change template creation issues"
context: "Change templates define the runbook, approval requirements, and notification settings for change requests."
---

## Phase 1 — Triage

MUST:
- Check IAM permissions for ssm:CreateDocument with DocumentType=ChangeTemplate
- Verify the automation runbook referenced in the template exists
- Check template document content for syntax errors

SHOULD:
- Verify SNS topic ARN in template is valid
- Check approver IAM ARNs are valid

MAY:
- Review existing templates for reference: `aws ssm list-documents --filters "Key=DocumentType,Values=ChangeTemplate"`

## Phase 2 — Remediate

MUST:
- Fix IAM permissions for template creation
- Ensure referenced runbook exists and is accessible
- Fix template document syntax

SHOULD:
- Validate all ARNs (SNS, IAM, runbook) before creation

MAY:
- Use CloudFormation for template management

## Common Issues

- symptoms: "CreateDocument fails with InvalidDocumentContent"
  diagnosis: "Template document has syntax errors or invalid references."
  resolution: "Validate document content. Ensure runbook ARN and approver ARNs are correct."

## Output Format

```yaml
root_cause: "template_creation — <specific_cause>"
evidence:
  - type: template_error
    content: "<creation error details>"
severity: HIGH
mitigation:
  immediate: "Fix template content"
  long_term: "Use IaC for template management"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Fix IAM permissions for template creation | YELLOW |
| Ensure referenced runbook exists and is accessible | GREEN |
| Fix template document syntax | GREEN |
| Validate all ARNs (SNS, IAM, runbook) before creation | GREEN |
| Use CloudFormation for template management | GREEN |

## Escalation Conditions

- Fix requires bypassing approval workflow
- Change affects production infrastructure

## Data Sensitivity

- **MEDIUM**: change request details, approval workflows
- Template configurations may reference production runbooks and approver identities

## Prohibited Actions

- NEVER suggest bypassing approval workflows

## Phase 3 — Rollback

1. If a template was created incorrectly, delete it: `aws ssm delete-document --name <template-name>`
2. If IAM permissions were broadened, revert the IAM policy to the previous version
3. If a runbook reference was changed, update the template to point to the correct runbook
4. Verify rollback by confirming `aws ssm list-documents --filters "Key=DocumentType,Values=ChangeTemplate"` shows expected templates

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
