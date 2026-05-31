---
title: "A2 — Account Closure Issues"
description: "Diagnose account closure failures and recovery scenarios"
status: active
severity: HIGH
triggers:
  - "close account"
  - "account closure"
  - "cannot close account"
  - "reopen account"
  - "suspended account"
owner: devops-agent
objective: "Successfully close accounts or recover accounts in the suspension period"
context: "Account closure puts the account in a suspended state for 90 days. During this period, resources persist but are inaccessible, and the account can be reopened. After 90 days, permanent deletion occurs. Some charges may continue during suspension. Only the management account or the account itself can initiate closure."
---

## Phase 1 — Triage

MUST:
- Check account status: `aws organizations describe-account --account-id <account-id>`
- Verify the account is in ACTIVE or SUSPENDED state
- Check if the caller has permission to close the account
- List recent closure events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CloseAccount --max-results 10`

SHOULD:
- Check for active resources in the account that may incur charges during suspension
- Verify no other accounts depend on resources in the account being closed
- Check for delegated administrator registrations that need to be removed first

MAY:
- Review the account's billing history for outstanding charges
- Check for cross-account resource dependencies

## Phase 2 — Remediate

MUST:
- Delete or terminate all resources in the account before closure to stop charges
- Remove delegated administrator registrations before closing
- Close the account: `aws organizations close-account --account-id <account-id>`

SHOULD:
- Document the account closure for audit purposes
- Remove the account from any OUs or policy targets
- Notify stakeholders about the account closure and 90-day recovery window

MAY:
- Create a backup of important data before closure
- Set a reminder for the 90-day permanent deletion date

## Common Issues

- symptoms: "Cannot close account — ConstraintViolationException"
  diagnosis: "The account is a delegated administrator or has other organizational dependencies."
  resolution: "Remove delegated admin registrations and organizational dependencies first."

- symptoms: "Account closed but charges still appearing"
  diagnosis: "Resources were not deleted before closure. Some resources incur charges during suspension."
  resolution: "Reopen the account, delete resources, then close again. Or wait for 90-day permanent deletion."

- symptoms: "Need to reopen a closed account within 90 days"
  diagnosis: "Account is in the suspension period and can be recovered."
  resolution: "Contact AWS Support or use root credentials to reopen the account."

## Output Format

```yaml
root_cause: "account_closure_issue — <specific_cause>"
evidence:
  - type: account_status
    content: "<account status and details>"
  - type: dependencies
    content: "<organizational dependencies>"
severity: HIGH
mitigation:
  immediate: "Resolve dependencies and close the account, or reopen if needed"
  long_term: "Implement account lifecycle management with pre-closure cleanup"
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
