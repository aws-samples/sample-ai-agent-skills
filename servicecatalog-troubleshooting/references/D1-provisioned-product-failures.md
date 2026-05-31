---
title: "D1 — Provisioned Product Failures"
description: "Diagnose provisioned product status failures and errors"
status: active
severity: HIGH
triggers: ["provisioned product failed", "TAINTED", "ERROR status", "stack failed", "product in error state"]
owner: devops-agent
objective: "Resolve provisioned product failures and error states"
context: "Provisioned products map to CloudFormation stacks. TAINTED status means the stack drifted or a previous operation failed. ERROR means the stack creation/update failed."
---

## Phase 1 — Triage

MUST:
- Check provisioned product status: `aws servicecatalog describe-provisioned-product --id <provisioned-product-id>`
- Get record details: `aws servicecatalog list-record-history --search-filter Key=PROVISIONED_PRODUCT,Value=<provisioned-product-id>`
- Check CloudFormation stack events: `aws cloudformation describe-stack-events --stack-name <stack-name> --query 'StackEvents[?ResourceStatus==`CREATE_FAILED` || ResourceStatus==`UPDATE_FAILED`]'`

SHOULD:
- Check stack drift: `aws cloudformation detect-stack-drift --stack-name <stack-name>`
- Verify resources created by the stack are healthy

MAY:
- Review CloudWatch logs for application-level errors
- Check resource-specific service quotas

## Phase 2 — Remediate

MUST:
- For TAINTED status: update provisioned product to reconcile state: `aws servicecatalog update-provisioned-product --provisioned-product-id <id> --product-id <product-id> --provisioning-artifact-id <artifact-id>`
- For ERROR status: fix the root cause in the template or parameters, then retry
- Check and fix IAM permissions on the launch role if permission errors

SHOULD:
- Use describe-record to get specific error messages before remediation
- Validate template changes in a non-production environment first

MAY:
- Terminate and re-provision if the product is in an unrecoverable state
- Use CloudFormation change sets to preview updates before applying

## Safety Ratings

```
safety_ratings:
  - "describe-provisioned-product / list-record-history: GREEN — read-only status check"
  - "describe-stack-events: GREEN — read-only stack inspection"
  - "update-provisioned-product: YELLOW — modifies resources via CloudFormation"
  - "terminate-provisioned-product: RED — deletes all resources in the stack"
```

## Escalation Conditions

- Provisioned product is in ERROR state and cannot be updated or terminated
- CloudFormation stack is in DELETE_FAILED state requiring manual resource cleanup
- Stack drift detected on critical resources
- Multiple provisioned products failing simultaneously

## Data Sensitivity

- `aws servicecatalog describe-provisioned-product`: LOW — product metadata
- `aws cloudformation describe-stack-events`: MEDIUM — may contain parameter values and resource IDs
- `aws servicecatalog list-record-history`: MEDIUM — operation history
- Stack outputs: MEDIUM — may contain endpoints, IDs, or connection strings

## Prohibited Actions

- NEVER modify the CloudFormation stack directly (always use Service Catalog APIs)
- NEVER terminate provisioned products with stateful resources without backup
- NEVER ignore TAINTED status — it indicates drift or partial failure
- NEVER force-delete CloudFormation stacks without understanding resource dependencies

## Phase 3 — Rollback

1. If update caused the failure: update back to previous product version
2. If resources were partially created: terminate provisioned product after data backup
3. If stack is stuck in UPDATE_ROLLBACK_FAILED: `aws cloudformation continue-update-rollback --stack-name <stack-name>`
4. If manual resource cleanup is needed: identify and delete orphaned resources
5. Document the failure root cause and update the product template

## Common Issues

- symptoms: "Provisioned product shows TAINTED status after a failed update"
  diagnosis: "A previous update operation failed and the stack rolled back, leaving Service Catalog in an inconsistent state."
  resolution: "Update the provisioned product again with correct parameters. If that fails, terminate and re-provision after backing up data."

## Output Format

```yaml
root_cause: "provisioned_product_failure — <specific_cause>"
evidence:
  - type: product_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Resolve product error state"
  long_term: "Implement monitoring and alerting for provisioned product status"
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
