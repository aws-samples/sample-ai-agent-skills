---
title: "B2 — Circular Dependencies"
description: "Diagnose and resolve circular dependency errors in CloudFormation templates"
status: active
severity: MEDIUM
triggers:
  - "Circular dependency"
  - "circular reference"
  - "depends on cycle"
  - "Circular dependency between resources"
owner: devops-agent
objective: "Identify the circular dependency chain and restructure the template to break the cycle"
context: "Circular dependencies are detected at template validation time, before any resources are provisioned. They occur when Resource A depends on Resource B which depends on Resource A (directly or transitively). Dependencies are created by Ref, Fn::GetAtt, DependsOn, and Fn::Sub with resource references."
---

## Phase 1 — Triage

MUST:
- Validate the template to see the circular dependency error: `aws cloudformation validate-template --template-body file://template.yaml`
- Identify the resources named in the circular dependency error message
- Map all dependencies between the named resources (Ref, GetAtt, DependsOn, Fn::Sub)
- Determine which dependency can be broken without losing functionality

SHOULD:
- Use `cfn-lint` for more detailed dependency analysis: `cfn-lint template.yaml`
- Draw the dependency graph to visualize the cycle
- Check if implicit dependencies (Ref, GetAtt) can be replaced with parameters or exports

MAY:
- Check if the cycle involves security groups referencing each other (very common pattern)
- Check if the cycle involves a Lambda function and its execution role

## Phase 2 — Remediate

MUST:
- Break the cycle using one of these strategies:
  - Split a resource into two (e.g., separate security group and its ingress rules)
  - Use `Fn::ImportValue` with cross-stack references instead of direct Ref
  - Remove unnecessary DependsOn declarations
  - Use `AWS::EC2::SecurityGroupIngress` / `AWS::EC2::SecurityGroupEgress` as separate resources instead of inline rules
- Re-validate the template after changes

SHOULD:
- Verify the fix doesn't change the deployment behavior
- Test the fixed template in a non-production environment

MAY:
- Refactor into nested stacks to separate tightly coupled resources
- Use SSM Parameter Store to share values between resources without direct dependencies

## Common Issues

- symptoms: "Circular dependency between SecurityGroupA and SecurityGroupB"
  diagnosis: "Two security groups reference each other in their ingress/egress rules."
  resolution: "Use AWS::EC2::SecurityGroupIngress as a separate resource instead of inline SecurityGroupIngress."

- symptoms: "Circular dependency between LambdaFunction and LambdaRole"
  diagnosis: "Lambda function references the role, and the role policy references the function ARN."
  resolution: "Use Fn::Sub with the function name pattern instead of Ref/GetAtt for the function ARN in the role policy."

- symptoms: "Circular dependency involving DependsOn"
  diagnosis: "Explicit DependsOn creates a cycle with implicit Ref/GetAtt dependencies."
  resolution: "Remove the DependsOn — implicit dependencies from Ref/GetAtt are usually sufficient."

## Output Format

```yaml
root_cause: "circular_dependency — <resource_a> ↔ <resource_b> [↔ <resource_c>]"
evidence:
  - type: validation_output
    content: "<circular dependency error message>"
  - type: dependency_chain
    content: "<A → B via Ref, B → A via GetAtt>"
severity: MEDIUM
mitigation:
  immediate: "Break the cycle by restructuring the dependency"
  long_term: "Use cfn-lint in CI/CD to catch circular dependencies early"
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
