---
title: "A1 — Account Creation Failures"
description: "Diagnose why AWS account creation via Organizations is failing"
status: active
severity: HIGH
triggers:
  - "account creation failed"
  - "create account error"
  - "cannot create account"
  - "CreateAccountStatus FAILED"
owner: devops-agent
objective: "Identify the root cause of account creation failures and successfully create the account"
context: "Account creation via Organizations is asynchronous. The create-account API returns a request ID, and the actual creation happens in the background. Failures can be due to email uniqueness, account limits, billing issues, or internal errors. The management account must have a valid payment method."
---

## Phase 1 — Triage

MUST:
- Check account creation status: `aws organizations list-create-account-status --states FAILED --max-results 10`
- Describe the specific failed request: `aws organizations describe-create-account-status --create-account-request-id <request-id>`
- Check the FailureReason field for the specific error
- Verify the email is not already in use: `aws organizations list-accounts --query 'Accounts[?Email==`<email>`]'`

SHOULD:
- Check organization account limits: `aws organizations describe-organization`
- Verify the management account has a valid payment method
- Check CloudTrail for the creation event: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateAccount --max-results 10`

MAY:
- Check for service quota limits on accounts: `aws service-quotas get-service-quota --service-code organizations --quota-code L-29A0C5DF`
- Verify no SCPs are blocking account creation actions

## Phase 2 — Remediate

MUST:
- Fix the identified failure reason (email, billing, limits)
- Use a unique email address for the new account
- Retry account creation after fixing the issue

SHOULD:
- Use email aliases (user+tag@domain.com) for creating multiple accounts
- Request an account limit increase if approaching the quota
- Ensure the management account billing is in good standing

MAY:
- Use AWS Control Tower Account Factory for standardized account creation
- Implement account creation automation with proper error handling

## Common Issues

- symptoms: "ACCOUNT_LIMIT_EXCEEDED"
  diagnosis: "The organization has reached the maximum number of accounts."
  resolution: "Request a quota increase via Service Quotas or AWS Support."

- symptoms: "EMAIL_ALREADY_EXISTS"
  diagnosis: "The email address is already associated with another AWS account."
  resolution: "Use a different email address. Try email aliases (user+tag@domain.com)."

- symptoms: "INTERNAL_FAILURE"
  diagnosis: "AWS internal error during account creation."
  resolution: "Wait and retry. If persistent, contact AWS Support."

- symptoms: "INVALID_EMAIL"
  diagnosis: "The email address format is invalid or the domain cannot receive email."
  resolution: "Use a valid email address that can receive verification emails."

## Output Format

```yaml
root_cause: "account_creation_failure — <specific_cause>"
evidence:
  - type: creation_status
    content: "<describe-create-account-status output>"
  - type: failure_reason
    content: "<specific failure reason>"
severity: HIGH
mitigation:
  immediate: "Fix the identified issue and retry account creation"
  long_term: "Implement account creation automation with validation"
```


## Safety Ratings
- GREEN: describe-organization, list-accounts, list-policies
- YELLOW: attach-policy, create-organizational-unit, move-account
- RED: close-account, delete-policy, remove-account-from-organization

## Escalation Conditions
- "Remediation involves modifying SCPs in production"
- "Fix requires moving accounts between OUs"
- "Change affects the management account"
- "SCP modification could lock out member accounts"

## Rollback
- Pre-change: "Document current SCP attachments and OU structure"
- Verification: "Test affected account access after SCP change"
- Revert: "Detach modified SCP and reattach original"

## Data Sensitivity
- HIGH: "list-accounts reveals all account IDs, names, and emails"
- HIGH: "SCP content reveals security boundaries"
- MEDIUM: "OU structure reveals organizational hierarchy"

## Prohibited Actions
- NEVER suggest removing all SCPs from an OU
- NEVER suggest closing an account without confirming data backup
- NEVER suggest modifying the management account's policies
- NEVER suggest removing an account from the organization without root email access confirmation

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
  - command: "list-accounts"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-organization"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-policies"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest removing SCPs without understanding blast radius"
  - "NEVER suggest disabling AI opt-out policies without review"
  - "NEVER modify organization structure without approval"

## Phase 3 � Rollback

- Document the current state before making any changes
- If remediation worsens the issue, revert to the documented pre-change state
- Remove any temporary resources created during investigation
- Verify the system returns to its previous functional state after rollback
