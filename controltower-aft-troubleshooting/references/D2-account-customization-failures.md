---
title: "D2 — Account Customization Failures"
description: "Diagnose and resolve AFT account-specific customization failures"
status: active
severity: HIGH
triggers: ["account customization failed", "account-specific Terraform error", "customization pipeline error for account"]
owner: devops-agent
objective: "Successfully apply account-specific Terraform customizations"
context: "Account-specific customizations run after global customizations and apply Terraform configurations to individual accounts based on their directory in the customizations repository. Failures are isolated to specific accounts."
---

## Phase 1 — Triage

MUST:
- Check account-customizations pipeline: `aws codepipeline get-pipeline-state --name ct-aft-account-customizations`
- Get failed build details: `aws codebuild list-builds-for-project --project-name aft-account-customizations --max-items 5`
- Identify which account's customization failed from CodeBuild logs

SHOULD:
- Verify the account-specific directory exists in the customizations repository
- Check Terraform code in the account's customization directory
- Verify global customizations succeeded (they must pass first)

MAY:
- Compare the failing account's customization with a working account's customization
- Check for account-specific resource limits or restrictions

## Phase 2 — Remediate

MUST:
- Fix Terraform code in the account-specific customization directory
- Ensure the AWSAFTExecution role in the target account has required permissions
- Verify global customizations completed successfully before re-running account customizations

SHOULD:
- Test the customization Terraform code locally with the target account role
- Check for resource naming conflicts specific to the target account
- Verify provider configuration correctly assumes the target account role

MAY:
- Implement account-specific variable files for environment differences
- Add pre-apply validation scripts to the customization repository

## Safety Ratings

```
safety_ratings:
  - "codepipeline get-pipeline-state: GREEN — read-only status check"
  - "codebuild batch-get-builds: GREEN — read-only build details"
  - "Account-specific Terraform apply: YELLOW — modifies resources in a single target account"
  - "terraform plan for target account: GREEN — read-only preview"
```

## Escalation Conditions

- Account customization fails due to Control Tower guardrail restrictions
- AWSAFTExecution role is missing or corrupted in the target account
- Customization requires cross-account resources that need additional permissions
- Multiple account customizations fail with the same error

## Data Sensitivity

- CodeBuild logs: HIGH — contain Terraform output with account-specific resource details
- Account customization code: MEDIUM — reveals account-specific infrastructure
- Target account Terraform state: HIGH — contains resource configuration
- AWSAFTExecution role policy: MEDIUM — reveals cross-account permission scope

## Prohibited Actions

- NEVER apply customizations directly in the target account bypassing AFT
- NEVER copy another account's customization without reviewing for account-specific values
- NEVER delete the account's Terraform state to "fix" customization issues
- NEVER modify the AWSAFTExecution role manually in target accounts

## Phase 3 — Rollback

1. If incorrect resources were created, update the customization code and re-run the pipeline
2. If Terraform state is inconsistent, use `terraform import` or `terraform state rm` to reconcile
3. If the customization needs to be completely removed, empty the account's directory and re-run
4. Document all changes and notify the account owner
5. Verify the target account is in a consistent state after rollback

## Common Issues

- symptoms: "Account customization fails with 'Error assuming role AWSAFTExecution'"
  diagnosis: "Role does not exist or trust policy does not include the AFT management account."
  resolution: "Check role in target account. If missing, re-run account provisioning to recreate it."

- symptoms: "Customization succeeds but resources are not visible in target account"
  diagnosis: "Provider region configuration does not match the expected deployment region."
  resolution: "Check provider block in customization code. Ensure region is correctly set or inherited."

## Output Format

```yaml
root_cause: "account_customization — <specific_cause>"
evidence:
  - type: customization_error
    content: "<account ID, CodeBuild logs, and Terraform error>"
severity: HIGH
mitigation:
  immediate: "Fix account-specific customization code"
  long_term: "Implement customization testing and validation"
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
