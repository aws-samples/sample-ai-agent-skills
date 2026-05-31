---
title: "H2 — Organization Restructuring"
description: "Diagnose issues during organization restructuring and OU reorganization"
status: active
severity: MEDIUM
triggers:
  - "restructure organization"
  - "reorganize OUs"
  - "merge organizations"
  - "split organization"
owner: devops-agent
objective: "Successfully restructure the organization with minimal disruption"
context: "Organization restructuring involves moving accounts between OUs, creating/deleting OUs, and updating SCPs. Each account move changes effective SCPs immediately. There is no bulk move operation — accounts must be moved one at a time. Merging organizations requires removing accounts from one org and inviting them to another."
---

## Phase 1 — Triage

MUST:
- Map the current OU hierarchy: list all OUs and their accounts
- Document current SCPs at each level
- Identify the target OU structure
- List all accounts and their current placement

SHOULD:
- Identify accounts with special roles (delegated admin, security)
- Check for cross-account dependencies
- Plan the migration order to minimize disruption

MAY:
- Create a visual diagram of current and target structures
- Estimate the time required for the restructuring

## Phase 2 — Remediate

MUST:
- Create new OUs before moving accounts: `aws organizations create-organizational-unit --parent-id <parent-id> --name <ou-name>`
- Move accounts one at a time: `aws organizations move-account --account-id <account-id> --source-parent-id <source> --destination-parent-id <dest>`
- Attach SCPs to new OUs before moving accounts
- Verify effective SCPs after each move

SHOULD:
- Perform restructuring during a maintenance window
- Move accounts in a planned order (least critical first)
- Delete empty OUs after migration: `aws organizations delete-organizational-unit --organizational-unit-id <ou-id>`

MAY:
- Use automation scripts for bulk account moves
- Implement rollback procedures for each step

## Common Issues

- symptoms: "Account lost permissions after OU restructuring"
  diagnosis: "New OU has different SCPs than the previous OU."
  resolution: "Attach the correct SCPs to the new OU before moving accounts."

- symptoms: "Cannot delete OU — still has children"
  diagnosis: "The OU still contains accounts or child OUs."
  resolution: "Move all accounts and delete child OUs before deleting the parent OU."

- symptoms: "Restructuring taking too long — no bulk move"
  diagnosis: "Accounts must be moved one at a time via the API."
  resolution: "Use automation scripts to move accounts in sequence. Plan for the time required."

## Output Format

```yaml
root_cause: "restructuring_issue — <specific_cause>"
evidence:
  - type: current_structure
    content: "<current OU hierarchy>"
  - type: target_structure
    content: "<target OU hierarchy>"
  - type: migration_status
    content: "<accounts moved and remaining>"
severity: MEDIUM
mitigation:
  immediate: "Complete the restructuring with proper SCP configuration"
  long_term: "Document the OU structure and implement change management"
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
