---
title: "A3 — Account Suspension Issues"
description: "Diagnose account suspension causes and recovery options"
status: active
severity: HIGH
triggers:
  - "account suspended"
  - "account locked"
  - "cannot access account"
  - "billing suspension"
owner: devops-agent
objective: "Understand the suspension cause and recover the account if possible"
context: "Accounts can be suspended due to closure (90-day grace period), billing issues (non-payment), or policy violations. Suspended accounts cannot be used but resources may persist. Recovery depends on the suspension reason."
---

## Phase 1 — Triage

MUST:
- Check account status: `aws organizations describe-account --account-id <account-id>`
- Determine the suspension reason (closure, billing, policy)
- Check if the account was recently closed: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CloseAccount --max-results 10`
- Verify the management account billing status

SHOULD:
- Check for billing-related suspension notifications
- Review the account's recent activity in CloudTrail
- Check if the account is within the 90-day recovery window

MAY:
- Contact AWS Support for suspension details
- Check for policy violation notifications

## Phase 2 — Remediate

MUST:
- For closure suspension: reopen within 90 days if needed
- For billing suspension: resolve the payment issue on the management account
- For policy violation: address the violation and contact AWS Support

SHOULD:
- Set up billing alerts to prevent payment-related suspensions
- Document the suspension and recovery for audit purposes

MAY:
- Migrate critical workloads to another account before the recovery window expires
- Implement automated billing monitoring

## Common Issues

- symptoms: "Account suspended after closure — need to recover"
  diagnosis: "Account is in the 90-day suspension period after closure."
  resolution: "Contact AWS Support or use root credentials to reopen within 90 days."

- symptoms: "Account suspended due to billing — all member accounts affected"
  diagnosis: "Management account has a billing issue affecting the entire organization."
  resolution: "Resolve the payment issue on the management account immediately."

## Output Format

```yaml
root_cause: "account_suspension — <specific_cause>"
evidence:
  - type: account_status
    content: "<account status and suspension details>"
  - type: cloudtrail
    content: "<relevant events>"
severity: HIGH
mitigation:
  immediate: "Resolve the suspension cause and recover the account"
  long_term: "Set up billing alerts and account monitoring"
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
