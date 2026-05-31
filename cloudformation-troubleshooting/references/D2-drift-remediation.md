---
title: "D2 — Drift Remediation"
description: "Remediate drifted CloudFormation resources by updating the template or reverting changes"
status: active
severity: MEDIUM
triggers:
  - "fix drift"
  - "remediate drift"
  - "sync template"
  - "update template to match"
  - "revert drift"
owner: devops-agent
objective: "Bring drifted resources back in sync with the CloudFormation template"
context: "Drift remediation has two approaches: (1) update the template to match the actual state (accept the drift), or (2) update the resource to match the template (revert the drift). The choice depends on whether the out-of-band change was intentional. CloudFormation does not have an automatic 'fix drift' feature."
---

## Phase 1 — Triage

MUST:
- Review drift detection results: `aws cloudformation describe-stack-resource-drifts --stack-name <stack> --stack-resource-drift-status-filters MODIFIED DELETED`
- For each drifted resource, determine if the change was intentional or accidental
- Identify the specific properties that drifted and their expected vs actual values
- Determine the remediation strategy: accept drift (update template) or revert drift (update resource)

SHOULD:
- Check CloudTrail to understand why the change was made
- Assess the impact of reverting vs accepting each drifted property
- Prioritize security-sensitive drift (security groups, IAM policies, encryption settings)

MAY:
- Create a drift report for change management review
- Check if the drift affects other stacks or resources

## Phase 2 — Remediate

MUST:
- For accepting drift (update template to match actual):
  - Update the template properties to match the actual resource configuration
  - Create a change set to verify no unintended changes: `aws cloudformation create-change-set --stack-name <stack> --template-body file://updated-template.yaml --change-set-name accept-drift`
  - Verify the change set shows no changes for the drifted resources (template now matches actual)
  - Execute the change set
- For reverting drift (update resource to match template):
  - Perform a stack update with the existing template: `aws cloudformation update-stack --stack-name <stack> --use-previous-template`
  - Or create a change set with the original template to preview the revert

SHOULD:
- Test remediation in a non-production environment first
- Verify drift is resolved after remediation: re-run drift detection

MAY:
- Implement AWS Config rules to detect and alert on drift automatically
- Use CloudFormation StackSets to enforce consistent configuration across accounts

## Common Issues

- symptoms: "Stack update shows no changes but drift still exists"
  diagnosis: "The drifted property is not managed by CloudFormation (read-only or computed property)."
  resolution: "Some properties cannot be reverted via CloudFormation. Use the AWS CLI or console to manually fix."

- symptoms: "Reverting drift causes a resource replacement"
  diagnosis: "The drifted property is immutable — changing it requires resource replacement."
  resolution: "Assess the impact of replacement. Consider accepting the drift instead."

- symptoms: "Multiple resources drifted with conflicting changes"
  diagnosis: "Out-of-band changes created dependencies between drifted resources."
  resolution: "Remediate in dependency order. Fix referenced resources before dependents."

## Output Format

```yaml
root_cause: "drift_remediation — strategy: <accept|revert>, resources: [<list>]"
evidence:
  - type: drift_details
    content: "<drifted properties and values>"
  - type: change_set
    content: "<change set preview of remediation>"
severity: MEDIUM
mitigation:
  immediate: "Apply the chosen remediation strategy"
  long_term: "Implement drift detection scheduling and IaC enforcement"
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
