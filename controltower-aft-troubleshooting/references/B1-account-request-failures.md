---
title: "B1 — Account Request Failures"
description: "Diagnose and resolve AFT account request and provisioning failures"
status: active
severity: CRITICAL
triggers: ["account request failed", "account not provisioned", "account creation error", "AFT account error"]
owner: devops-agent
objective: "Successfully provision new AWS accounts through AFT account requests"
context: "AFT account requests are submitted via the aft-account-request repository. The request triggers a pipeline that validates the request, submits it to Control Tower Account Factory, and tracks status in DynamoDB. Failures can occur at validation, submission, or provisioning stages."
---

## Phase 1 — Triage

MUST:
- Check account request status in DynamoDB: `aws dynamodb scan --table-name aft-request --filter-expression 'contains(id, :val)' --expression-attribute-values '{":val":{"S":"<request-id>"}}'`
- Check account-request pipeline status: `aws codepipeline get-pipeline-state --name ct-aft-account-request`
- Verify the account request file in the repository is valid JSON/HCL

SHOULD:
- Check Control Tower account factory for provisioning status: `aws servicecatalog search-provisioned-products --filters SearchQuery=<account-name>`
- Review CodeBuild logs for the account-request pipeline
- Verify the target OU exists: `aws organizations list-organizational-units-for-parent --parent-id <root-or-parent-ou-id>`

MAY:
- Check if the account email is already in use: `aws organizations list-accounts --query 'Accounts[?Email==`<email>`]'`
- Review CloudTrail for CreateAccount or CreateManagedAccount events

## Phase 2 — Remediate

MUST:
- Fix account request file syntax and required fields (account name, email, OU, SSO)
- Ensure the account email is unique and not already used by another AWS account
- Verify the target OU exists and is registered with Control Tower

SHOULD:
- Check SSO permission set names referenced in the request exist: `aws sso-admin list-permission-sets --instance-arn <sso-instance-arn>`
- Verify account name follows naming conventions (no special characters)
- Ensure the AFT management account has permissions to invoke Control Tower APIs

MAY:
- Submit a test account request with minimal configuration to isolate the issue
- Check Control Tower guardrails that may block account provisioning

## Safety Ratings

```
safety_ratings:
  - "dynamodb scan: GREEN — read-only table scan"
  - "codepipeline get-pipeline-state: GREEN — read-only status check"
  - "organizations list-accounts: GREEN — read-only account listing"
  - "Account provisioning: RED — creates a new AWS account with billing implications"
```

## Escalation Conditions

- Account provisioning fails at the Control Tower level with internal errors
- Account email domain is not verified or allowed by organizational policies
- SCP prevents account creation in the target OU
- Control Tower guardrail enrollment fails for the new account

## Data Sensitivity

- `aws dynamodb scan --table-name aft-request`: HIGH — contains account emails and configuration
- `aws organizations list-accounts`: MEDIUM — reveals organization account structure
- Account request files: HIGH — contain account emails, SSO assignments, and OU placement
- `aws sso-admin list-permission-sets`: MEDIUM — reveals SSO configuration

## Prohibited Actions

- NEVER submit duplicate account requests without verifying the first request's status
- NEVER use personal email addresses for AWS account root emails
- NEVER bypass Control Tower by creating accounts directly via Organizations API
- NEVER delete entries from the aft-request DynamoDB table

## Phase 3 — Rollback

1. If an account was provisioned incorrectly, do NOT delete it — follow the account closure process
2. If the account request pipeline is stuck, check DynamoDB for lock entries and clear if stale
3. If incorrect SSO assignments were made, update the account request and re-run customizations
4. Document the provisioning attempt and any manual interventions
5. Notify the Control Tower admin about failed provisioning attempts

## Common Issues

- symptoms: "Account request pipeline succeeds but account is not created"
  diagnosis: "Control Tower Account Factory provisioning failed silently. Check Service Catalog."
  resolution: "Check `aws servicecatalog search-provisioned-products` for the account. Review CT provisioning events."

- symptoms: "ValidationError: Account email already exists"
  diagnosis: "The email address is already associated with an existing AWS account."
  resolution: "Use a unique email address. Consider using email aliases (e.g., aws+account1@company.com)."

## Output Format

```yaml
root_cause: "account_request — <specific_cause>"
evidence:
  - type: request_error
    content: "<DynamoDB status, pipeline error, or CT provisioning details>"
severity: CRITICAL
mitigation:
  immediate: "Fix account request configuration"
  long_term: "Implement account request validation before submission"
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
