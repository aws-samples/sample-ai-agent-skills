---
title: "B1 — Accounts Config Errors"
description: "Diagnose and resolve accounts-config.yaml configuration errors"
status: active
severity: HIGH
triggers: ["accounts config error", "account provisioning failed", "accounts-config.yaml invalid", "account not created"]
owner: devops-agent
objective: "Fix accounts-config.yaml errors to enable successful account provisioning"
context: "The accounts-config.yaml defines AWS accounts to be provisioned, their OU placement, and account-level settings. Errors in this file cause account provisioning failures and can block the entire pipeline at the Accounts stage."
---

## Phase 1 — Triage

MUST:
- Download and review accounts-config.yaml: `aws s3 cp s3://aws-accelerator-config-<account-id>-<region>/accounts-config.yaml ./`
- Validate YAML syntax (no tabs, correct indentation)
- Check account email uniqueness across all defined accounts

SHOULD:
- Verify OU names referenced in accounts-config match organization-config.yaml
- Check existing accounts: `aws organizations list-accounts --query 'Accounts[*].{Id:Id,Name:Name,Email:Email,Status:Status}'`
- Review CodeBuild logs for the Accounts stage for specific errors

MAY:
- Compare with a known-good accounts-config.yaml from a previous successful deployment
- Check for account name length and character restrictions

## Phase 2 — Remediate

MUST:
- Fix YAML syntax errors (indentation, quoting, special characters)
- Ensure all account emails are unique and valid
- Verify OU references match exactly (case-sensitive)

SHOULD:
- Validate the configuration against the LZA schema documentation
- Ensure mandatory accounts (Management, LogArchive, Audit) are defined
- Check for duplicate account names

MAY:
- Use a YAML linter to validate syntax before committing
- Implement a pre-commit hook for configuration validation

## Safety Ratings

```
safety_ratings:
  - "s3 cp (config download): GREEN — read-only file download"
  - "organizations list-accounts: GREEN — read-only account listing"
  - "Configuration file update: YELLOW — changes will be deployed on next pipeline run"
  - "Account provisioning: RED — creates new AWS accounts with billing implications"
```

## Escalation Conditions

- Account provisioning fails at the Control Tower level despite correct configuration
- Account email domain is blocked by organizational policies
- Mandatory accounts (LogArchive, Audit) cannot be provisioned
- Account limit reached and quota increase is needed

## Data Sensitivity

- accounts-config.yaml: HIGH — contains account emails and organizational structure
- `aws organizations list-accounts`: MEDIUM — reveals organization account structure
- Account email addresses: MEDIUM — organizational email addresses
- OU structure: MEDIUM — reveals organizational hierarchy

## Prohibited Actions

- NEVER remove mandatory accounts (Management, LogArchive, Audit) from the configuration
- NEVER use personal email addresses for AWS account root emails
- NEVER modify account configurations for existing production accounts without review
- NEVER commit accounts-config.yaml with sensitive data to public repositories

## Phase 3 — Rollback

1. If incorrect accounts were provisioned, they cannot be deleted — follow account closure process
2. If account configuration was changed incorrectly, revert the file and re-run the pipeline
3. If OU placement was wrong, update the configuration to move accounts to correct OUs
4. Document all account configuration changes
5. Notify account owners about any changes to their account settings

## Common Issues

- symptoms: "Pipeline fails with 'Account email already exists'"
  diagnosis: "An account with the same email address already exists in the organization."
  resolution: "Use unique email addresses. Consider email aliases (e.g., aws+accountname@company.com)."

- symptoms: "Pipeline fails with 'OU not found'"
  diagnosis: "The OU name in accounts-config.yaml does not match organization-config.yaml."
  resolution: "Check OU names are case-sensitive matches. Verify the OU is defined in organization-config.yaml."

## Output Format

```yaml
root_cause: "accounts_config — <specific_cause>"
evidence:
  - type: config_error
    content: "<configuration details and validation errors>"
severity: HIGH
mitigation:
  immediate: "Fix accounts-config.yaml"
  long_term: "Implement configuration validation pipeline"
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
