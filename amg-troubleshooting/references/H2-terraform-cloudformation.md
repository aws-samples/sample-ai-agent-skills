---
title: "H2 — Terraform/CloudFormation Provisioning Issues"
description: "Diagnose IaC provisioning problems with Amazon Managed Grafana"
status: active
severity: MEDIUM
triggers:
  - "Terraform"
  - "CloudFormation"
  - "provisioning failed"
  - "IaC error"
  - "stack failed"
owner: devops-agent
objective: "Identify why Terraform or CloudFormation fails to provision or update AMG resources and fix the IaC configuration"
context: "AMG workspaces can be provisioned via Terraform (aws_grafana_workspace) or CloudFormation (AWS::Grafana::Workspace). Common failures include IAM role issues, invalid parameter combinations, dependency ordering, and state drift. The Grafana Terraform provider can also manage dashboards, data sources, and alerts."
---

## Phase 1 — Triage

MUST:
- Review the Terraform plan output or CloudFormation stack events for error messages
- Check if the IAM role specified in the IaC template exists and has correct trust policy
- Verify parameter values match AMG API requirements (authentication types, data source types)
- Check for resource dependencies (IAM role must exist before workspace creation)

SHOULD:
- For Terraform: check state for drift: `terraform plan`
- For CloudFormation: check stack events: `aws cloudformation describe-stack-events --stack-name <stack-name>`
- Verify the AWS provider version supports the AMG resource attributes being used

MAY:
- Check Terraform provider documentation for aws_grafana_workspace resource
- Review CloudFormation resource documentation for AWS::Grafana::Workspace
- Check if the workspace was modified outside of IaC causing state drift

## Phase 2 — Remediate

MUST:
- Fix IAM role configuration in the IaC template
- Correct invalid parameter values based on the error message
- Ensure resource dependencies are properly defined (depends_on in Terraform, DependsOn in CloudFormation)

SHOULD:
- Import existing resources into Terraform state if they were created manually: `terraform import aws_grafana_workspace.example <workspace-id>`
- Use data sources to reference existing IAM roles and VPC resources
- Pin provider versions to avoid breaking changes

MAY:
- Use the Grafana Terraform provider for dashboard and data source provisioning
- Implement CI/CD pipeline for IaC deployment with validation steps

## Common Issues

- symptoms: "Terraform apply fails with 'InvalidParameterException'"
  diagnosis: "Invalid combination of parameters (e.g., specifying both SSO and SAML, or invalid data source type)."
  resolution: "Review the parameter values against AMG API documentation. Fix the invalid parameter combination."

- symptoms: "CloudFormation stack fails with 'Resource creation failed'"
  diagnosis: "IAM role does not exist, trust policy is incorrect, or service-linked role is missing."
  resolution: "Create the IAM role with correct trust policy before the workspace. Add the service-linked role as a dependency."

- symptoms: "Terraform plan shows unexpected changes on every run"
  diagnosis: "State drift due to manual changes in the console, or the provider normalizes values differently."
  resolution: "Import the current state. Avoid manual changes. Use lifecycle ignore_changes for known drift attributes."

## Safety Ratings

```
safety_ratings:
  - "review Terraform plan output: GREEN — read-only analysis"
  - "describe-stack-events: GREEN — read-only CloudFormation check"
  - "terraform import: YELLOW — modifies Terraform state"
  - "terraform apply: RED — provisions or modifies infrastructure"
  - "CloudFormation stack update: RED — modifies infrastructure"
```

## Escalation Conditions

- IaC provisioning failure blocks production workspace deployment
- State drift detected between IaC and actual workspace configuration
- Terraform import needed for manually created production workspaces
- CloudFormation stack rollback fails, leaving resources in inconsistent state

## Data Sensitivity

- IaC configuration: MEDIUM — reveals infrastructure architecture and IAM roles
- Terraform state: HIGH — contains resource IDs, ARNs, and configuration details
- CloudFormation stack events: MEDIUM — reveals provisioning history and errors

## Prohibited Actions

- NEVER suggest creating local Grafana users (not supported)
- NEVER suggest downgrading Grafana version
- NEVER run terraform destroy on production workspaces without explicit approval
- NEVER modify Terraform state manually (use terraform import/state commands)

## Phase 3 — Rollback

1. For Terraform: revert the configuration change and run `terraform apply` to restore previous state
2. For CloudFormation: update the stack with the previous template, or roll back: `aws cloudformation rollback-stack --stack-name <name>`
3. If terraform import was done incorrectly, remove from state: `terraform state rm <resource>`
4. Verify the workspace configuration matches the intended IaC state
5. Document the IaC failure and fix for future deployments

## Output Format

```yaml
root_cause: "iac_provisioning — <specific_cause>"
evidence:
  - type: error_message
    content: "<Terraform/CloudFormation error details>"
  - type: iac_config
    content: "<relevant IaC resource configuration>"
  - type: api_validation
    content: "<parameter validation results>"
severity: MEDIUM
mitigation:
  immediate: "Fix IaC configuration and re-apply"
  long_term: "Implement IaC validation in CI/CD pipeline and prevent manual changes"
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
