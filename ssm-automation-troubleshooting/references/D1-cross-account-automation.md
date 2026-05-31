---
title: "D1 — Cross-Account Automation"
description: "Diagnose cross-account SSM Automation execution failures including Organizations, IAM roles, and trust policies"
status: active
severity: HIGH
triggers: ["cross-account failed", "target account error", "AssumeRole failed", "multi-account automation"]
owner: devops-agent
objective: "Resolve cross-account automation execution failures"
context: "Cross-account automation requires AWS Organizations. The management account or delegated admin initiates executions that assume roles in target accounts. Target accounts must have IAM roles with trust policies."
---

## Phase 1 — Triage

MUST:
- Verify Organizations is enabled: `aws organizations describe-organization`
- Check if caller is management account or delegated admin: `aws organizations list-delegated-administrators --service-principal ssm.amazonaws.com`
- Get execution details: `aws ssm get-automation-execution --automation-execution-id <execution-id>`

SHOULD:
- Verify target account IAM role exists: `aws sts assume-role --role-arn arn:aws:iam::<target-account>:role/<role-name> --role-session-name test` (from management account)
- Check the target role trust policy allows the management account
- Review the automation document's TargetLocations configuration

MAY:
- Check CloudTrail in the target account for AssumeRole failures
- Verify the target account is an active member of the organization: `aws organizations list-accounts`

## Phase 2 — Remediate

MUST:
- Create or fix IAM roles in target accounts with correct trust policies
- Ensure the management account or delegated admin initiates the execution
- Verify the TargetLocations parameter specifies correct account IDs and regions

SHOULD:
- Use CloudFormation StackSets to deploy consistent IAM roles across target accounts
- Configure the automation document with explicit TargetLocations
- Test cross-account access with sts:AssumeRole before running automation

MAY:
- Set up delegated admin to avoid using the management account directly
- Implement resource data sync for cross-account visibility

## Safety Ratings

```
safety_ratings:
  - "organizations describe-organization: GREEN — read-only org metadata"
  - "sts assume-role: YELLOW — tests cross-account access, creates temporary credentials"
  - "start-automation-execution with TargetLocations: RED — executes automation in other accounts"
  - "iam create-role in target account: YELLOW — creates IAM role with cross-account trust"
```

## Escalation Conditions

- Cross-account automation affects production accounts and fails mid-execution
- Target account IAM role has overly broad permissions that need security review
- Organizations structure changes affect automation trust relationships
- Delegated admin account change required

## Data Sensitivity

- `aws organizations describe-organization`: LOW — returns organization metadata
- `aws organizations list-accounts`: MEDIUM — reveals all account IDs and names
- `aws sts assume-role`: HIGH — creates temporary credentials for another account
- `aws iam get-role`: MEDIUM — reveals trust policy and permissions

## Prohibited Actions

- NEVER create cross-account IAM roles with AdministratorAccess
- NEVER use the management account for routine automation — use delegated admin
- NEVER hardcode account IDs in automation documents — use parameters or Organizations queries
- NEVER run cross-account automation against accounts without their team's knowledge

## Phase 3 — Rollback

1. Stop cross-account execution: `aws ssm stop-automation-execution --automation-execution-id <id> --type Cancel`
2. Check child executions in each target account for partial changes
3. Coordinate with target account teams to revert changes
4. If IAM roles were created incorrectly, remove them from target accounts
5. Document which accounts were affected and what changes were made

## Common Issues

- symptoms: "Execution fails with AssumeRole AccessDenied for target account"
  diagnosis: "The IAM role in the target account does not exist or its trust policy does not allow the management account."
  resolution: "Create the role in the target account with a trust policy allowing the management account or delegated admin."

- symptoms: "Cross-account execution fails with OrganizationsException"
  diagnosis: "The caller is not the management account or a delegated admin for SSM."
  resolution: "Use the management account or register a delegated admin: aws organizations register-delegated-administrator."

- symptoms: "Automation succeeds in some accounts but fails in others"
  diagnosis: "IAM roles or permissions differ across target accounts."
  resolution: "Use CloudFormation StackSets to ensure consistent IAM role deployment across all target accounts."

## Output Format

```yaml
root_cause: "cross_account — <specific_cause>"
evidence:
  - type: cross_account_error
    content: "<AssumeRole or Organizations error details>"
severity: HIGH
mitigation:
  immediate: "Fix cross-account IAM configuration"
  long_term: "Standardize IAM roles via StackSets"
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
  - "NEVER suggest overly broad SSM document permissions"
  - "NEVER suggest disabling session logging"
  - "NEVER suggest sharing SSM documents publicly"
