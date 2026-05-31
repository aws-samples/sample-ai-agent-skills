---
title: "C4 — Resource Dependency Failure"
description: "Diagnose cascading resource failures caused by dependency chain issues"
status: active
severity: HIGH
triggers:
  - "Resource creation cancelled"
  - "dependency failed"
  - "dependent resource"
  - "cancelled"
  - "Resource is not in the state"
owner: devops-agent
objective: "Identify the root cause resource in the dependency chain and fix it to unblock dependent resources"
context: "When a resource fails, all resources that depend on it (via Ref, GetAtt, or DependsOn) are cancelled. The stack events show multiple CREATE_FAILED events, but only the FIRST one is the root cause. Subsequent 'Resource creation cancelled' events are cascading failures."
---

## Phase 1 — Triage

MUST:
- Find ALL CREATE_FAILED events: `aws cloudformation describe-stack-events --stack-name <stack> --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`]'`
- Identify the FIRST CREATE_FAILED event (bottom of the list) — this is the root cause
- Distinguish between actual failures and cascading cancellations (look for "Resource creation cancelled" in the reason)
- Read the ResourceStatusReason of the root cause resource

SHOULD:
- Map the dependency chain from the root cause to understand the cascade
- Check if the dependency is explicit (DependsOn) or implicit (Ref, GetAtt)
- Verify the root cause resource's prerequisites exist

MAY:
- Visualize the dependency graph using `cfn-lint --graph`
- Check if removing a DependsOn would allow parallel creation

## Phase 2 — Remediate

MUST:
- Fix the root cause resource (the FIRST failure) — see the appropriate runbook for that resource type
- Do NOT try to fix the cascading failures — they will resolve when the root cause is fixed
- Retry the stack operation after fixing the root cause

SHOULD:
- Review the dependency chain for unnecessary DependsOn declarations
- Consider if the dependency can be made more resilient (e.g., using Conditions)

MAY:
- Restructure the template to reduce dependency chain depth
- Use parallel resource creation where possible by removing unnecessary ordering

## Common Issues

- symptoms: "Multiple CREATE_FAILED events, most saying 'Resource creation cancelled'"
  diagnosis: "One resource failed and all dependents were cancelled."
  resolution: "Find the FIRST CREATE_FAILED event (not cancelled). Fix that resource."

- symptoms: "Resource A failed because Resource B is 'not in the state stackCreateComplete'"
  diagnosis: "Resource A depends on Resource B, which hasn't finished creating."
  resolution: "Check why Resource B is delayed or failed. Fix Resource B first."

- symptoms: "Cascading failures across nested stacks"
  diagnosis: "A resource in an inner nested stack failed, causing the parent to fail."
  resolution: "Navigate to the innermost failing stack and fix the root cause resource."

## Output Format

```yaml
root_cause: "dependency_failure — root_cause_resource: <logical_id>, cascading_failures: [<list>]"
evidence:
  - type: stack_event
    content: "<first CREATE_FAILED event (root cause)>"
  - type: dependency_chain
    content: "<root_cause → dependent_1 → dependent_2>"
severity: HIGH
mitigation:
  immediate: "Fix the root cause resource — cascading failures will resolve"
  long_term: "Minimize dependency chain depth and remove unnecessary DependsOn"
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
