---
title: "D3 — Resource Import"
description: "Import existing AWS resources into a CloudFormation stack"
status: active
severity: MEDIUM
triggers:
  - "import resource"
  - "resource import"
  - "bring under management"
  - "adopt resource"
  - "import into stack"
owner: devops-agent
objective: "Successfully import existing AWS resources into CloudFormation management"
context: "Resource import allows you to bring existing AWS resources under CloudFormation management without recreating them. Requirements: the resource must have DeletionPolicy: Retain in the template, the resource must not be managed by another stack, and the resource type must support import. Import uses a special IMPORT change set type."
---

## Phase 1 — Triage

MUST:
- Verify the resource exists and is not managed by another stack: `aws cloudformation describe-stack-resources --physical-resource-id <physical-id>`
- Verify the resource type supports import (check AWS documentation)
- Identify the resource's physical identifier (e.g., bucket name, table name, instance ID)
- Prepare the template with the resource definition including `DeletionPolicy: Retain`

SHOULD:
- Match the template resource properties to the actual resource configuration to avoid drift
- Check if the resource has tags that indicate previous CloudFormation management
- Verify IAM permissions for the import operation

MAY:
- Import multiple resources in a single change set
- Plan the import as part of a larger migration strategy

## Phase 2 — Remediate

MUST:
- Add the resource to the template with `DeletionPolicy: Retain`
- Create an IMPORT change set: `aws cloudformation create-change-set --stack-name <stack> --change-set-name import-resource --change-set-type IMPORT --resources-to-import '[{"ResourceType":"<type>","LogicalResourceId":"<logical-id>","ResourceIdentifier":{"<key>":"<value>"}}]' --template-body file://template.yaml`
- Review the change set: `aws cloudformation describe-change-set --stack-name <stack> --change-set-name import-resource`
- Execute the change set: `aws cloudformation execute-change-set --stack-name <stack> --change-set-name import-resource`
- Verify the import succeeded: `aws cloudformation describe-stack-resource --stack-name <stack> --logical-resource-id <logical-id>`

SHOULD:
- Run drift detection after import to verify template matches actual configuration
- Update the template properties to match the actual resource if drift is detected
- Remove DeletionPolicy: Retain after import if you want CloudFormation to delete the resource on stack deletion

MAY:
- Import into a new stack by creating the stack with the IMPORT change set type
- Use `aws cloudformation get-template-summary` to check supported import identifiers

## Common Issues

- symptoms: "Import fails with 'Resource is already managed by stack <other-stack>'"
  diagnosis: "The resource is managed by another CloudFormation stack."
  resolution: "Remove the resource from the other stack first (with DeletionPolicy: Retain)."

- symptoms: "Import fails with 'DeletionPolicy must be Retain'"
  diagnosis: "The template does not have DeletionPolicy: Retain on the resource."
  resolution: "Add DeletionPolicy: Retain to the resource in the template."

- symptoms: "Import succeeds but drift is detected immediately"
  diagnosis: "Template properties don't match the actual resource configuration."
  resolution: "Update the template to match the actual resource properties."

## Output Format

```yaml
root_cause: "resource_import — <resource_type>: <physical_id>"
evidence:
  - type: import_change_set
    content: "<change set details>"
  - type: resource_status
    content: "<post-import resource status>"
severity: MEDIUM
mitigation:
  immediate: "Execute the import change set"
  long_term: "Run drift detection and align template with actual configuration"
```


## Safety Ratings
- GREEN: describe-stacks, describe-stack-events, get-template, validate-template
- YELLOW: continue-update-rollback, create-change-set, update-stack
- RED: delete-stack, set-stack-policy with open policy, skip resources during rollback

## Escalation Conditions
- "Stack is in UPDATE_ROLLBACK_FAILED state"
- "Remediation requires skipping resources during rollback"
- "Fix involves modifying stack policy"
- "Nested stack failure in production"

## Rollback
- Pre-change: "Create change set before any update"
- Verification: "Check stack events after operation"
- Revert: "Use rollback triggers or manual rollback"

## Data Sensitivity
- MEDIUM: "Stack templates may contain hardcoded credentials or account IDs"
- MEDIUM: "Stack outputs may expose endpoint URLs and resource IDs"

## Prohibited Actions
- NEVER leave a stack policy open after temporary modification
- NEVER suggest deleting a production stack to fix UPDATE_ROLLBACK_FAILED without exploring ContinueUpdateRollback
- NEVER suggest `--resources-to-skip` without documenting which resources and why
- NEVER suggest modifying CT-managed stacks (AWSControlTower* prefix)

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
  - "NEVER leave stack policy open after temporary modification"
  - "NEVER delete a stack without exploring ContinueUpdateRollback first"
  - "NEVER suggest disabling rollback on production stacks"

## Phase 3 � Rollback

- Document the current state before making any changes
- If remediation worsens the issue, revert to the documented pre-change state
- Remove any temporary resources created during investigation
- Verify the system returns to its previous functional state after rollback
