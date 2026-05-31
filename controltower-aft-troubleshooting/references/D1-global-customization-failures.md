---
title: "D1 — Global Customization Failures"
description: "Diagnose and resolve AFT global customization pipeline and Terraform failures"
status: active
severity: HIGH
triggers: ["global customization failed", "global Terraform error", "global pipeline failed", "all accounts customization error"]
owner: devops-agent
objective: "Successfully apply global customizations across all AFT-managed accounts"
context: "Global customizations apply Terraform configurations to ALL AFT-managed accounts. They run before account-specific customizations. A global customization failure blocks all account customizations and affects every managed account."
---

## Phase 1 — Triage

MUST:
- Check global-customizations pipeline: `aws codepipeline get-pipeline-state --name ct-aft-global-customizations`
- Get failed build details: `aws codebuild list-builds-for-project --project-name aft-global-customizations --max-items 5`
- Review CodeBuild logs for Terraform errors: `aws codebuild batch-get-builds --ids <build-id>`

SHOULD:
- Check the global customizations repository for recent changes
- Verify Terraform code syntax: `terraform validate`
- Check if the failure affects all accounts or specific ones

MAY:
- Review Terraform plan output for unexpected changes
- Check for resource quota limits that may be hit across multiple accounts

## Phase 2 — Remediate

MUST:
- Fix Terraform code errors in the global customizations repository
- Ensure the AWSAFTExecution role in target accounts has permissions for global resources
- Resolve any resource conflicts across managed accounts

SHOULD:
- Test global customization changes in a single account before applying globally
- Review Terraform plan for each affected account
- Check for provider configuration issues in the global customizations

MAY:
- Implement a staged rollout for global customizations (subset of accounts first)
- Add Terraform validation to the repository's CI pipeline

## Safety Ratings

```
safety_ratings:
  - "codepipeline get-pipeline-state: GREEN — read-only status check"
  - "codebuild batch-get-builds: GREEN — read-only build details"
  - "Global Terraform apply: RED — modifies resources in ALL managed accounts"
  - "terraform validate: GREEN — syntax check only"
```

## Escalation Conditions

- Global customization failure blocks all account customizations
- Terraform apply partially succeeded in some accounts but failed in others (inconsistent state)
- Global customization requires resources that conflict with Control Tower guardrails
- Rollback of global customization would cause service disruption

## Data Sensitivity

- CodeBuild logs: HIGH — contain Terraform output for multiple accounts
- Global customization code: MEDIUM — reveals organization-wide infrastructure patterns
- Terraform state for global customizations: HIGH — contains resource details across accounts
- Account list affected by global customizations: MEDIUM — reveals organization structure

## Prohibited Actions

- NEVER apply untested global customizations to production accounts
- NEVER manually apply global customizations in individual accounts (bypassing AFT)
- NEVER delete global customization Terraform state
- NEVER ignore partial failures — inconsistent state across accounts is dangerous

## Phase 3 — Rollback

1. If global customization applied incorrect resources, revert the repository commit and re-run the pipeline
2. For partial failures, identify which accounts succeeded and which failed
3. Manually reconcile accounts in inconsistent state using targeted Terraform operations
4. Document all affected accounts and their current state
5. Notify account owners about any changes or rollbacks

## Common Issues

- symptoms: "Global customization succeeds for some accounts but fails for others"
  diagnosis: "Account-specific differences (existing resources, different permissions, quota limits)."
  resolution: "Review failed accounts individually. Use conditional logic in Terraform for account-specific variations."

- symptoms: "Global customization pipeline runs but makes no changes"
  diagnosis: "Terraform code is valid but produces an empty plan (no changes needed)."
  resolution: "Verify the customization code references correct resources. Check provider assume_role configuration."

## Output Format

```yaml
root_cause: "global_customization — <specific_cause>"
evidence:
  - type: customization_error
    content: "<pipeline status, CodeBuild logs, and affected accounts>"
severity: HIGH
mitigation:
  immediate: "Fix global customization code"
  long_term: "Implement staged rollout and pre-apply validation"
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
