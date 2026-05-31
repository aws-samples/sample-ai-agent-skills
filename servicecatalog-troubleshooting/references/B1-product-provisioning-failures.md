---
title: "B1 — Product Provisioning Failures"
description: "Diagnose Service Catalog product provisioning failures"
status: active
severity: HIGH
triggers: ["provisioning failed", "cannot provision product", "product launch failed", "TAINTED status"]
owner: devops-agent
objective: "Resolve product provisioning failures"
context: "Provisioning creates a CloudFormation stack. Failures are usually stack creation errors. Launch constraints define the provisioning role."
---

## Phase 1 — Triage

MUST:
- Check provisioned product status: `aws servicecatalog describe-provisioned-product --id <provisioned-product-id>`
- Get record details with errors: `aws servicecatalog describe-record --id <record-id> --query 'RecordDetail.{Status:Status,RecordErrors:RecordErrors}'`
- Check underlying CloudFormation stack: `aws cloudformation describe-stack-events --stack-name <stack-name> --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`]'`

SHOULD:
- Verify launch constraint exists: `aws servicecatalog list-constraints-for-portfolio --portfolio-id <portfolio-id> --query 'ConstraintDetails[?Type==`LAUNCH`]'`
- Check the launch role permissions: `aws iam simulate-principal-policy --policy-source-arn <launch-role-arn> --action-names <required-actions>`

MAY:
- Review the product template for issues: `aws servicecatalog describe-provisioning-artifact --product-id <product-id> --provisioning-artifact-id <artifact-id>`
- Check CloudTrail for ProvisionProduct events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=ProvisionProduct --max-results 10`

## Phase 2 — Remediate

MUST:
- Fix launch constraint role permissions to include all resources in the template
- Ensure CloudFormation stack parameters match template constraints
- Verify resource limits are not exceeded in the target account/region

SHOULD:
- Create or update launch constraint with proper IAM role
- Validate CloudFormation template before creating product version

MAY:
- Use CloudFormation drift detection to identify configuration issues
- Set up SNS notifications for provisioning failures

## Safety Ratings

```
safety_ratings:
  - "describe-provisioned-product / describe-record: GREEN — read-only status check"
  - "describe-stack-events: GREEN — read-only stack inspection"
  - "provision-product: YELLOW — creates resources via CloudFormation"
  - "terminate-provisioned-product: RED — deletes CloudFormation stack and resources"
```

## Escalation Conditions

- Launch role lacks permissions that require security team to grant
- CloudFormation stack fails due to service quota limits
- Template references resources that don't exist in the target account
- Provisioning fails with internal service errors after fixing all known issues

## Data Sensitivity

- `aws servicecatalog describe-provisioned-product`: LOW — product metadata
- `aws cloudformation describe-stack-events`: MEDIUM — may contain parameter values
- `aws iam simulate-principal-policy`: LOW — permission simulation
- `aws servicecatalog describe-record`: MEDIUM — may contain error details with resource info

## Prohibited Actions

- NEVER terminate provisioned products to fix provisioning issues without data backup
- NEVER modify the CloudFormation stack directly (use Service Catalog update)
- NEVER grant AdministratorAccess to the launch role
- NEVER provision products in production without testing in a non-production account first

## Phase 3 — Rollback

1. If provisioning created partial resources: `aws servicecatalog terminate-provisioned-product --provisioned-product-id <id>` (after data backup)
2. If launch constraint was modified: restore original role ARN in constraint
3. If template was updated: revert to previous product version
4. Verify no orphaned resources remain after failed provisioning
5. Document the failure and update the product template or launch role

## Common Issues

- symptoms: "Provisioning fails with 'User: arn:aws:iam::... is not authorized to perform: ec2:RunInstances'"
  diagnosis: "No launch constraint set, so end user's permissions are used for CloudFormation stack creation."
  resolution: "Create a launch constraint with an IAM role that has permissions for all resources in the template."

## Output Format

```yaml
root_cause: "provisioning_failure — <specific_cause>"
evidence:
  - type: provisioning_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix provisioning blocker"
  long_term: "Validate templates and launch roles before publishing products"
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
