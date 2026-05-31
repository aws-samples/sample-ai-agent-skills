---
title: "A2 — Template Approval"
description: "Diagnose template approval issues"
status: active
severity: MEDIUM
triggers: ["template pending", "template not approved", "template rejected"]
owner: devops-agent
objective: "Resolve template approval issues"
context: "Templates must be approved before use. Template approval is separate from change request approval."
---

## Phase 1 — Triage

MUST:
- Check template status: `aws ssm describe-document --name <template> --query 'Document.{Status:DocumentStatus,ReviewStatus:ReviewStatus}'`
- Verify template approvers have been notified
- Check if template was rejected (review comments)

SHOULD:
- Verify approvers have IAM permissions to approve templates
- Check SNS delivery for approval notifications

MAY:
- Review template approval history

## Phase 2 — Remediate

MUST:
- If rejected: address review comments and resubmit
- If pending: ensure approvers are notified and have access

SHOULD:
- Follow up with approvers directly if notifications failed

MAY:
- Streamline approval process for standard templates

## Common Issues

- symptoms: "Template stuck in PENDING for days"
  diagnosis: "Approvers not notified or don't have approval permissions."
  resolution: "Verify SNS notifications. Ensure approvers have ssm:UpdateDocumentMetadata permission."

## Output Format

```yaml
root_cause: "template_approval — <specific_cause>"
evidence:
  - type: template_status
    content: "<template review status>"
severity: MEDIUM
mitigation:
  immediate: "Resolve approval blocker"
  long_term: "Define SLA for template approvals"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| If rejected: address review comments and resubmit | GREEN |
| If pending: ensure approvers are notified and have access | GREEN |
| Follow up with approvers directly if notifications failed | GREEN |
| Streamline approval process for standard templates | YELLOW |

## Escalation Conditions

- Fix requires bypassing approval workflow
- Change affects production infrastructure

## Data Sensitivity

- **MEDIUM**: change request details, approval workflows
- Approval status and reviewer comments may contain sensitive operational details

## Prohibited Actions

- NEVER suggest bypassing approval workflows

## Phase 3 — Rollback

1. If a template was approved in error, update its status or delete and recreate with correct configuration
2. If approver permissions were modified, revert IAM changes
3. If SNS notifications were reconfigured, restore previous notification settings
4. Verify rollback by checking template review status: `aws ssm describe-document --name <template>`

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
