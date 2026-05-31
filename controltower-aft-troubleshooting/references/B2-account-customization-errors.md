---
title: "B2 — Account Customization Errors"
description: "Diagnose and resolve errors during AFT account customization application"
status: active
severity: HIGH
triggers: ["customization failed", "account customization error", "Terraform apply failed on account", "post-provisioning error"]
owner: devops-agent
objective: "Successfully apply Terraform customizations to provisioned accounts"
context: "After account provisioning, AFT applies customizations via Terraform. Customization errors are typically Terraform plan/apply failures caused by IAM permission issues, resource conflicts, or Terraform code errors in the customization repositories."
---

## Phase 1 — Triage

MUST:
- Check account-customizations pipeline: `aws codepipeline get-pipeline-state --name ct-aft-account-customizations`
- Get failed build details: `aws codebuild list-builds-for-project --project-name aft-account-customizations --max-items 5`
- Review CodeBuild logs for Terraform errors: `aws codebuild batch-get-builds --ids <build-id>`

SHOULD:
- Check if global customizations succeeded first (they run before account customizations)
- Verify the account-specific customization repository has valid Terraform code
- Check the target account's IAM role for AFT: `aws iam get-role --role-name AWSAFTExecution`

MAY:
- Review Terraform plan output for the specific account
- Check for resource naming conflicts in the target account

## Phase 2 — Remediate

MUST:
- Fix Terraform code errors in the customization repository
- Ensure the AWSAFTExecution role exists in the target account with correct permissions
- Resolve resource conflicts (duplicate names, existing resources)

SHOULD:
- Test Terraform code locally before committing to the repository
- Verify provider configuration assumes the correct role in the target account
- Check for Terraform state drift in the target account

MAY:
- Use `terraform plan` in isolation to validate changes before pipeline execution
- Implement Terraform validation in a pre-commit hook

## Safety Ratings

```
safety_ratings:
  - "codepipeline get-pipeline-state: GREEN — read-only status check"
  - "codebuild batch-get-builds: GREEN — read-only build details"
  - "iam get-role: GREEN — read-only role check"
  - "Terraform apply in target account: RED — modifies target account resources"
```

## Escalation Conditions

- AWSAFTExecution role is missing or has insufficient permissions in the target account
- Terraform state for the target account is corrupted
- Customization requires resources that conflict with Control Tower guardrails
- Multiple accounts fail customization simultaneously

## Data Sensitivity

- CodeBuild logs: HIGH — contain Terraform output with resource details and potential secrets
- Terraform state for target accounts: HIGH — contains resource configuration
- AWSAFTExecution role policy: MEDIUM — reveals cross-account permission scope
- Customization repository code: MEDIUM — reveals infrastructure configuration

## Prohibited Actions

- NEVER apply customizations directly in the target account bypassing AFT
- NEVER modify the AWSAFTExecution role without understanding the impact on AFT operations
- NEVER delete Terraform state for target accounts
- NEVER commit secrets or credentials to customization repositories

## Phase 3 — Rollback

1. If customization applied incorrect resources, update the Terraform code and re-run the pipeline
2. If Terraform state is inconsistent, use `terraform import` to reconcile
3. If the AWSAFTExecution role was modified, restore from the AFT-managed configuration
4. Document all customization failures and manual interventions
5. Verify the target account is in a consistent state after rollback

## Common Issues

- symptoms: "Terraform apply fails with 'Error assuming role AWSAFTExecution'"
  diagnosis: "The AWSAFTExecution role does not exist in the target account or trust policy is incorrect."
  resolution: "Verify role exists: `aws iam get-role --role-name AWSAFTExecution`. Check trust policy allows the AFT management account."

- symptoms: "Resource already exists error during customization"
  diagnosis: "Resources were created manually or by another process before AFT customization ran."
  resolution: "Import existing resources into Terraform state or modify code to use data sources for existing resources."

## Output Format

```yaml
root_cause: "account_customization — <specific_cause>"
evidence:
  - type: customization_error
    content: "<CodeBuild logs, Terraform error, and target account details>"
severity: HIGH
mitigation:
  immediate: "Fix customization code or IAM configuration"
  long_term: "Implement Terraform validation and testing pipeline"
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
