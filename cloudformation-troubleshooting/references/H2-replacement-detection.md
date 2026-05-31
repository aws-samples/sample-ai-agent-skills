---
title: "H2 — Replacement Detection"
description: "Identify and manage resource replacements in CloudFormation change sets"
status: active
severity: HIGH
triggers:
  - "replacement"
  - "resource will be replaced"
  - "Replacement: True"
  - "Replacement: Conditional"
  - "data loss"
  - "recreate resource"
owner: devops-agent
objective: "Detect resource replacements before they happen and plan for data preservation"
context: "Some property changes require CloudFormation to replace the resource (delete old, create new). This can cause data loss for stateful resources (RDS, DynamoDB, EBS). Change sets show Replacement: True (always replaces), Replacement: Conditional (may replace depending on values), or Replacement: False (in-place update). Always check change sets before executing updates to stateful resources."
---

## Phase 1 — Triage

MUST:
- Create a change set to preview: `aws cloudformation create-change-set --stack-name <stack> --template-body file://template.yaml --change-set-name preview`
- Check for replacements: `aws cloudformation describe-change-set --stack-name <stack> --change-set-name preview --query 'Changes[?ResourceChange.Replacement!=`False`]'`
- Identify which resources will be replaced and which properties trigger the replacement
- Assess data loss risk for stateful resources (databases, storage, etc.)

SHOULD:
- Check the UpdateReplacePolicy on resources that will be replaced
- For Conditional replacements: determine if the specific value change triggers replacement
- Create a backup of stateful resources before proceeding

MAY:
- Check the CloudFormation resource type documentation for which properties require replacement
- Test the update in a non-production environment first

## Phase 2 — Remediate

MUST:
- For stateful resources with data loss risk:
  - Set `UpdateReplacePolicy: Snapshot` to create a snapshot before replacement (RDS, EBS)
  - Set `UpdateReplacePolicy: Retain` to keep the old resource
  - Create manual backups before executing the change set
- Review and accept the replacement risk before executing

SHOULD:
- Add UpdateReplacePolicy to all stateful resources in the template
- Document which property changes trigger replacement for critical resources
- Use blue-green deployment patterns for zero-downtime replacements

MAY:
- Use resource import (D3) to adopt the old resource into a different stack
- Implement custom resource-based backup before replacement

## Common Issues

- symptoms: "RDS instance replaced after changing Engine version"
  diagnosis: "Some RDS property changes require replacement."
  resolution: "Use UpdateReplacePolicy: Snapshot. Or use RDS native upgrade instead of CloudFormation."

- symptoms: "Replacement: Conditional on EC2 instance"
  diagnosis: "The property change may or may not require replacement depending on the value."
  resolution: "Test in non-production. Check AWS documentation for the specific property."

- symptoms: "Data lost after stack update replaced DynamoDB table"
  diagnosis: "DynamoDB table was replaced and no backup was taken."
  resolution: "Add UpdateReplacePolicy: Retain or Snapshot. Enable point-in-time recovery."

## Output Format

```yaml
root_cause: "replacement_detected — <resource_logical_id>: <triggering_property>"
evidence:
  - type: change_set
    content: "<change set replacement details>"
  - type: resource_properties
    content: "<properties causing replacement>"
severity: HIGH
mitigation:
  immediate: "Set UpdateReplacePolicy and create backups before executing"
  long_term: "Add UpdateReplacePolicy to all stateful resources"
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
