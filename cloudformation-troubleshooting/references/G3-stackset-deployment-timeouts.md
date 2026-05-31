---
title: "G3 — StackSet Deployment Timeouts"
description: "Diagnose and resolve StackSet operations that take too long or appear stuck"
status: active
severity: MEDIUM
triggers:
  - "StackSet timeout"
  - "StackSet slow"
  - "StackSet stuck"
  - "deployment taking too long"
  - "operation RUNNING"
owner: devops-agent
objective: "Identify why a StackSet operation is slow and optimize concurrency and failure tolerance settings"
context: "StackSet operations deploy sequentially by default (MaxConcurrentCount=1). Large deployments across many accounts/regions can take hours. Operations can appear stuck if individual stack instances are waiting for resources (custom resources, ECS services, etc.). The operation timeout is controlled by the individual stack timeouts, not the StackSet operation itself."
---

## Phase 1 — Triage

MUST:
- Check the operation status and progress: `aws cloudformation describe-stack-set-operation --stack-set-name <name> --operation-id <id>`
- Check which instances are still in progress: `aws cloudformation list-stack-set-operation-results --stack-set-name <name> --operation-id <id> --query 'Summaries[?Status==`RUNNING` || Status==`PENDING`]'`
- Check the operation preferences (concurrency settings): look at MaxConcurrentCount or MaxConcurrentPercentage
- Calculate expected duration: (total instances / concurrency) × average stack deployment time

SHOULD:
- Check if any in-progress instances are stuck on specific resources
- Verify the stack template doesn't have long-running custom resources or wait conditions
- Check if API throttling is slowing down the operation

MAY:
- Monitor CloudTrail for throttling events
- Check if other StackSet operations are queued (only one operation per StackSet at a time)

## Phase 2 — Remediate

MUST:
- If the operation is genuinely stuck: cancel it: `aws cloudformation stop-stack-set-operation --stack-set-name <name> --operation-id <id>`
- Increase concurrency for faster deployment: `--operation-preferences MaxConcurrentCount=10` or `MaxConcurrentPercentage=25`
- If individual stacks are slow: optimize the stack template (reduce custom resource timeouts, optimize resource creation)

SHOULD:
- Use region-based concurrency for multi-region deployments
- Set appropriate failure tolerance to avoid stopping on non-critical failures
- Use StackSet operation preferences to balance speed and safety

MAY:
- Split large deployments into smaller batches
- Use service-managed StackSets with automatic deployment for new accounts

## Common Issues

- symptoms: "StackSet operation running for hours with MaxConcurrentCount=1"
  diagnosis: "Sequential deployment across many accounts/regions."
  resolution: "Increase MaxConcurrentCount or MaxConcurrentPercentage."

- symptoms: "Operation stuck on one account while others wait"
  diagnosis: "One stack instance has a long-running resource (custom resource, ECS service)."
  resolution: "Fix the slow resource in that account. Consider increasing failure tolerance to skip it."

- symptoms: "Cannot start new operation — previous operation still running"
  diagnosis: "Only one operation per StackSet at a time."
  resolution: "Wait for the current operation to complete or cancel it."

## Output Format

```yaml
root_cause: "deployment_timeout — concurrency: <n>, instances: <total>, stuck_on: <account/region>"
evidence:
  - type: operation_status
    content: "<operation details and progress>"
  - type: stuck_instances
    content: "<instances still in progress>"
severity: MEDIUM
mitigation:
  immediate: "Increase concurrency or cancel and retry with optimized settings"
  long_term: "Optimize stack templates and set appropriate concurrency/tolerance"
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
