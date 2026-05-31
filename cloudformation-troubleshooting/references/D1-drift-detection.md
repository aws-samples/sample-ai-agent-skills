---
title: "D1 — Drift Detection"
description: "Run and interpret CloudFormation drift detection to find out-of-band changes"
status: active
severity: MEDIUM
triggers:
  - "drift"
  - "out of band"
  - "manual change"
  - "resource modified outside"
  - "stack drift"
owner: devops-agent
objective: "Detect and report resources that have drifted from their CloudFormation-managed configuration"
context: "Drift detection compares the actual resource configuration with the expected configuration defined in the stack template. Not all resource types support drift detection — unsupported resources are silently skipped. Drift detection is asynchronous and may take several minutes for large stacks."
---

## Phase 1 — Triage

MUST:
- Initiate drift detection: `aws cloudformation detect-stack-drift --stack-name <stack>`
- Check detection status (wait for DETECTION_COMPLETE): `aws cloudformation describe-stack-drift-detection-status --stack-drift-detection-id <id>`
- Get drift results: `aws cloudformation describe-stack-resource-drifts --stack-name <stack> --stack-resource-drift-status-filters MODIFIED DELETED`
- Review each drifted resource's property differences

SHOULD:
- Check which resources were actually checked vs skipped: `aws cloudformation describe-stack-resource-drifts --stack-name <stack> --stack-resource-drift-status-filters IN_SYNC NOT_CHECKED`
- Identify who made the out-of-band change via CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=ResourceName,AttributeValue=<physical-resource-id>`
- Document all drifted properties for remediation planning

MAY:
- Detect drift on a specific resource: `aws cloudformation detect-stack-resource-drift --stack-name <stack> --logical-resource-id <id>`
- Compare drift across multiple stacks in the same environment

## Phase 2 — Remediate

See D2 (Drift Remediation) for remediation steps.

## Common Issues

- symptoms: "Drift detection shows DETECTION_FAILED"
  diagnosis: "IAM permissions insufficient to read resource configurations."
  resolution: "Ensure the CloudFormation role has read permissions for all resource types in the stack."

- symptoms: "Resource shows NOT_CHECKED"
  diagnosis: "Resource type does not support drift detection."
  resolution: "Manually verify the resource configuration. Not all types are supported."

- symptoms: "Drift detection takes very long"
  diagnosis: "Large stacks with many resources take longer to check."
  resolution: "Wait for completion. Use resource-level drift detection for specific resources."

## Output Format

```yaml
root_cause: "drift_detected — <count> resources drifted"
evidence:
  - type: drift_results
    content: "<list of drifted resources and changed properties>"
  - type: cloudtrail
    content: "<who made the out-of-band changes>"
severity: MEDIUM
mitigation:
  immediate: "Document drift and plan remediation (see D2)"
  long_term: "Implement drift detection on a schedule and enforce IaC-only changes"
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
