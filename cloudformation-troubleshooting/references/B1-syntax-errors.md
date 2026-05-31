---
title: "B1 — Template Syntax Errors"
description: "Diagnose and fix CloudFormation template syntax and validation errors"
status: active
severity: MEDIUM
triggers:
  - "Template format error"
  - "Template validation error"
  - "Invalid template"
  - "YAML syntax"
  - "JSON syntax"
owner: devops-agent
objective: "Identify and fix template syntax errors, structural issues, and validation failures"
context: "Template syntax errors are caught during validate-template or at the start of create/update. Common causes: invalid YAML/JSON, unknown resource types, invalid intrinsic functions, missing required properties, and incorrect template structure."
---

## Phase 1 — Triage

MUST:
- Validate the template: `aws cloudformation validate-template --template-body file://template.yaml`
- For S3-hosted templates: `aws cloudformation validate-template --template-url https://s3.amazonaws.com/<bucket>/<key>`
- Read the exact error message — it usually points to the specific location
- Check template format version: must be `AWSTemplateFormatVersion: '2010-09-09'`

SHOULD:
- Validate YAML syntax with a linter before CloudFormation validation
- Check for common YAML pitfalls: tabs vs spaces, incorrect indentation, unquoted special characters
- Verify all `Ref` and `Fn::GetAtt` targets exist in the template
- Check that all intrinsic functions use correct syntax (`!Ref` vs `Fn::Ref`)

MAY:
- Use `cfn-lint` for deeper validation: `cfn-lint template.yaml`
- Check for deprecated resource types or properties
- Validate against the CloudFormation resource specification

## Phase 2 — Remediate

MUST:
- Fix the specific syntax error reported by validate-template
- For YAML issues: fix indentation, quoting, and special characters
- For structural issues: ensure all required sections (Resources) are present
- Re-validate after each fix

SHOULD:
- Add cfn-lint to the CI/CD pipeline for pre-deployment validation
- Use an IDE with CloudFormation schema support for real-time validation

MAY:
- Convert between YAML and JSON to isolate format-specific issues
- Use `rain fmt` to auto-format CloudFormation templates

## Common Issues

- symptoms: "Template format error: YAML not well-formed"
  diagnosis: "YAML syntax error — tabs, bad indentation, or unquoted special characters."
  resolution: "Use spaces not tabs. Quote strings containing colons, brackets, or special YAML characters."

- symptoms: "Template format error: Unrecognized resource types: [AWS::Foo::Bar]"
  diagnosis: "Resource type doesn't exist or requires a transform/extension."
  resolution: "Check the resource type name. For SAM resources, add Transform: AWS::Serverless-2016-10-31."

- symptoms: "Template error: instance of Fn::GetAtt references undefined resource"
  diagnosis: "GetAtt references a logical ID that doesn't exist in the template."
  resolution: "Fix the logical ID reference or add the missing resource."

## Output Format

```yaml
root_cause: "syntax_error — <specific_error_location>"
evidence:
  - type: validation_output
    content: "<validate-template error message>"
  - type: template_section
    content: "<relevant template section>"
severity: MEDIUM
mitigation:
  immediate: "Fix the syntax error and re-validate"
  long_term: "Add cfn-lint to CI/CD pipeline"
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
