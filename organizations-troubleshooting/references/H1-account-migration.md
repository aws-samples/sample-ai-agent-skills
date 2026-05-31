---
title: "H1 — Account Migration"
description: "Diagnose account migration between OUs and organizations"
status: active
severity: HIGH
triggers:
  - "move account"
  - "migrate account"
  - "account migration"
  - "remove from organization"
  - "standalone account"
owner: devops-agent
objective: "Successfully migrate accounts between OUs or out of the organization"
context: "Moving accounts between OUs changes their effective SCPs immediately. Removing an account from the organization makes it standalone and requires root email access, a valid payment method, and support plan. Accounts created via Organizations may not have root credentials configured."
---

## Phase 1 — Triage

MUST:
- Check current account placement: `aws organizations list-parents --child-id <account-id>`
- Verify the account status: `aws organizations describe-account --account-id <account-id>`
- For removal: verify root email access is available for the account
- Check for dependencies (delegated admin, service integrations)

SHOULD:
- Review SCPs in the destination OU before moving
- Check for cross-account resource dependencies
- Verify the account has a valid payment method (for removal)

MAY:
- Plan the migration during a maintenance window
- Document the current and target configuration

## Phase 2 — Remediate

MUST:
- Move between OUs: `aws organizations move-account --account-id <account-id> --source-parent-id <source> --destination-parent-id <dest>`
- For removal: ensure root credentials are set up (use forgot password flow if needed)
- Verify effective SCPs after the move

SHOULD:
- Test the move in a sandbox environment first
- Notify stakeholders about SCP changes
- Update any automation that references the account's OU

MAY:
- Use AWS Control Tower Account Factory for standardized migrations
- Implement migration runbooks for repeatable processes

## Common Issues

- symptoms: "Account permissions changed after moving to new OU"
  diagnosis: "The new OU has different SCPs than the previous OU."
  resolution: "Review and adjust SCPs in the new OU, or move the account back."

- symptoms: "Cannot remove account — no root credentials"
  diagnosis: "Account created via Organizations API without root password setup."
  resolution: "Use the forgot password flow with the account email to establish root credentials."

- symptoms: "Removed account cannot access AWS services"
  diagnosis: "Standalone account needs a valid payment method and support plan."
  resolution: "Configure a payment method and support plan using root credentials."

## Output Format

```yaml
root_cause: "account_migration_issue — <specific_cause>"
evidence:
  - type: account_status
    content: "<account status and current OU>"
  - type: dependencies
    content: "<account dependencies>"
severity: HIGH
mitigation:
  immediate: "Complete the migration and verify configuration"
  long_term: "Document migration procedures and maintain root credentials"
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
