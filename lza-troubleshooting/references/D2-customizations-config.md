---
title: "D2 — Customizations Config"
description: "Diagnose and resolve customizations-config.yaml configuration errors"
status: active
severity: MEDIUM
triggers: ["customizations config error", "custom stack failed", "customizations-config.yaml invalid", "custom resource error"]
owner: devops-agent
objective: "Fix customizations-config.yaml errors to enable custom resource deployment"
context: "The customizations-config.yaml extends LZA with custom CloudFormation stacks and SSM documents. It adds resources on top of LZA-managed infrastructure but does NOT override core LZA resources. Customization errors are isolated from core LZA functionality."
---

## Phase 1 — Triage

MUST:
- Download and review customizations-config.yaml: `aws s3 cp s3://aws-accelerator-config-<account-id>-<region>/customizations-config.yaml ./`
- Check CloudFormation stack events for custom stacks: `aws cloudformation describe-stack-events --stack-name <custom-stack-name> --max-items 20`
- Verify custom CloudFormation templates exist in the S3 bucket

SHOULD:
- Validate custom CloudFormation template syntax
- Check target account and region specifications
- Review CodeBuild logs for the Customizations stage

MAY:
- Test custom CloudFormation templates independently before LZA deployment
- Check for resource naming conflicts with LZA-managed resources

## Phase 2 — Remediate

MUST:
- Fix YAML syntax and schema errors in customizations-config.yaml
- Ensure custom CloudFormation templates are valid
- Verify target accounts and regions are correct

SHOULD:
- Validate custom templates do not conflict with LZA-managed resources
- Check CloudFormation parameter values are correct
- Ensure custom stack dependencies are properly ordered

MAY:
- Deploy custom stacks independently for testing before integrating with LZA
- Implement template validation in a pre-commit hook

## Safety Ratings

```
safety_ratings:
  - "s3 cp (config download): GREEN — read-only file download"
  - "cloudformation describe-stack-events: GREEN — read-only event history"
  - "Custom stack deployment: YELLOW — creates custom resources in target accounts"
  - "cloudformation validate-template: GREEN — read-only template validation"
```

## Escalation Conditions

- Custom stacks conflict with LZA-managed resources causing deployment failures
- Custom resources require permissions not available in the LZA execution role
- Custom stacks create resources that violate organizational SCPs
- Custom stack failures block the entire LZA pipeline

## Data Sensitivity

- customizations-config.yaml: MEDIUM — reveals custom infrastructure extensions
- Custom CloudFormation templates: MEDIUM — reveal custom resource configurations
- Custom stack parameters: HIGH — may contain sensitive configuration values
- Target account details: MEDIUM — reveals deployment scope

## Prohibited Actions

- NEVER use customizations to override LZA-managed resources
- NEVER deploy custom stacks that create resources conflicting with core LZA
- NEVER include secrets in custom CloudFormation template parameters
- NEVER skip template validation before deployment

## Phase 3 — Rollback

1. If custom stacks deployed incorrect resources, update the template and re-deploy
2. If custom stacks need to be removed, remove them from customizations-config.yaml and re-run
3. If custom stacks caused conflicts with LZA resources, remove the conflicting resources
4. Document all customization changes and their impact
5. Verify LZA core functionality is not affected by customization changes

## Common Issues

- symptoms: "Custom CloudFormation stack fails with 'Template validation error'"
  diagnosis: "Custom CloudFormation template has syntax errors or invalid resource types."
  resolution: "Validate template: `aws cloudformation validate-template --template-body file://template.yaml`. Fix errors."

- symptoms: "Custom stack creates resources that conflict with LZA-managed resources"
  diagnosis: "Customization attempts to manage resources already managed by core LZA configuration."
  resolution: "Remove conflicting resources from customizations. Modify core config files for LZA-managed resources."

## Output Format

```yaml
root_cause: "customizations_config — <specific_cause>"
evidence:
  - type: customization_error
    content: "<template details, stack events, and configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix customizations-config.yaml or templates"
  long_term: "Implement template validation and testing pipeline"
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
