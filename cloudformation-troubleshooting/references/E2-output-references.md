---
title: "E2 — Nested Stack Output References"
description: "Diagnose issues with cross-stack output references and exports"
status: active
severity: MEDIUM
triggers:
  - "output not found"
  - "export not found"
  - "Fn::ImportValue"
  - "cross-stack reference"
  - "No export named"
owner: devops-agent
objective: "Fix broken cross-stack references by ensuring exports exist and match imports"
context: "Cross-stack references use Outputs with Export and Fn::ImportValue. Common issues: the exporting stack doesn't exist or hasn't been created yet, the export name doesn't match, the export was removed while still imported, or the exporting stack is in a different region. Nested stack outputs use Fn::GetAtt with the nested stack logical ID."
---

## Phase 1 — Triage

MUST:
- Identify the missing export from the error message
- List all exports in the region: `aws cloudformation list-exports`
- Check if the exporting stack exists and is in a healthy state
- For nested stacks: verify the nested stack has the expected output: `aws cloudformation describe-stacks --stack-name <nested-stack> --query 'Stacks[0].Outputs'`

SHOULD:
- Check which stacks import a specific export: `aws cloudformation list-imports --export-name <export-name>`
- Verify the export name matches exactly (case-sensitive)
- Check if the exporting stack was recently updated and the export was removed

MAY:
- Check if the export exists in a different region
- Review the deployment order to ensure the exporting stack is created first

## Phase 2 — Remediate

MUST:
- For missing exports: create or fix the exporting stack first
- For name mismatches: correct the export name or import reference
- For nested stack outputs: ensure the nested stack template has the Output defined
- Ensure deployment order: exporting stack must be deployed before importing stack

SHOULD:
- Use consistent naming conventions for exports
- Document cross-stack dependencies

MAY:
- Use SSM Parameter Store as an alternative to cross-stack references for more flexibility
- Use CloudFormation StackSets for multi-region deployments with consistent exports

## Common Issues

- symptoms: "No export named 'VpcId' found"
  diagnosis: "The exporting stack hasn't been created or the export name is wrong."
  resolution: "Deploy the exporting stack first. Verify the export name matches exactly."

- symptoms: "Export 'VpcId' cannot be deleted as it is in use by stack 'AppStack'"
  diagnosis: "Cannot remove an export while another stack imports it."
  resolution: "Update the importing stack to remove the Fn::ImportValue first, then remove the export."

- symptoms: "Fn::GetAtt on nested stack returns empty"
  diagnosis: "The nested stack doesn't define the expected Output."
  resolution: "Add the Output to the nested stack template."

## Output Format

```yaml
root_cause: "output_reference — <export_name>: <missing|mismatched|removed>"
evidence:
  - type: error_message
    content: "<export/import error>"
  - type: exports_list
    content: "<available exports>"
severity: MEDIUM
mitigation:
  immediate: "Fix the export/import reference"
  long_term: "Document cross-stack dependencies and enforce deployment order"
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
