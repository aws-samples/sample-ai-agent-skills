---
title: "D2 — Update Failures"
description: "Diagnose provisioned product update failures"
status: active
severity: HIGH
triggers: ["update failed", "cannot update product", "update rollback", "version upgrade failed"]
owner: devops-agent
objective: "Resolve provisioned product update and version upgrade failures"
context: "Updates change parameters or product version. They trigger CloudFormation stack updates. Failed updates roll back automatically."
---

## Phase 1 — Triage

MUST:
- Check update record: `aws servicecatalog describe-record --id <record-id> --query 'RecordDetail.{Status:Status,RecordErrors:RecordErrors}'`
- Check CloudFormation stack update events: `aws cloudformation describe-stack-events --stack-name <stack-name> --query 'StackEvents[?ResourceStatus==`UPDATE_FAILED`]'`
- Verify the target product version template is valid

SHOULD:
- Compare old and new template for breaking changes
- Check if resources require replacement (causes downtime)
- Verify launch role has permissions for new resources in updated template

MAY:
- Use CloudFormation change set to preview update: `aws cloudformation create-change-set --stack-name <stack-name> --template-url <url> --change-set-name preview`
- Check resource-specific update constraints (e.g., RDS instance class changes)

## Phase 2 — Remediate

MUST:
- Fix the root cause identified in stack events (permission, quota, or resource error)
- Retry update with corrected parameters: `aws servicecatalog update-provisioned-product --provisioned-product-id <id> --provisioning-artifact-id <artifact-id> --provisioning-parameters <params>`
- If stack is in UPDATE_ROLLBACK_FAILED: `aws cloudformation continue-update-rollback --stack-name <stack-name>`

SHOULD:
- Test updates in non-production environment first
- Use update constraints to prevent breaking parameter changes

MAY:
- Create a new product version that handles migration gracefully
- Use CloudFormation stack policies to protect critical resources

## Safety Ratings

```
safety_ratings:
  - "describe-record / describe-stack-events: GREEN — read-only inspection"
  - "update-provisioned-product: YELLOW — modifies resources"
  - "continue-update-rollback: YELLOW — completes rollback, may skip resources"
  - "terminate-provisioned-product: RED — deletes all resources"
```

## Escalation Conditions

- Stack stuck in UPDATE_ROLLBACK_FAILED requiring manual intervention
- Update requires resource replacement causing data loss
- Launch role lacks permissions for new resources in updated template
- Update affects multiple dependent provisioned products

## Data Sensitivity

- `aws servicecatalog describe-record`: MEDIUM — error details and parameters
- `aws cloudformation describe-stack-events`: MEDIUM — resource details
- `aws cloudformation create-change-set`: MEDIUM — preview of resource changes
- Update parameters: MEDIUM — may contain configuration values

## Prohibited Actions

- NEVER force update without understanding what resources will be replaced
- NEVER skip testing updates in non-production environments
- NEVER use continue-update-rollback without understanding which resources will be skipped
- NEVER update provisioned products during maintenance windows without approval

## Phase 3 — Rollback

1. If update failed and auto-rollback succeeded: investigate root cause before retrying
2. If stack is in UPDATE_ROLLBACK_FAILED: `aws cloudformation continue-update-rollback --stack-name <stack-name> --resources-to-skip <resource-ids>`
3. If update succeeded but caused issues: update back to previous version
4. If data was lost during resource replacement: restore from backups
5. Document the update failure and update the product template

## Common Issues

- symptoms: "Update fails with 'Resource handler returned message: ... requires replacement'"
  diagnosis: "The parameter change requires CloudFormation to replace the resource, which may fail due to naming conflicts or dependencies."
  resolution: "Review change set to understand replacements. Update template to handle replacement gracefully or use a different parameter value."

## Output Format

```yaml
root_cause: "update_failure — <specific_cause>"
evidence:
  - type: update_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Resolve update failure"
  long_term: "Implement change set preview before updates and test in non-production"
```

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
