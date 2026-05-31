---
title: "F1 — Custom Resource Cards"
description: "Diagnose custom resource card issues in Application Composer"
status: active
severity: LOW
triggers: ["custom resource error", "generic card", "custom resource not rendering", "Custom:: resource"]
owner: devops-agent
objective: "Resolve custom resource card rendering and configuration issues"
context: "Custom::* resources and AWS::CloudFormation::CustomResource render as generic cards with limited property editing."
---

## Phase 1 — Triage

MUST:
- Identify custom resources in the template: `grep -n 'Custom::' template.yaml`
- Check if the resource type has a CloudFormation schema: `aws cloudformation describe-type --type RESOURCE --type-name <type-name>`
- Validate the template with custom resources: `sam validate --template-file template.yaml --lint`

SHOULD:
- Verify the custom resource Lambda function exists and is functional
- Check ServiceToken property points to a valid Lambda ARN or SNS topic

MAY:
- Review CloudFormation custom resource documentation for the specific type
- Check if a third-party resource provider is registered

## Phase 2 — Remediate

MUST:
- Ensure ServiceToken is a valid Lambda function ARN or SNS topic ARN
- Configure custom resource properties in the template source view (not visual editor)
- Verify the backing Lambda function handles Create, Update, and Delete events

SHOULD:
- Use AWS::CloudFormation::CustomResource type for standard custom resources
- Implement proper response handling in the Lambda function

MAY:
- Register a custom resource type with CloudFormation Registry for enhanced card support
- Use CloudFormation modules for reusable custom resource patterns

## Safety Ratings

```
safety_ratings:
  - "grep template.yaml: GREEN — read-only template search"
  - "cloudformation describe-type: GREEN — read-only schema retrieval"
  - "sam validate: GREEN — read-only template validation"
  - "cloudformation register-type: YELLOW — registers new resource type"
```

## Escalation Conditions

- Custom resource Lambda function fails during stack operations causing stack hangs
- Custom resource does not send response to CloudFormation, causing 1-hour timeout
- Registered resource type conflicts with existing types
- Custom resource creates resources outside CloudFormation management

## Data Sensitivity

- `grep template.yaml`: LOW — searches for resource type patterns
- `aws cloudformation describe-type`: LOW — retrieves public resource schema
- Custom resource Lambda code: MEDIUM — may contain business logic and API integrations

## Prohibited Actions

- NEVER delete a custom resource without understanding what external resources it manages
- NEVER modify ServiceToken to point to a non-existent Lambda function
- NEVER skip implementing the Delete handler in custom resource Lambda functions
- NEVER assume custom resources are managed by CloudFormation drift detection

## Phase 3 — Rollback

1. If a custom resource was misconfigured, restore from version control: `git checkout -- template.yaml`
2. If the custom resource Lambda was modified, restore the function code
3. If a stack is stuck waiting for custom resource response, send manual response via signed URL
4. Verify custom resource-managed external resources are in expected state
5. Document custom resource behavior and dependencies

## Common Issues

- symptoms: "Custom resource shows as a generic card with no property editing"
  diagnosis: "Application Composer does not have an enhanced card for Custom::* resource types."
  resolution: "Edit custom resource properties in the template source view. Consider registering the type with CloudFormation Registry for better support."

## Output Format

```yaml
root_cause: "custom_resource — <specific_cause>"
evidence:
  - type: resource_error
    content: "<error details>"
severity: LOW
mitigation:
  immediate: "Fix custom resource configuration"
  long_term: "Register custom types with CloudFormation Registry"
```

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
