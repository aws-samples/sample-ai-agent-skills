---
title: "B3 — Parameter Issues"
description: "Diagnose and resolve CloudFormation parameter validation and constraint failures"
status: active
severity: MEDIUM
triggers:
  - "Parameter validation failed"
  - "parameter constraint"
  - "AllowedValues"
  - "AllowedPattern"
  - "parameter not found"
  - "missing parameter"
owner: devops-agent
objective: "Identify parameter validation failures and fix parameter values or constraints"
context: "Parameter issues are caught at stack creation/update time. Common causes: missing required parameters, values not matching AllowedValues or AllowedPattern, type mismatches, SSM parameter resolution failures, and exceeding the 200 parameter limit."
---

## Phase 1 — Triage

MUST:
- Check the error message for the specific parameter and constraint that failed
- Review the template parameters section: `aws cloudformation get-template --stack-name <stack> --query 'TemplateBody.Parameters'`
- Check current parameter values: `aws cloudformation describe-stacks --stack-name <stack> --query 'Stacks[0].Parameters'`
- Validate the template to check parameter definitions: `aws cloudformation validate-template --template-body file://template.yaml`

SHOULD:
- Check if SSM parameter types resolve correctly: `aws ssm get-parameter --name <parameter-name>`
- Verify parameter count is under 200
- Check if default values are valid against constraints

MAY:
- Review parameter overrides in CI/CD pipeline configuration
- Check if the parameter file (JSON) has correct syntax

## Phase 2 — Remediate

MUST:
- Fix the parameter value to match the constraint (AllowedValues, AllowedPattern, Type)
- For missing parameters: provide the value or add a Default in the template
- For SSM resolution failures: ensure the SSM parameter exists and the role has ssm:GetParameter permission
- Retry the stack operation with corrected parameters

SHOULD:
- Use parameter files for consistent deployments: `aws cloudformation create-stack --parameters file://params.json`
- Add meaningful ConstraintDescription to help users understand requirements

MAY:
- Use AWS::SSM::Parameter::Value<Type> for dynamic parameter resolution
- Use Mappings instead of parameters for environment-specific values

## Common Issues

- symptoms: "Parameter 'InstanceType' failed to satisfy constraint: Value must be one of [t3.micro, t3.small]"
  diagnosis: "Parameter value not in AllowedValues list."
  resolution: "Use a value from the AllowedValues list or update the constraint."

- symptoms: "Parameters: [KeyName] must have values"
  diagnosis: "Required parameter without a default was not provided."
  resolution: "Provide the parameter value or add a Default in the template."

- symptoms: "SSM parameter /path/to/param not found"
  diagnosis: "AWS::SSM::Parameter::Value references a non-existent SSM parameter."
  resolution: "Create the SSM parameter or fix the parameter name."

## Output Format

```yaml
root_cause: "parameter_issue — <parameter_name>: <constraint_violation>"
evidence:
  - type: error_message
    content: "<parameter validation error>"
  - type: parameter_definition
    content: "<template parameter definition>"
severity: MEDIUM
mitigation:
  immediate: "Fix the parameter value or constraint"
  long_term: "Use parameter files and add ConstraintDescription for clarity"
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
