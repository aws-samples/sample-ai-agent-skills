---
title: "C1 — OU Structure Issues"
description: "Diagnose organizational unit structure and hierarchy problems"
status: active
severity: MEDIUM
triggers:
  - "OU structure"
  - "organizational unit"
  - "OU hierarchy"
  - "move account"
  - "OU design"
owner: devops-agent
objective: "Resolve OU structure issues and optimize the organizational hierarchy"
context: "OUs organize accounts into logical groups for policy application. OUs can be nested up to 5 levels. Moving accounts between OUs changes their effective SCPs immediately. OU design should align with security, compliance, and operational requirements."
---

## Phase 1 — Triage

MUST:
- List the root: `aws organizations list-roots`
- List OUs under root: `aws organizations list-organizational-units-for-parent --parent-id <root-id>`
- List accounts in the OU: `aws organizations list-children --parent-id <ou-id> --child-type ACCOUNT`
- Check OU nesting depth (max 5 levels)

SHOULD:
- Map the complete OU hierarchy
- Check SCPs attached to each OU
- Identify accounts that may be in the wrong OU

MAY:
- Compare the OU structure against AWS best practices (Security, Sandbox, Workloads, etc.)
- Check for empty OUs that can be cleaned up

## Phase 2 — Remediate

MUST:
- Move accounts to the correct OU: `aws organizations move-account --account-id <account-id> --source-parent-id <source-ou> --destination-parent-id <dest-ou>`
- Verify SCPs are correct after moving accounts
- Ensure the OU structure supports the required policy boundaries

SHOULD:
- Follow AWS recommended OU structure (Security, Infrastructure, Sandbox, Workloads, etc.)
- Document the OU hierarchy and purpose of each OU
- Plan OU changes carefully as they affect SCPs immediately

MAY:
- Use AWS Control Tower for standardized OU management
- Implement OU structure as code using CloudFormation or Terraform

## Common Issues

- symptoms: "Account in wrong OU — receiving incorrect SCPs"
  diagnosis: "Account was placed in or moved to an incorrect OU."
  resolution: "Move the account to the correct OU. Verify effective SCPs after the move."

- symptoms: "Cannot create nested OU — limit reached"
  diagnosis: "OU nesting has reached the 5-level maximum."
  resolution: "Restructure the hierarchy to stay within 5 levels. Use tags for additional grouping."

## Output Format

```yaml
root_cause: "ou_structure_issue — <specific_cause>"
evidence:
  - type: ou_hierarchy
    content: "<current OU structure>"
  - type: account_placement
    content: "<account OU membership>"
severity: MEDIUM
mitigation:
  immediate: "Move accounts to correct OUs"
  long_term: "Document and maintain the OU hierarchy"
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
